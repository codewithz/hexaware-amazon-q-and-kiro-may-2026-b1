# Lab 6 — Building the 5-Hook Automation Pipeline

**Day:** 2  
**Layer:** Layer 4 (Agent Hooks)  
**Duration:** 75 minutes (13:00–14:15)  
**Tool:** Kiro IDE  
**Deliverable:** Five hooks committed to `.kiro/hooks/` that automate Javadoc, test generation, security scanning, PR descriptions, and documentation sync

---

## What You Will Learn

By the end of this lab you will have:
- Created the `.kiro/hooks/` directory and written all 5 hook definition files
- Understood the three hook event types: file save, commit, and manual trigger
- Understood the difference between `autopilot` and `supervised` hook modes
- Seen each hook fire in real time by triggering the relevant event
- Understood how hooks turn individual agents into a team-wide automation pipeline

---

## What Are Hooks?

Agents respond when you invoke them. Hooks respond when **events happen**.

Without hooks, a developer has to remember to:
- Run the code review agent before committing
- Ask the docs agent to update Javadoc after changing a class
- Ask the test generator agent to create tests for a new class
- Generate a PR description before opening a PR

With hooks, these things happen automatically. The developer saves a file, commits, or pushes — and the pipeline fires.

The 5 hooks you will build today cover the full development cycle:

| Hook | Trigger | Agent | Mode |
|---|---|---|---|
| `javadoc-on-save` | Any `.java` file saved | `docs-agent` | Autopilot |
| `test-on-new-class` | New class created in `src/main/` | `test-generator-agent` | Supervised |
| `security-lint-on-commit` | Developer types "ready to commit" | `security-agent` (preview) | Supervised |
| `pr-description-on-push` | Spec task marked complete | `pr-agent` (preview) | Autopilot |
| `doc-sync-on-stop` | Manual trigger | `docs-agent` | Supervised |

**Autopilot:** The hook fires and the agent acts without asking permission. Use for low-risk, easily-reversible operations (adding Javadoc, generating a PR description).

**Supervised:** The hook fires but shows you what it wants to do and waits for approval. Use for operations you want to review before they happen (running tests, security checks).

---

## Step 1 — Create the Hooks Directory

```bash
mkdir -p .kiro/hooks

# Verify
ls -la .kiro/
# Expected: agents/  hooks/  specs/  steering/
```

---

## Step 2 — Hook 1: `javadoc-on-save`

This hook fires every time a Java source file is saved. It invokes the docs agent to add or update Javadoc on changed methods. Because Javadoc is purely additive and easily reviewed in a diff, this runs in autopilot mode.

Create `.kiro/hooks/javadoc-on-save.md`:

```bash
touch .kiro/hooks/javadoc-on-save.md
code .kiro/hooks/javadoc-on-save.md
```

Paste this content:

```text
---
name: Javadoc on Save
description: Automatically adds or updates Javadoc on public methods when a Java source file is saved.
trigger:
  type: fileEvent
  event: onSave
  filePattern: "src/main/java/**/*.java"
agent: docs-agent
mode: autopilot
enabled: true
---

# Hook Instructions

When this hook fires, you have been given a Java source file that was just saved.

## Your Task
1. Read the file that was saved
2. Identify all public methods and public classes that are missing Javadoc, or where Javadoc is outdated (does not match the current parameters or return type)
3. Add or update Javadoc for only those methods — do not change any other code
4. Save the file with the updated Javadoc

## Rules
- Only add Javadoc to `public` and `protected` methods — never to private methods
- Do not rewrite existing Javadoc that is already accurate
- Do not add implementation comments inside method bodies
- Do not rename parameters or change method signatures
- If a method is trivial (getter/setter), skip it
- Write Javadoc that accurately describes what the method does — read the implementation first

## Output
After completing: report which methods had Javadoc added or updated, and which were already correct.
```

### Test Hook 1

1. Open `InventoryService.java` in Kiro IDE
2. Make a trivial change to a method — add a space somewhere, then remove it (just to trigger a save)
3. Save the file (Ctrl+S / Cmd+S)
4. Watch the Kiro panel — the hook should fire automatically
5. Check the file — Javadoc should have been added or updated on public methods

---

## Step 3 — Hook 2: `test-on-new-class`

