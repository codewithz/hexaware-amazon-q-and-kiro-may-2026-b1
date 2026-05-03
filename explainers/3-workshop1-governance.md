# Workshop 1 — Agent Governance Model Design

**Day:** 1  
**Layer:** Layer 1 (Frontier Agent Governance)  
**Duration:** 60 minutes (10:45–11:45)  
**Tool:** Any text editor (no IDE required for most of this workshop)  
**Deliverable:** Completed governance document in `.kiro/governance/agent-governance.md` covering IAM policy, branch protection rules, network access tier, approval matrix, and incident response

---

## What You Will Learn

By the end of this workshop you will have:
- Written an IAM policy that constrains what your AI agent can do in AWS
- Defined Git branch protection rules that prevent agents from pushing directly to main
- Chosen and documented your team's network access tier for agents
- Completed an approval matrix that defines which actions require human sign-off
- Documented an incident response procedure for agent-caused issues

---

## Why Governance Comes First

Before agents write a single line of code, you need to know:
- What can the agent access in your AWS account?
- Which Git branches can the agent push to?
- Which actions require a human to approve?
- If the agent does something wrong, what do you do?

Without this, you are not practising AI-assisted development — you are hoping nothing goes wrong. Governance turns that hope into a policy.

---

## Step 1 — Create the Governance Directory

```bash
mkdir -p .kiro/governance
touch .kiro/governance/agent-governance.md
code .kiro/governance/agent-governance.md
```

Start the document:

```markdown
# Agent Governance Policy

**Team:** [Your team name]  
**Author:** [Your name]  
**Date:** [Today's date]  
**Review cycle:** Quarterly  

---

This document defines the rules under which AI agents operate in our development workflow.
All team members must read and agree to this policy before using agentic AI tools.
```

---

## Section 1 — IAM Policy (15 minutes)

### What Is at Stake

When an agent has AWS credentials (through your IDE or CLI), it can in theory do anything those credentials allow. An agent with `AdministratorAccess` could delete production data, expose secrets, or rack up unexpected costs.

The IAM policy you write here defines the **minimum permissions** the agent needs for this project. Use the principle of least privilege.

### Your Project's AWS Services

For the Inventory Management Service, the agent needs access to:
- **SQS** — publish messages to the inventory events queue
- **CloudWatch Logs** — write application logs
- **SSM Parameter Store** — read database credentials and configuration

It does NOT need:
- S3 (we are not storing files)
- EC2 (we are not managing compute)
- IAM (the agent should never modify permissions)
- RDS (Flyway manages the schema, not the agent directly)

### Task: Write the IAM Policy

Add this section to your governance document and fill in the blanks based on your specific resource names:

```markdown
---

## 1. IAM Policy

The following IAM policy defines the minimum permissions for agent operation.
This policy must be attached to the IAM role or user used by the development tools.

**Policy name:** `ai-agent-dev-policy-[team-name]`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SQSInventoryQueueAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:GetQueueUrl",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:ap-south-1:ACCOUNT_ID:inventory-events-queue"
    },
    {
      "Sid": "CloudWatchLogsWrite",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:ap-south-1:ACCOUNT_ID:log-group:/application/inventory-service:*"
    },
    {
      "Sid": "SSMParameterRead",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/inventory-service/*"
    },
    {
      "Sid": "ExplicitDenyDangerousActions",
      "Effect": "Deny",
      "Action": [
        "iam:*",
        "sts:AssumeRole",
        "ec2:*",
        "rds:*",
        "s3:DeleteBucket",
        "s3:DeleteObject",
        "sqs:DeleteQueue",
        "cloudformation:DeleteStack"
      ],
      "Resource": "*"
    }
  ]
}
```

**Explicit Deny rationale:**
The `ExplicitDenyDangerousActions` statement uses Deny (not the absence of Allow) 
because Deny always overrides Allow, even if another policy grants broader access.
This is a safety net for permission misconfiguration.
```

> **Fill in `ACCOUNT_ID`** with your actual AWS account ID or a placeholder if in a training environment. Replace queue name, log group name, and SSM path with your actual values.

---

## Section 2 — Branch Protection Rules (10 minutes)

### Why Branch Protection Matters for Agents

An agent with Git write access can push to any branch — including `main`. Without branch protection:
- An agent could push untested code directly to production
- An agent could push code that breaks the build
- There is no audit trail of what the agent changed

Branch protection ensures that agent-generated code goes through the same review process as human-generated code.

### Task: Define Your Branch Protection Rules

Add this section to your governance document:

```markdown
---

## 2. Branch Protection Rules

These rules must be configured in the Git platform (GitHub/GitLab/Bitbucket) 
before agents are given any Git credentials.

### Protected Branches: `main` and `release/*`

