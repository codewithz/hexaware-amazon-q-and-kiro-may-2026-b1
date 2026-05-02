# Workshop 1 — Designing Your Agent Governance Model

**Module:** 1.3 — Governing Frontier Agents (Layer 1)
**Duration:** 45 minutes | **Time Slot:** 10:45 – 11:30 AM
**Deliverable:** `agent-governance.md` — a documented governance policy committed to `.kiro/governance/`

---

> **📘 Note for Candidates**
> This workshop includes AWS Console steps. If you do not have an AWS account, do not skip this document — read every step and every explainer. You are learning the *design thinking*, not just the button clicks. The concepts here apply to any cloud provider and any automated system.

---

## Objective

An autonomous agent that can read your entire codebase, write files, run shell commands, and commit to Git is a powerful actor in your engineering system. Before you give it any of those permissions, you need to decide exactly what it is allowed to do — and what it is not.

In this workshop you will design the governance model for your training project's AI agent. You will make real design decisions about IAM permissions, Git access, network boundaries, and incident response. The output is a living document that travels with your repository.

By the end you will have:
- An IAM policy limiting what your agent can do in AWS
- A branch protection ruleset preventing the agent from pushing to `main`
- A documented network access tier decision
- A signed governance document committed to your repository

---

## Why Governance Comes Before Features

Most teams skip this step and regret it. Common outcomes:

- An agent commits directly to `main`, bypassing all review
- An agent calls an AWS API it was not supposed to have access to, creating unexpected charges or data exposure
- An agent executes a shell command that deletes files that cannot be recovered
- No one can determine after the fact what the agent did or why

Governance is not bureaucracy — it is the engineering design of your agent's trust boundary.

> **💡 Core Concept: Trust Boundaries**
>
> Every software system has a trust boundary — a line that separates what is inside your control from what is not. For a human developer, that boundary is enforced by culture, code review, and professional accountability.
>
> An AI agent has none of those. It will do exactly what its permissions allow, nothing more, nothing less. Governance is how you draw that boundary *programmatically* — before the agent ever runs.
>
> This is not unique to AI agents. The same principle applies to CI/CD service accounts, Lambda execution roles, and third-party integrations. Any automated actor needs a deliberately designed trust boundary.

---

## Part 1 — IAM Policy Design (15 minutes)

### Step 1 — Identify what your agent actually needs

Look at what the agent will do in this training program:

| Agent Action | AWS Service Required |
|---|---|
| Read and write source code locally | None (local only) |
| Publish events to SQS (Lab 3, Capstone 1) | `sqs:SendMessage` |
| Read SQS queue configuration | `sqs:GetQueueAttributes`, `sqs:GetQueueUrl` |
| Run tests that connect to a local PostgreSQL | None (local Docker) |
| Pull base Docker images for Testcontainers | None (Docker Hub, not AWS) |

> **💡 Why Map Actions Before Writing Policy**
>
> Before you can write a least-privilege IAM policy, you must enumerate every action the agent will take. This seems obvious, but most teams skip it — they start with a broad policy ("just give it S3 access") and plan to narrow it down later.
>
> The problem: "later" never comes.
>
> Starting from this table forces you to prove each permission is necessary. If you cannot point to a specific action in the table, the permission does not go in the policy. If a new action is needed later, it triggers a deliberate review — not a silent permission expansion.
>
> Notice how most actions in this table require *no* AWS permissions at all. That is the correct outcome. The default answer should always be "no AWS service needed."

---

### Step 2 — Write the least-privilege IAM policy

