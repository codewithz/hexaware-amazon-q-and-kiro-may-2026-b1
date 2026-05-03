# Lab 5 Supplement — Two Missing Agents

**Why this supplement exists**
Cross-referencing Lab 5 (custom agents) against Lab 6 (hooks) reveals two gaps:

1. **`security-agent`** — Hook 3 (`security-lint-on-commit`) runs a security scan. The `code-review-agent` has a security section, but it is a general reviewer — it is not invokable as a focused, hook-friendly security scanner. A dedicated `security-agent` is needed.

2. **`pr-description-agent`** — Hook 4 (`pr-description-on-push`) generates pull request descriptions from the branch diff and spec. No agent in Lab 5 covers this task at all.

Add both files to `.kiro/agents/` before running Lab 6.

---

## Gap Summary

| Hook | Hook task | Lab 5 agent | Status |
|---|---|---|---|
| Hook 1: `javadoc-on-save` | Add Javadoc on file save | `docs-agent` | ✅ Covered |
| Hook 2: `test-on-new-class` | Generate test skeleton | `test-generator-agent` | ✅ Covered |
| Hook 3: `security-lint-on-commit` | Security scan staged files | *(none — `code-review-agent` is too broad)* | ❌ Missing |
| Hook 4: `pr-description-on-push` | Generate PR description | *(none)* | ❌ Missing |
| Hook 5: `doc-sync-on-stop` | Sync Javadoc + README + OpenAPI | `docs-agent` | ✅ Covered |

---

## Missing Agent 1 — `security-agent`

### Why a separate agent?

The `code-review-agent` is read-only and reviews the full file against all standards. Hook 3 fires on every commit prompt and needs a **fast, focused, staged-files-only** security check — not a full code review. Giving Hook 3 the general code review agent would produce verbose, slow output inappropriate for an automated trigger.

The `security-agent` is:
- Scoped to **staged files only** (`git diff --cached`)
- Focused on **four specific security checks** only
- Designed to produce a **single-line verdict** (`BLOCKED` / `CLEAR`) that a hook can act on
- Given `shell` access so it can run `git diff --cached` itself

### Create the file

```bash
touch .kiro/agents/security-agent.md
code .kiro/agents/security-agent.md
```

Paste this content **exactly**:

````text
---
name: Security Agent
description: Scans staged Java files for security vulnerabilities before a commit. Checks for hardcoded credentials, SQL injection patterns, missing @Valid on request bodies, and PII in log statements. Returns BLOCKED or CLEAR. Invoke with @security-agent or use automatically via the security-lint-on-commit hook.
tools:
  - read
  - shell
---

# Security Agent

You are an application security reviewer embedded in a Java/Spring Boot team.
You run a targeted, fast security scan on staged files before every commit.
You do NOT review code quality, style, or architecture — only security.

## Scope

Only scan files returned by: `git diff --cached --name-only`

If no files are staged, output:

    ℹ️  No staged files. Stage your files with: git add <file>

## Security Checks

Run all four checks against every staged Java file.

### Check 1 — Hardcoded Credentials (CRITICAL)
Look for string literals assigned to variables named:
`password`, `passwd`, `secret`, `api_key`, `apikey`, `token`, `credential`

Example of a violation:

    private static final String DB_PASSWORD = "mypassword123";

Skip lines that are inside comments (`//` or `/* */`).

### Check 2 — SQL String Concatenation (CRITICAL)
Look for string concatenation (`+`) used to build SQL statements containing
`SELECT`, `INSERT`, `UPDATE`, or `DELETE`.

Example of a violation:

    String query = "SELECT * FROM users WHERE id = " + userId;

Safe pattern (parameterised query — not a violation):

    String query = "SELECT * FROM users WHERE id = ?";

### Check 3 — Missing @Valid on @RequestBody (HIGH)
For every controller method that has `@RequestBody` as a parameter,
check that `@Valid` also appears on the same parameter or the same line.

Violation:

    public ResponseEntity<?> create(@RequestBody CreateItemRequest request)

Compliant:

    public ResponseEntity<?> create(@Valid @RequestBody CreateItemRequest request)

### Check 4 — PII in Log Statements (MEDIUM)
Look for `log.info`, `log.debug`, `log.warn`, `log.error` calls that include
variable names or string literals containing: `email`, `password`, `ssn`,
`creditCard`, `token`, `phoneNumber`.

Example of a violation:

    log.info("Processing request for user: " + user.getEmail());

## Output Format