| Rule | Setting | Rationale |
|---|---|---|
| Require pull request before merging | ✅ Enabled | Forces code review, even for agent output |
| Required approvals | 1 (minimum) | At least one human must approve |
| Dismiss stale reviews on push | ✅ Enabled | Forces re-review if agent pushes again |
| Require status checks before merging | ✅ Enabled | CI must pass (build + tests) |
| Required status checks | `build`, `test`, `security-scan` | All three must be green |
| Require branches to be up to date | ✅ Enabled | Prevents stale branch merges |
| Restrict who can push | Humans only (not the agent service account) | Agents cannot bypass the PR process |
| Allow force pushes | ❌ Disabled | Prevents history rewriting |
| Allow deletions | ❌ Disabled | Prevents accidental branch deletion |

### Agent Working Branch Convention

Agents must only push to branches matching this pattern:
```
agent/[task-type]/[brief-description]
```

Examples:
- `agent/feature/inventory-reservation-service`
- `agent/fix/null-pointer-in-stock-calculation`
- `agent/refactor/extract-event-publisher`

Branches not matching this pattern will be rejected by the pre-push hook.

### GitHub CLI Commands to Apply These Rules

```bash
# Using GitHub CLI (gh)
gh api repos/[org]/[repo]/branches/main/protection \
  -X PUT \
  -f required_status_checks='{"strict":true,"contexts":["build","test","security-scan"]}' \
  -f enforce_admins=true \
  -f required_pull_request_reviews='{"required_approving_review_count":1,"dismiss_stale_reviews":true}' \
  -f restrictions=null
```
```

---

## Section 3 — Network Access Tier (10 minutes)

### The Three Tiers

Agents can be configured with different levels of internet access. Choosing the right tier prevents the agent from leaking code, accessing external systems, or introducing supply-chain risks.

| Tier | Internet Access | When to Use |
|---|---|---|
| **Tier 1 — Air-gapped** | None — local files only | Highly regulated environments (healthcare, defence, banking core systems) |
| **Tier 2 — Controlled** | Specific approved domains only | Most enterprise teams — allow package registries, block general internet |
| **Tier 3 — Open** | Full internet access | Open-source projects, public APIs, teams comfortable with higher risk |

### Task: Choose and Document Your Tier

Add this section to your governance document:

```markdown
---

## 3. Network Access Tier

**Selected tier:** [Choose: Tier 1 / Tier 2 / Tier 3]

**Rationale:** [Write 2-3 sentences explaining why you chose this tier for your project]

### Approved Domains (Tier 2 only)

If you selected Tier 2, list the domains the agent is allowed to access:

| Domain | Purpose |
|---|---|
| `*.maven.org` | Maven Central — Java dependency downloads |
| `*.npmjs.org` | npm registry — Node.js dependency downloads |
| `kiro.dev` | Kiro IDE updates |
| `docs.spring.io` | Spring documentation reference |
| `docs.aws.amazon.com` | AWS service documentation |

All other domains are blocked by default.

### Network Controls Implementation

[Describe how this tier is enforced in your organisation:
- Corporate proxy with allowlist?
- VPC with restricted egress?
- IDE plugin setting?
- IAM policy restricting API calls by source IP?]
```

---

## Section 4 — Approval Matrix (15 minutes)

The approval matrix answers the question: **for each type of action the agent might take, who must approve it before it happens?**

### The Four Approval Levels

| Level | Who Approves | When Used |
|---|---|---|
| **Autopilot** | No human required | Safe, easily reversible actions (adding Javadoc) |
| **Developer** | The developer at the keyboard | Low-risk actions visible in the IDE |
| **Tech Lead** | Senior developer or tech lead | Architectural changes, new dependencies |
| **Committee** | Security + tech lead + engineering manager | Production data access, new AWS services |

### Task: Complete the Approval Matrix

Add this section to your governance document and fill in the Approval Level column for each action:

```markdown
---

## 4. Approval Matrix

For each action type, the specified approval is required BEFORE the agent proceeds.

| Action Type | Example | Approval Level | Notes |
|---|---|---|---|
| Write code to a feature branch | Creating a new service class | [choose level] | |
| Write code to a release branch | Hotfix for production | [choose level] | |
| Add a new dependency to pom.xml | Adding a new Spring Boot starter | [choose level] | |
| Add a new npm dependency | Adding an Angular library | [choose level] | |
| Add a Flyway database migration | Creating a new table | [choose level] | |
| Delete a Flyway migration | Removing an existing migration | [choose level] | |
| Modify an existing Flyway migration | Changing a column type | [choose level] | |
| Run automated tests | mvn test | [choose level] | |
| Run a security scan | /scan on codebase | [choose level] | |
| Publish to a package registry | Deploying a library | [choose level] | |
| Make an API call to an external service | Calling a third-party REST API | [choose level] | |
| Read production logs | CloudWatch log search | [choose level] | |
| Query production database (read) | SELECT from prod RDS | [choose level] | |
| Write to production database | UPDATE or INSERT on prod | [choose level] | |
| Delete production data | DELETE from prod | [choose level] | |
| Create an AWS resource | New SQS queue, S3 bucket | [choose level] | |
| Delete an AWS resource | Removing an SQS queue | [choose level] | |
| Modify IAM policies | Adding permissions | [choose level] | |