This hook fires when a new Java class is created in the `src/main/` directory. It invokes the test generator agent to create a corresponding test class. This runs in **supervised mode** because test generation is a non-trivial operation — you want to review the proposed tests before they are written.

Create `.kiro/hooks/test-on-new-class.md`:

```bash
touch .kiro/hooks/test-on-new-class.md
code .kiro/hooks/test-on-new-class.md
```

Paste this content:

```text
---
name: Test on New Class
description: Proposes a test skeleton when a new Java class is created in the main source directory.
trigger:
  type: fileEvent
  event: onCreate
  filePattern: "src/main/java/**/*.java"
agent: test-generator-agent
mode: supervised
enabled: true
---

# Hook Instructions

A new Java class has just been created in the project. Your job is to propose
a test class for it and wait for the developer's approval before writing it.

## Your Task
1. Read the new class that was created
2. Identify what type of class it is:
   - **Service class** → generate unit tests with Mockito mocks
   - **Controller class** → generate MockMvc integration tests
   - **Repository class** → generate Testcontainers integration tests
   - **Entity or DTO class** → skip (no tests needed for data classes)
   - **Utility or helper class** → generate unit tests
3. Propose the test class — show the developer what tests you intend to write
4. Wait for approval before writing anything

## What to Show Before Acting
Present a plan like this:

~~~~
New class detected: InventoryService (service class)
Proposed test file: src/test/java/com/training/service/InventoryServiceTest.java

Proposed tests:
1. shouldReturnStockLevel_whenProductExists
2. shouldThrow404Exception_whenProductNotFound
3. [more tests...]

Shall I create the test file? (Type 'yes' to proceed, or describe changes you want)
~~~~

## After Approval
Once the developer approves:
1. Write the test file at `src/test/java/[same package]/[ClassName]Test.java`
2. Run `mvn test -Dtest=[TestClassName] -q`
3. Report the results
4. If any tests fail, show the failure and fix the test

## Rules
- Always follow testing-standards.md naming and structure
- Never create tests for entities, DTOs, or records
- Always propose before acting — do not create files without approval
```

### Test Hook 2

1. Create a new empty Java class in `src/main/java/com/training/service/`:
   ```bash
   touch src/main/java/com/training/service/PricingService.java
   ```
2. Add a minimal class body:
   ```java
   package com.training.service;
   public class PricingService {
       public double calculateDiscount(double price, int quantity) {
           return quantity > 10 ? price * 0.1 : 0.0;
       }
   }
   ```
3. Save the file
4. Watch the Kiro panel — the hook fires and proposes tests
5. Review the proposed tests and type `yes` to approve if they look correct

---

## Step 4 — Hook 3: `security-lint-on-commit`

This hook fires when you type specific words in the Kiro chat — in this case, "ready to commit". It uses a prompt event trigger rather than a file event. This means the hook is under your control — it fires when you decide you are ready to commit, not automatically.

This runs in **supervised mode** because a security finding should always involve a human decision before proceeding.

Create `.kiro/hooks/security-lint-on-commit.md`:

```bash
touch .kiro/hooks/security-lint-on-commit.md
code .kiro/hooks/security-lint-on-commit.md
```

Paste this content:

```text
---
name: Security Lint on Commit
description: Runs a targeted security scan on staged files when the developer indicates they are ready to commit. Blocks the commit workflow if Critical or High issues are found.
trigger:
  type: promptEvent
  pattern: "ready to commit"
agent: security-agent
mode: supervised
enabled: true
---

# Hook Instructions

The developer has indicated they are ready to commit. Run a security scan on
all files that are currently staged (`git diff --cached --name-only`).

## Your Task
1. Run `git diff --cached --name-only` to get the list of staged files
2. Read each staged Java file
3. Scan for these security issues:
   - **Hardcoded secrets**: passwords, API keys, tokens, connection strings in source files
   - **SQL injection risk**: string concatenation in queries
   - **Missing input validation**: REST controller methods accepting user input without `@Valid`
   - **Sensitive data exposure**: PII written to log statements
   - **Insecure deserialization**: `ObjectInputStream` or unvalidated JSON parsing

4. Report findings in this format:

~~~~
## Pre-Commit Security Scan

Files scanned: [N]
Issues found: [N]

### Critical Issues (BLOCK COMMIT)
[Issue details with file:line, description, fix required]

### High Issues (BLOCK COMMIT)  
[Issue details]

### Medium Issues (warn but allow commit)
[Issue details]

### Verdict
BLOCKED — fix [N] critical/high issues before committing
OR
CLEAR — no critical or high issues found. Safe to commit.
~~~~

5. If verdict is BLOCKED: do not proceed. Wait for the developer to fix the issues.
6. If verdict is CLEAR: inform the developer they can proceed.

## Rules
- Shell access is for `git diff --cached --name-only` only — do not modify, commit, or push anything
- A single Critical issue is always a BLOCK
- A single High issue is always a BLOCK
- Never mark a scan as CLEAR if you found Critical or High issues
```