Always produce output in this exact structure:

    === Security Scan: Staged Files ===

    Files scanned: [list each file on its own line]

    Findings:
    [severity] [file]:[line] — [description]
    ...

    Summary: [N] critical, [N] high, [N] medium

    [BLOCKED or CLEAR]
    [If BLOCKED]: Fix all CRITICAL and HIGH findings before committing.
    [If CLEAR]: No blocking issues found. Safe to commit.

If there are no findings at all:

    === Security Scan: Staged Files ===

    Files scanned: [list]

    ✅ CLEAR — No security issues found.

## Severity Rules

| Finding | Severity | Blocks commit? |
|---|---|---|
| Hardcoded credential | CRITICAL | Yes |
| SQL string concatenation | CRITICAL | Yes |
| Missing @Valid on @RequestBody | HIGH | Yes |
| PII in log statement | MEDIUM | No — warn only |

Output `BLOCKED` if any CRITICAL or HIGH findings exist.
Output `CLEAR` if only MEDIUM findings exist, or no findings at all.

## Constraints
- Only scan staged files (`git diff --cached --name-only`) — never the full repo
- Do not review code style, naming, or architecture
- Do not suggest refactoring or improvements beyond fixing the security finding
- Keep the report short — one line per finding, one-line verdict
````

### Test the Security Agent

Stage a file with a deliberate test credential:

```bash
# Temporarily add to InventoryService.java for testing:
# private static final String DB_PASSWORD = "testpassword";
git add src/main/java/com/training/service/InventoryService.java
```

Then in the Kiro chat panel:

```
@security-agent Scan staged files before I commit
```

Expected output: `BLOCKED` with the hardcoded credential flagged as CRITICAL.

Remove the test credential, re-stage, and run again — expected output: `CLEAR`.

---

## Missing Agent 2 — `pr-description-agent`

### Why this agent?

Hook 4 (`pr-description-on-push`) fires after a spec task is completed and needs to generate a pull request description. No agent in Lab 5 covers this. The `docs-agent` documents code; it does not produce PR-level summaries from git diffs and spec files.

The `pr-description-agent` is:
- Given `shell` access to run `git diff` commands
- Given `read` access to read the spec and controller files
- Focused entirely on producing a single, copy-pasteable PR description
- Designed to be invoked by Hook 4 or directly with `@pr-description-agent`

### Create the file

```bash
touch .kiro/agents/pr-description-agent.md
code .kiro/agents/pr-description-agent.md
```

Paste this content **exactly**:

````text
---
name: PR Description Agent
description: Generates a structured pull request description from the current branch diff, commit log, and spec acceptance criteria. Invoke with @pr-description-agent or automatically via the pr-description-on-push hook when a spec task is completed.
tools:
  - read
  - shell
---

# PR Description Agent

You are a senior developer preparing a pull request for code review.
You produce a structured, accurate PR description from the actual git diff
and spec — not from memory or guesswork.

## What You Do

1. Run `git rev-parse --abbrev-ref HEAD` to get the current branch name
2. Run `git diff main...HEAD --stat` to see which files changed and how much
3. Run `git diff main...HEAD` to read the actual changes
4. Run `git log main...HEAD --oneline` to see the commit history
5. Read the spec file in `.kiro/specs/` — look for `requirements.md` and `tasks.md`
6. Identify which spec task was just completed (the most recently checked task,
   or the task whose acceptance criteria match the diff)
7. Read any changed controller files to extract the new or modified endpoints
8. Produce the PR description in the format below

## PR Description Format

Produce the description inside a code block so the developer can copy it directly.

    ## Summary

    [2-3 sentences describing what this PR implements. Write for a non-technical
    stakeholder — avoid implementation details. Focus on the user-visible outcome.]

    ## Spec Task Completed

    | Field | Value |
    |---|---|
    | Task ID | [from tasks.md] |
    | Task name | [from tasks.md] |
    | Spec | [spec folder name in .kiro/specs/] |

    ## What Changed

    | File | Change |
    |---|---|
    | [filename] | [one-line description of what changed and why] |
    | ... | ... |

    ## New or Modified Endpoints

    [List any new or changed REST endpoints. Include HTTP method, path, and
    one-line description. If no endpoints changed, write: "No endpoint changes."]

    | Method | Path | Description |
    |---|---|---|
    | POST | /api/v1/... | ... |

    ## Acceptance Criteria Status

    [Copy the acceptance criteria from the spec's requirements.md.
    Mark each one as Addressed ✅ or Pending ⏳.]

    | Criteria | Status |
    |---|---|
    | [WHEN condition THEN outcome from spec] | ✅ Addressed |
    | [WHEN condition THEN outcome from spec] | ⏳ Pending |

    ## How to Test

    1. Checkout this branch: `git checkout [branch-name]`
    2. Start the application: `mvn spring-boot:run`
    3. Run tests: `mvn test -q`
    4. [Add 1-3 specific curl commands or UI steps to verify the feature]

    ## Reviewer Checklist

    - [ ] Constructor injection used (no `@Autowired` on fields)
    - [ ] `@Valid` present on all `@RequestBody` parameters
    - [ ] Tests cover both success and error cases
    - [ ] No hardcoded credentials or secrets
    - [ ] Javadoc present on all new public methods
    - [ ] No business logic inside `@RestController` classes
    - [ ] Spec acceptance criteria addressed (see table above)