Create the file `.kiro/governance/agent-iam-policy.json` with the following content. Replace `YOUR_ACCOUNT_ID` and `YOUR_REGION` with your actual values:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSQSForInventoryEvents",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:YOUR_REGION:YOUR_ACCOUNT_ID:inventory-events-*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "YOUR_REGION"
        }
      }
    },
    {
      "Sid": "DenyAllOtherActions",
      "Effect": "Deny",
      "NotAction": [
        "sqs:SendMessage",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl"
      ],
      "Resource": "*"
    }
  ]
}
```

> **💡 Anatomy of This Policy — Line by Line**
>
> **`"Version": "2012-10-17"`**
> This is the IAM policy language version. Always use this exact date string. It enables condition operators and other modern features. Using an older version silently disables some policy capabilities — it is not just a label.
>
> **`"Sid"` (Statement ID)**
> A human-readable label for each statement. It has no security function but is essential for auditability: when CloudTrail logs a denied action, the Sid appears in the log, telling you exactly which rule triggered. Name Sids like function names — descriptive and specific.
>
> **`"Effect": "Allow"` / `"Deny"`**
> IAM evaluates Deny statements first, then Allow. If no statement matches an action, the default is implicit Deny. An explicit Deny always wins over any Allow, even one granted by a different policy attached to the same user.
>
> **`"Resource"` with wildcard suffix (`inventory-events-*`)**
> The agent is allowed to act on any SQS queue whose name starts with `inventory-events-`. This accommodates dev/prod/staging environments without rewriting the policy for each. The wildcard is intentional and bounded — it does not mean "all SQS resources in all accounts."
>
> **`"Condition": aws:RequestedRegion`**
> Even if someone copied these credentials and tried to use them in another AWS region, this condition blocks the action. This is *defence in depth*: multiple independent controls, each providing partial protection. Any one of them failing alone does not compromise the system.
>
> **`"NotAction"` in the Deny statement**
> This is an inverted match. Instead of listing what to deny, it denies everything *except* the listed actions. This pattern is safer because new AWS services launched in the future are automatically denied — you do not need to update the policy to block them.

---

### Step 3 — Create a dedicated IAM user for the agent

> **⚠️ Important — Never Use Personal Credentials for an Agent**
>
> Do not use your personal AWS credentials for the agent. The agent's credentials must be isolated from yours.
>
> If the agent's key is accidentally committed to Git, only the agent's bounded permissions are exposed — not your full AWS account. Credential separation is not optional.

In the AWS Console:

1. Go to **IAM → Users → Create user**
2. Name the user: `kiro-agent-training-[your-name]`
3. Select **Attach policies directly**
4. Click **Create policy** and paste the JSON from Step 2
5. Name the policy: `KiroAgentTrainingPolicy`
6. Attach the policy to the new user
7. Go to **Security credentials → Create access key**
8. Select **Application running outside AWS**
9. Download the credentials — you will need them in Lab 3

> **💡 Why a Separate IAM User? — The Principle of Isolation**
>
> When a developer uses their own AWS credentials for tooling, the developer's full permissions become the tool's permissions. If that tool is compromised, an attacker has developer-level access to everything the developer can do.
>
> A dedicated IAM user with a restricted policy means the agent is a **separate identity** with a **separate, bounded trust level**. This is the same pattern used for:
> - CI/CD pipeline service accounts
> - Lambda function execution roles
> - Microservice IAM roles
> - Third-party SaaS integrations
>
> Any automated actor gets its own identity. That identity has only the permissions required for its specific function.
>
> In a real enterprise: the agent's access key would be stored in AWS Secrets Manager or a CI secrets vault — never in `.env` files, never in code. Rotation would be automated every 90 days. The training setup here mirrors that pattern at a smaller scale.

---

### Step 4 — Review the design decisions

Answer these questions in your notes (you will include them in your governance document):

**Q: Why did you scope the SQS resource with a wildcard suffix (`inventory-events-*`)?**
Because new queues may be created during development (e.g. a test queue and a prod queue). Locking to a specific queue name would break the policy every time a new queue is created.

**Q: Why is there an explicit Deny for all other actions?**
Because AWS IAM evaluates Deny before Allow. An explicit Deny ensures that even if another policy accidentally grants the agent broader access, it cannot act outside its defined scope.

**Q: Why not just use AdministratorAccess for simplicity?**
Because if the agent's credentials are compromised (e.g. accidentally committed to Git), the blast radius is bounded to only the SQS actions defined. Admin credentials would give an attacker full AWS account access.

> **🔑 Key Concept: Blast Radius**
>
> "Blast radius" is a security term borrowed from explosives engineering. It describes how far damage spreads when something goes wrong.
>
> **With `AdministratorAccess`:** a leaked key could delete all S3 buckets, spin up thousands of EC2 instances (creating a five-figure cloud bill overnight), exfiltrate RDS databases, and lock you out of your own account.
>
> **With least-privilege:** a leaked key can only send messages to SQS queues whose names start with `inventory-events-`. That is the entire blast radius.
>
> Containment is designed in, not bolted on after an incident. The goal of least-privilege is to make the worst-case scenario survivable.

---

## Part 2 — Git Branch Protection (10 minutes)

### Step 5 — Define the agent's Git access level

Your agent will be making commits during the labs. Decide where it can commit:

| Branch | Agent Access | Reason |
|---|---|---|
| `feature/agent-*` | ✅ Read + Write | Agent works on dedicated feature branches only |
| `develop` | ✅ Read only | Agent can read but cannot push |
| `main` | ❌ No access | Human review required before anything reaches main |

> **💡 Why Branch Protection Is Your Most Important Governance Control**
>
> IAM controls what the agent can do *in the cloud*. Branch protection controls what the agent can do *to your codebase*.
>
> `main` (and `develop`) represent production-ready or near-production code. Every organisation has a process for getting code there: code review, automated tests, QA sign-off. If an agent can push directly to `main`, it bypasses every single one of those gates.
>
> This has happened in practice: AI coding assistants given "full repository access" have committed breaking changes, security vulnerabilities, and even secrets directly to `main` — all without human review. Branch protection is the enforcement mechanism that prevents this, regardless of how the agent is configured internally.
>
> The table above encodes a simple principle: **the agent proposes, humans approve**. The agent can do as much work as it wants on a feature branch. Nothing reaches a protected branch without human review.

---

### Step 6 — Configure branch protection on GitHub

Go to your training repository on GitHub:

1. **Settings → Branches → Add branch protection rule**
2. Branch name pattern: `main`
3. Enable:
   - [x] **Require a pull request before merging**
   - [x] **Require approvals** — set to **1**
   - [x] **Dismiss stale pull request approvals when new commits are pushed**
   - [x] **Restrict who can push to matching branches** — add only your own GitHub username (not the agent's)
4. Click **Save changes**

Repeat for `develop` with the same settings.

> **💡 What Each Protection Setting Actually Does**
>
> **"Require a pull request before merging"**
> Forces all changes through a PR, even if someone has direct push access. No exceptions, no shortcuts. This is the structural gate that makes human review mandatory.
>
> **"Require approvals: 1"**
> At least one human must explicitly approve the PR before it can be merged. This is the human-in-the-loop checkpoint — the moment a person looks at what the agent produced and says "yes, this is correct."
>
> **"Dismiss stale approvals when new commits are pushed"**
> If the agent (or anyone) pushes new commits after a human approved, the approval is automatically revoked. This prevents the approve-then-sneak-in-changes pattern — a previously approved state cannot be used to merge a different state.
>
> **"Restrict who can push to matching branches"**
> Even with the rules above, a repository admin could force-push and bypass protections. This setting removes that option for everyone except specifically named users. The agent's GitHub account should never appear on this list.

---

### Step 7 — Verify branch protection is active

```bash
# Try to push directly to main (this should be rejected)
git checkout main
echo "test" >> README.md
git add README.md
git commit -m "test: verify branch protection is blocking direct push"
git push origin main
```

You should see an error like:
```
remote: error: GH006: Protected branch update failed for refs/heads/main.
remote: error: Changes must be made through a pull request.
```

This confirms branch protection is working. Reset the test:
```bash
git reset --hard HEAD~1
git checkout -b feature/agent-workshop1
```

> **💡 Why Run a Verification Test — Not Just Trust the Configuration**
>
> Configuration is not the same as enforcement. It is entirely possible to configure branch protection incorrectly — for example, forgetting to check "Include administrators" — and believe it is active when it is not.
>
> The test above *proves* the control is actually enforced, not just configured. In professional security practice, this is called a **control effectiveness test**. You should run it every time you change branch protection settings.
>
> A control you have not tested is a control you do not actually have.

---

## Part 3 — Network Access Tier (5 minutes)

### Step 8 — Choose your agent's network access tier

Agents can be configured with different levels of network access. Choose the tier that fits your use case:

| Tier | What the agent can reach | Use this when |
|---|---|---|
| **Tier 0 — No network** | Nothing — local files only | Sensitive codebases, air-gapped environments |
| **Tier 1 — Internal only** | Your VPC, internal tools (Jira, Confluence, private npm) | Most enterprise teams |
| **Tier 2 — Allow-listed external** | Specific external APIs and registries (npmjs.com, pypi.org, your public API docs) | Teams using public package registries and public APIs |
| **Tier 3 — Full internet** | Any URL | Prototyping, non-sensitive projects |

**For this training program:** Select **Tier 2**. Your agent needs access to:
- `registry.npmjs.org` (if frontend work is involved)
- `repo.maven.apache.org` (Maven Central for dependencies)
- AWS API endpoints (`sqs.amazonaws.com`, etc.)
- `hub.docker.com` (for Testcontainers images)

Document your chosen tier and the allowed domains in your governance file.

> **💡 Why Network Tiers Matter — Not Just a Firewall Setting**
>
> An agent with unrestricted internet access can:
> - Exfiltrate source code or secrets to an external server
> - Contact command-and-control infrastructure if the agent process itself is compromised
> - Be manipulated through prompt injection from external web content it fetches
>
> **Tier 0** is the most secure but limits usefulness — the agent cannot pull dependencies or call external APIs. **Tier 3** maximises capability but maximises attack surface. The tiering model forces a deliberate, documented choice — not a default nobody thought about.
>
> **Tier 2 (allow-listed)** reflects real enterprise practice: the agent gets exactly the external access it needs, nothing more. Each allowed domain is a conscious decision recorded in the governance document, creating an audit trail.
>
> Allow-listing also protects against supply-chain attacks: if a malicious package tries to phone home to an unlisted domain, the network tier blocks it before it can succeed — regardless of whether anyone noticed the malicious package was installed.

---

## Part 4 — Write the Governance Document (15 minutes)

### Step 9 — Create the governance directory and file

```bash
mkdir -p .kiro/governance
touch .kiro/governance/agent-governance.md
```

> **💡 Why Governance Lives in the Repository — Not in a Wiki**
>
> Governance stored in Confluence, a shared drive, or a wiki is governance that will drift out of sync with the actual system. Within six months, nobody will know if the wiki reflects what was configured, what was intended, or what the team agreed to three reorganisations ago.
>
> Governance committed to the repository:
> - **Travels with the code** — every developer who clones the repo gets the governance document
> - **Is versioned** — you can see exactly what changed, when, and who approved it
> - **Is diffable** — a PR that modifies governance is visible and reviewable
> - **Cannot be out of sync** — the document is in the same place as the system it governs
>
> The `.kiro/` directory follows the same convention as `.github/` for GitHub Actions or `.gitlab/` for GitLab CI — a well-known location for AI agent tooling and configuration that travels with the repository.

---

### Step 10 — Fill in the governance document

Open `.kiro/governance/agent-governance.md` and add the following content, filling in your own decisions:

```markdown
---
document: agent-governance
version: 1.0
author: [Your Name]
date: [Today's Date]
review-date: [3 months from today]
---

# Agent Governance Policy — Training Project

## Purpose

This document defines the trust boundary, access permissions, and behavioural 
constraints for AI agents operating on this codebase. All agents (Amazon Q Developer, 
Kiro) must operate within the boundaries defined here.

This document must be reviewed when:
- A new AWS service is added to the project scope
- The agent is granted access to a new tool or integration
- A security incident occurs involving the agent
- A new team member joins who will work with the agent

---

## 1. Identity & Credentials

| Attribute | Value |
|---|---|
| Agent IAM User | `kiro-agent-training-[your-name]` |
| IAM Policy | `KiroAgentTrainingPolicy` |
| Credential Rotation Schedule | Every 90 days |
| Credential Storage | AWS Secrets Manager (never in code or `.env` files committed to Git) |

---

## 2. AWS Permissions

The agent is permitted to perform the following AWS actions:

- `sqs:SendMessage` on `arn:aws:sqs:REGION:ACCOUNT:inventory-events-*`
- `sqs:GetQueueAttributes` on the same resource pattern
- `sqs:GetQueueUrl` on the same resource pattern

The agent is explicitly **denied** all other AWS actions via the `DenyAllOtherActions` 
statement in `agent-iam-policy.json`.

---

## 3. Git Access

| Branch | Agent Permission |
|---|---|
| `feature/agent-*` | Read + Write |
| `develop` | Read only |
| `main` | No access |
| Any other branch | Read only |

Branch protection is configured in GitHub repository settings. The agent may not 
bypass pull request requirements or approvals.

---

## 4. Network Access

**Tier:** 2 — Allow-listed external domains

**Permitted domains:**
- `registry.npmjs.org`
- `repo.maven.apache.org`
- `sqs.REGION.amazonaws.com`
- `hub.docker.com`
- `registry-1.docker.io`

The agent may not make requests to any domain not listed above without an explicit 
update to this document.

---

## 5. File System Permissions

The agent may:
- Read all files in the repository
- Write to `src/`, `test/`, `.kiro/`
- Execute `mvn`, `npm`, `git` commands
- Run Docker containers for test infrastructure

The agent may **not**:
- Delete files outside of `target/` and `node_modules/`
- Modify `.github/workflows/` (CI/CD pipeline definitions)
- Access files outside the repository root
- Execute `rm -rf` or equivalent destructive shell commands

---

## 6. Incident Response

If the agent takes an unexpected action:

1. **Stop the agent immediately** — close the IDE or kill the agent process
2. **Review the Git diff** — `git status` and `git diff` to see what changed
3. **Roll back if necessary** — `git checkout -- .` to discard unstaged changes
4. **Document the incident** — add an entry to `.kiro/governance/incidents.md`
5. **Review this governance document** — update permissions if a gap is identified

---

## 7. Sign-off

By committing this document to the repository, the team agrees to operate 
AI agents within the boundaries defined here.

| Name | Role | Date |
|---|---|---|
| [Your Name] | Developer | [Today's Date] |
```

> **💡 Why Each of the 7 Sections Exists**
>
> **Section 1 — Identity & Credentials**
> Answers "who is the agent?" An agent without a distinct identity cannot be audited. Every action in a cloud account needs a traceable actor. The credential rotation schedule here (90 days) is not arbitrary — it is the standard used by AWS security best practice guidelines. If credentials are rotated regularly, the window of exposure for any leaked key is bounded.
>
> **Section 2 — AWS Permissions**
> The human-readable version of the IAM policy JSON. When a non-engineer asks "what can the agent do in AWS?", this section answers it without requiring them to parse IAM syntax. Governance documents must be readable by everyone who is responsible for the system — not just the people who wrote the policy.
>
> **Section 3 — Git Access**
> Explicitly documents what the agent can and cannot change in the repository. This prevents scope creep: "can the agent edit the CI pipeline?" is answered by checking here, not by asking whoever set it up. The answer is no — `.github/workflows/` is explicitly off-limits in Section 5.
>
> **Section 4 — Network Access**
> Documents the tier decision and every allowed domain. If a new dependency needs to call an external service, adding it here is a deliberate governance change with a PR, a review, and a commit trail — not an invisible side-effect of installing a package.
>
> **Section 5 — File System Permissions**
> The local equivalent of IAM. What can the agent read? Write? Execute? What is completely off-limits? Note the specific prohibition on `rm -rf` — this is included because AI agents have, in documented incidents, executed destructive shell commands when given ambiguous instructions about "cleaning up" a project.
>
> **Section 6 — Incident Response**
> The most critical section for organisational safety. What do you do when something goes wrong? Teams that have not written this down improvise under pressure, which leads to compounding mistakes. A pre-written procedure reduces both the blast radius and the response time of any incident. The five steps here are ordered deliberately: stop first, understand second, roll back third, document fourth, fix the governance gap fifth.
>
> **Section 7 — Sign-off**
> Governance without accountability is a suggestion. The sign-off table makes the governance a commitment: a named person has reviewed and accepts responsibility for operating within these boundaries. When a new team member joins, they should add their name here as part of their onboarding — not as a formality, but as a signal that they have read and understood the governance model.

---

### Step 11 — Commit the governance files

```bash
git add .kiro/governance/
git commit -m "governance: add Layer 1 agent governance policy

- IAM policy with least-privilege SQS access
- Git branch protection configuration
- Network access Tier 2 with allow-listed domains
- Incident response procedure
- Signed by team

Workshop 1 — Day 1"
git push origin feature/agent-workshop1
```

> **💡 Why the Commit Message Format Matters**
>
> The message follows Conventional Commits format: `type(scope): description`. This is a widely adopted standard that makes repository history searchable and machine-readable — some CI tools parse it to auto-generate changelogs and release notes.
>
> The multi-line body summarises exactly what was added. Six months from now, when someone runs `git log --oneline`, this commit will be immediately identifiable as a governance change — not a feature, not a bug fix, a policy decision.
>
> Governance changes should always have descriptive commit messages. They are policy changes, not code changes, and they deserve the same documentation rigour. A commit message like `"add files"` on a governance document is a signal that the team does not treat governance as a serious engineering artifact.

---

## Lab Completion Checklist

- [ ] `agent-iam-policy.json` created with least-privilege SQS permissions
- [ ] Dedicated IAM user created (not using personal AWS credentials)
- [ ] Branch protection enabled on `main` — direct push rejected with error
- [ ] Network access tier chosen and documented
- [ ] `agent-governance.md` complete with all 7 sections filled in
- [ ] All files committed to `feature/agent-workshop1` branch
- [ ] Branch pushed to remote

---

## Key Takeaways

> **🔑 Least privilege is not optional**
> Every permission you do not grant is a risk you have eliminated. The question is not "what might the agent need?" — it is "what does it provably need right now?" Start with nothing and add only what is demonstrably necessary.

> **🔑 Governance is a Git artifact**
> Treat it like code: version it, review it, update it. A governance document that lives outside the repository is a document that will drift out of sync. Commit it alongside the code it governs.

> **🔑 Branch protection is your first line of defence**
> No agent should ever be able to push to `main` directly. This is not about distrust — it is about engineering a system where human review is structurally required, not just culturally expected.

> **🔑 An explicit Deny in IAM is stronger than the absence of an Allow**
> AWS defaults to implicit Deny, but that default can be overridden by another policy. An explicit Deny cannot. Always add the deny statement to make your intent unambiguous and override-resistant.

---

## What's Next

Your governance model is now documented and enforced at the IAM and Git layer. In Lab 2, you will build the second line of defence: steering files that tell the agent what to build and how to build it — without you having to repeat those constraints in every prompt.