**Reasoning for your choices:**
[Write 3-5 sentences explaining the overall approach you took to assigning approval levels.
What principles guided you? What were the hardest decisions?]
```

> **Discuss with your group** before filling this in. There is no single correct answer, but certain choices should be hard to justify — for example, "Autopilot" for "Delete production data" requires extraordinary justification.

---

## Section 5 — Incident Response (10 minutes)

### What Counts as an Incident?

An agent incident is any situation where the agent:
- Modifies or deletes data it should not have touched
- Accesses a system outside its approved network tier
- Pushes code that breaks the build or fails security checks
- Incurs unexpected AWS costs (even if small)
- Exposes credentials or sensitive data in a commit or log

### Task: Write the Incident Response Procedure

Add this final section to your governance document:

```markdown
---

## 5. Incident Response

### Incident Classification

| Severity | Definition | Response Time |
|---|---|---|
| P1 — Critical | Production data modified or deleted, credentials exposed | Immediately (within 15 minutes) |
| P2 — High | Security scan blocked, build broken on main, unexpected cost > $50 | Within 2 hours |
| P3 — Medium | Agent accessed a domain outside the approved list | Within 24 hours |
| P4 — Low | Agent wrote code that did not follow steering file standards | Next sprint |

### P1/P2 Incident Response Steps

1. **Disable the agent immediately** — revoke or rotate the IAM credentials used by the agent
2. **Contain the blast radius** — identify all actions the agent took in the last 30 minutes (use CloudTrail for AWS actions, Git log for code changes)
3. **Roll back if possible** — `git revert` for code, AWS console for infrastructure
4. **Notify** — tech lead + engineering manager within 30 minutes
5. **Preserve evidence** — export CloudTrail logs, Git history, and IDE session logs before making any further changes
6. **Root cause analysis** — within 48 hours, document what the agent did, what governance gap allowed it, and what rule change prevents recurrence
7. **Update this document** — add the new rule to the relevant section of this governance policy

### Agent Activity Audit

All agent actions in AWS are logged in CloudTrail. Review agent activity:

```bash
# View agent IAM role activity in the last 24 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=ai-agent-role \
  --start-time $(date -d '24 hours ago' --iso-8601=seconds) \
  --query 'Events[].{Time:EventTime,Event:EventName,Resource:Resources[0].ResourceName}' \
  --output table
```

All agent Git actions are logged in the repository's audit log (GitHub/GitLab settings → Audit log).

### Contact List

| Role | Name | Contact |
|---|---|---|
| Tech Lead | [Name] | [Slack/email] |
| Engineering Manager | [Name] | [Slack/email] |
| Security Lead | [Name] | [Slack/email] |
| AWS Account Owner | [Name] | [Slack/email] |
```

---

## Step 6 — Sign-Off and Commit

Add a sign-off block at the end of the document:

```markdown
---

## Sign-Off

This governance policy has been reviewed and agreed to by:

- [ ] Tech Lead: _______________  Date: ___________
- [ ] Engineering Manager: _______________  Date: ___________
- [ ] Security Representative: _______________  Date: ___________

**Next review date:** [3 months from today]
```

Now commit:

```bash
git add .kiro/governance/
git commit -m "feat: Layer 1 — agent governance policy

Governance document covering:
- IAM policy with explicit deny for dangerous actions
- Branch protection rules for main and release/* branches
- Network access Tier [N] with domain allowlist
- Approval matrix for [N] action types
- P1-P4 incident classification and response procedure

Signed off by: [names]"

git push origin main
```

---

## Workshop Completion Criteria

- [ ] `.kiro/governance/agent-governance.md` exists and is committed
- [ ] IAM policy JSON is complete with at least 3 Allow statements and an ExplicitDeny block
- [ ] Branch protection table is filled in for all 10 rule types
- [ ] Network access tier is chosen with written rationale
- [ ] Approval matrix has a level assigned for every action type (18 total)
- [ ] Rationale paragraph explains the overall decision-making approach
- [ ] Incident response procedure covers P1–P4 with response times
- [ ] Contact list is filled in (can use placeholder names for training)
- [ ] Sign-off block is present