## Rules

- Always run the git commands yourself — never guess what changed
- If no spec file exists in `.kiro/specs/`, omit the spec sections and note:
  "No spec found — add acceptance criteria manually"
- If the diff is empty (no commits beyond main), output:
  "No changes detected between this branch and main. Nothing to describe."
- Write the Summary for a stakeholder, not a developer
- Keep the Reviewer Checklist exactly as shown — do not add or remove items
- Output the full PR description as a single markdown code block the developer
  can copy and paste into GitHub, GitLab, or Bitbucket
````

### Test the PR Description Agent

Complete a spec task in Kiro (mark any task checkbox in your spec), then in the Kiro chat panel:

```
@pr-description-agent Generate a PR description for the current branch
```

Or directly, without a completed spec task:

```
@pr-description-agent Generate a PR description — I've just finished 
implementing Task 2 from the inventory-service spec
```

**What to look for:**
- The agent runs `git diff` commands (you can see the shell calls)
- The Summary is written in plain English, not technical jargon
- The acceptance criteria table is pulled from the actual spec file, not invented
- The output is a single copy-pasteable markdown block

---

## Updated Lab 5 Completion Criteria

Add these two items to the Lab 5 checklist:

- [ ] `security-agent.md` exists with `tools: [read, shell]`
- [ ] `pr-description-agent.md` exists with `tools: [read, shell]`
- [ ] `@security-agent` returns `BLOCKED` when a credential is staged, `CLEAR` when clean
- [ ] `@pr-description-agent` produces a complete PR description from the actual git diff

## Updated Agent Directory

After adding both files, your `.kiro/agents/` folder should contain five files:

```
.kiro/agents/
├── code-review-agent.md        # read only — reviews against steering standards
├── test-generator-agent.md     # read + write + shell — generates and runs tests
├── docs-agent.md               # read + write — Javadoc, README, OpenAPI annotations
├── security-agent.md           # read + shell — staged file security scan ← NEW
└── pr-description-agent.md     # read + shell — PR description from diff + spec ← NEW
```

Commit both new agents:

```bash
git add .kiro/agents/security-agent.md .kiro/agents/pr-description-agent.md
git commit -m "feat: Layer 3 — add security-agent and pr-description-agent

security-agent (read + shell):
- Scans only staged files (git diff --cached)
- Checks: hardcoded credentials, SQL injection, missing @Valid, PII in logs
- Returns BLOCKED or CLEAR verdict for hook integration

pr-description-agent (read + shell):
- Reads git diff, commit log, and spec acceptance criteria
- Generates structured PR description in copy-pasteable markdown
- Invoked by pr-description-on-push hook on postTaskExecution"

git push origin main
```

---

## How the Full 5-Agent Suite Maps to the 5 Hooks

Once all five agents are in place, the hook-to-agent mapping is complete:

| Hook | Trigger | Agent invoked |
|---|---|---|
| `javadoc-on-save` | `fileEdited` (*.java in src/main) | `docs-agent` |
| `test-on-new-class` | `fileCreated` (*.java in src/main) | `test-generator-agent` |
| `security-lint-on-commit` | `promptSubmit` | `security-agent` |
| `pr-description-on-push` | `postTaskExecution` | `pr-description-agent` |
| `doc-sync-on-stop` | `userTriggered` | `docs-agent` |

Each hook's `askAgent` prompt is now handled by a specialist agent with exactly the tool access that task requires — not the general Kiro agent with unbounded access.

---

*Lab 5 Supplement | Day 2 | Layer 3 — Custom Agents | May 2026*