### Test Hook 3

1. Stage a file with a deliberate issue (add a hardcoded password temporarily):
   ```java
   // Temporarily add this to test the hook
   private static final String DB_PASSWORD = "mypassword123"; // hardcoded - test only
   ```
2. Stage the file: `git add src/main/java/com/training/service/InventoryService.java`
3. In the Kiro chat panel, type: `ready to commit`
4. The hook should fire and report the hardcoded password as a Critical issue
5. Remove the hardcoded password and verify the hook clears on the next run

---

## Step 5 — Hook 4: `pr-description-on-push`

This hook fires when a spec task is marked as complete. It invokes the PR agent to generate a pull request description from the git diff and the completed spec tasks.

Create `.kiro/hooks/pr-description-on-push.md`:

```bash
touch .kiro/hooks/pr-description-on-push.md
code .kiro/hooks/pr-description-on-push.md
```

Paste this content:

```text
---
name: PR Description on Push
description: Automatically generates a pull request description from the current branch diff and completed spec tasks when a spec task is marked complete.
trigger:
  type: specTaskEvent
  event: onTaskComplete
agent: pr-agent
mode: autopilot
enabled: true
---

# Hook Instructions

A spec task has just been marked as complete. Generate a pull request description
that summarises the changes made to implement this task.

## Your Task
1. Run `git diff main...HEAD --stat` to see which files changed
2. Run `git diff main...HEAD` to read the actual changes
3. Read the spec file in `.kiro/specs/` to understand which task was completed
   and which acceptance criteria it addressed
4. Generate a PR description using this format:

~~~~markdown
## Summary
[2-3 sentences describing what was implemented and why]

## Spec Task Completed
[Task ID and name from the spec]

## What Changed
- `[FileName]`: [What changed and why]
- `[FileName]`: [What changed and why]

## Acceptance Criteria Addressed
- [ ] [Criterion 1 from spec — ✅ if addressed by this PR, ⬜ if still pending]
- [ ] [Criterion 2 from spec]

## How to Test
1. [Step to reproduce the feature]
2. [Step to verify the acceptance criteria]

## Reviewer Checklist
- [ ] Constructor injection used (no @Autowired on fields)
- [ ] All new endpoints have @Valid on request body
- [ ] Test coverage includes happy path and at least one error case
- [ ] No hardcoded credentials
- [ ] Javadoc present on all public methods
~~~~

5. Output the PR description to the Kiro chat panel so the developer can copy it

## Rules
- Shell access is for git read commands only — do NOT push, commit, or modify files
- If there are no uncommitted changes, say so and skip the description
- The summary must be understandable by a non-technical stakeholder
```

### Test Hook 4

1. In the Kiro spec runner (left sidebar → Spec icon), open your inventory-service spec
2. Mark one task as complete (click the checkbox)
3. The hook fires and generates a PR description in the chat panel
4. Review the generated description — verify it references the correct spec task and acceptance criteria

---

## Step 6 — Hook 5: `doc-sync-on-stop`

This hook is a **manual trigger**. It fires when you click the play button in the Kiro Hooks panel, not automatically. It does a comprehensive documentation sync at the end of a work session.

Create `.kiro/hooks/doc-sync-on-stop.md`:

```bash
touch .kiro/hooks/doc-sync-on-stop.md
code .kiro/hooks/doc-sync-on-stop.md
```

Paste this content:

```text
---
name: Doc Sync on Stop
description: Performs a comprehensive documentation sync at the end of a work session. Updates README, Javadoc, and OpenAPI annotations. Run manually before pushing a branch.
trigger:
  type: manual
agent: docs-agent
mode: supervised
enabled: true
---

# Hook Instructions

This is an end-of-session documentation sync. Read the entire changed codebase
and bring all documentation up to date.

## Your Task
1. Run `git diff main...HEAD --name-only` to get the list of changed files
2. For each changed Java file:
   - Check if all public methods have accurate Javadoc
   - Check if any method signature changed (parameters, return type) — Javadoc needs updating
   - Add or update Javadoc where needed
3. For each changed controller file:
   - Check if SpringDoc @Operation and @ApiResponse annotations are present
   - Add missing annotations
4. Read `README.md` and compare the API Endpoints table with the current controller files
   - Add any endpoints that are missing from the table
   - Remove any endpoints that no longer exist
   - Update descriptions for endpoints that changed behaviour

## Before Acting
Present a summary of what you intend to change:

~~~~
Documentation sync plan:
- InventoryService.java: 3 methods need Javadoc (createReservation, releaseReservation, fulfilReservation)
- InventoryController.java: Missing @ApiResponse for 404 on GET /products/{id}/stock
- README.md: Missing endpoint: DELETE /api/v1/inventory/reservations/{id}

Shall I proceed? (Type 'yes' to apply all changes)
~~~~

## After Acting
Report exactly what was changed, file by file.

## Rules
- Never change business logic — only documentation
- Always show the plan before applying changes
- Do not add redundant Javadoc to simple getters/setters
```

### Test Hook 5

1. In the Kiro panel, navigate to the **Hooks** section
2. Find `doc-sync-on-stop` and click the **play button** (▶)
3. The hook fires and presents its sync plan
4. Review the plan and type `yes` to approve
5. Verify that README.md and Javadoc are updated

---

## Step 7 — Verify All Hooks Are Active

In Kiro IDE, open the Hooks panel (left sidebar → Hooks icon). You should see all 5 hooks listed:

| Hook | Trigger | Mode | Status |
|---|---|---|---|
| javadoc-on-save | File save (*.java) | Autopilot | ✅ Enabled |
| test-on-new-class | File create (src/main/) | Supervised | ✅ Enabled |
| security-lint-on-commit | Prompt ("ready to commit") | Supervised | ✅ Enabled |
| pr-description-on-push | Spec task complete | Autopilot | ✅ Enabled |
| doc-sync-on-stop | Manual | Supervised | ✅ Enabled |

If any hook shows as disabled or errored, open its `.md` file and check the frontmatter syntax.

---

## Step 8 — Commit the Hooks

```bash
# Verify all 5 files exist
ls -la .kiro/hooks/
# Expected:
# javadoc-on-save.md
# test-on-new-class.md
# security-lint-on-commit.md
# pr-description-on-push.md
# doc-sync-on-stop.md

git add .kiro/hooks/
git commit -m "feat: Layer 4 — 5-hook automation pipeline committed

Hook 1: javadoc-on-save (autopilot, onSave, *.java)
  → docs-agent adds Javadoc automatically on every save

Hook 2: test-on-new-class (supervised, onCreate, src/main/**/*.java)
  → test-generator-agent proposes tests for new classes

Hook 3: security-lint-on-commit (supervised, promptEvent: 'ready to commit')
  → security-agent scans staged files, blocks on Critical/High

Hook 4: pr-description-on-push (autopilot, specTaskEvent: onTaskComplete)
  → pr-agent generates PR description from diff + spec tasks

Hook 5: doc-sync-on-stop (supervised, manual trigger)
  → docs-agent syncs README, Javadoc, and OpenAPI annotations"

git push origin main
```

---

## Lab Completion Criteria

- [ ] `.kiro/hooks/` directory exists with all 5 hook files
- [ ] `javadoc-on-save.md` fires on file save and adds Javadoc (verified)
- [ ] `test-on-new-class.md` proposes tests when a new class is created (verified)
- [ ] `security-lint-on-commit.md` fires when "ready to commit" is typed (verified)
- [ ] `pr-description-on-push.md` generates a PR description when a spec task completes (verified)
- [ ] `doc-sync-on-stop.md` fires on manual trigger and presents a sync plan (verified)
- [ ] Kiro Hooks panel shows all 5 hooks as enabled
- [ ] All 5 hooks committed and pushed