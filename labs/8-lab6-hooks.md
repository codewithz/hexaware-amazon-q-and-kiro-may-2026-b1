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
- Understood the correct Kiro hook file format (JSON, `.kiro.hook` extension)
- Understood the available trigger types: File Save, File Create, Prompt Submit, Post Task Execution, and Manual
- Understood the difference between `askAgent` (AI-powered) and `shellCommand` (deterministic) hook actions
- Seen each hook fire in real time by triggering the relevant event

> **Important — file format:** Kiro hook files are **JSON** files with the **`.kiro.hook`** extension. They are NOT Markdown files. If you create `.md` files in `.kiro/hooks/`, Kiro will not recognise them as hooks.

---

## What Are Hooks?

Agents respond when you invoke them. Hooks respond when **events happen**.

Without hooks, a developer has to remember to run the code review agent before committing, ask the docs agent to update Javadoc after changing a class, and generate a PR description before opening a PR.

With hooks, these things happen automatically. The developer saves a file or commits — and the pipeline fires.

The 5 hooks you will build today cover the full development cycle:

| Hook file | Trigger type | Action | When it fires |
|---|---|---|---|
| `javadoc-on-save.kiro.hook` | `fileEdited` | `askAgent` | Any `.java` file saved in `src/main/` |
| `test-on-new-class.kiro.hook` | `fileCreated` | `askAgent` | New `.java` file created in `src/main/` |
| `security-lint-on-commit.kiro.hook` | `promptSubmit` | `askAgent` | User submits any prompt |
| `pr-description-on-push.kiro.hook` | `postTaskExecution` | `askAgent` | A spec task is completed |
| `doc-sync-on-stop.kiro.hook` | `userTriggered` | `askAgent` | Manual click in the Hooks panel |

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

This hook fires every time a Java source file is saved in the main source directory. It sends a prompt to the docs agent to add or update Javadoc on any methods that are missing it.

Create `.kiro/hooks/javadoc-on-save.kiro.hook`:

```bash
touch .kiro/hooks/javadoc-on-save.kiro.hook
code .kiro/hooks/javadoc-on-save.kiro.hook
```

Paste this content **exactly** — this is valid JSON:

```json
{
  "name": "Javadoc on Save",
  "description": "Automatically adds or updates Javadoc on public methods when a Java source file is saved in the main source directory.",
  "version": "1",
  "when": {
    "type": "fileEdited",
    "patterns": [
      "src/main/java/**/*.java"
    ]
  },
  "then": {
    "type": "askAgent",
    "prompt": "A Java source file was just saved. Review the file and add or update Javadoc for any public or protected methods and classes that are missing it or have outdated Javadoc that does not match the current parameters or return type. Rules: only add Javadoc to public and protected members — never private; do not rewrite Javadoc that is already accurate; do not add comments inside method bodies; skip trivial getters and setters; read the implementation before writing the Javadoc so it is accurate. After completing, report which methods had Javadoc added or updated and which were already correct."
  }
}
```

### Verify Hook 1 Appears in the IDE

1. Open the **Kiro panel** (ghost icon in left sidebar)
2. Scroll to the **Agent Hooks** section
3. You should see **"Javadoc on Save"** listed
4. If it does not appear, check the file has the `.kiro.hook` extension and contains valid JSON (`cat .kiro/hooks/javadoc-on-save.kiro.hook | python3 -m json.tool`)

### Test Hook 1

1. Open `InventoryService.java` in Kiro IDE
2. Add a space anywhere inside a method, then save the file (Ctrl+S / Cmd+S)
3. Watch the Kiro panel — a task should appear automatically under Agent Hooks
4. Click **Task list** at the top of the chat panel to see the hook running
5. Check the file after — Javadoc should have been added to public methods

---

## Step 3 — Hook 2: `test-on-new-class`

This hook fires when a new Java class is created in `src/main/`. It proposes a test skeleton for the new class. The agent will show a plan and wait — it does not create files without your review because the `askAgent` action runs in the chat panel where you can respond.

Create `.kiro/hooks/test-on-new-class.kiro.hook`:

```bash
touch .kiro/hooks/test-on-new-class.kiro.hook
code .kiro/hooks/test-on-new-class.kiro.hook
```

Paste this content:

```json
{
  "name": "Test on New Class",
  "description": "Proposes a test skeleton when a new Java class is created in the main source directory. Shows a plan and waits for approval before writing any files.",
  "version": "1",
  "when": {
    "type": "fileCreated",
    "patterns": [
      "src/main/java/**/*.java"
    ]
  },
  "then": {
    "type": "askAgent",
    "prompt": "A new Java class was just created. Read the new class and identify what type it is: Service class (generate unit tests with Mockito), Controller class (generate MockMvc integration tests), Repository class (generate Testcontainers integration tests), Entity or DTO class (skip — no tests needed), or Utility class (generate unit tests). Then propose a test plan: show the proposed test file path and list the test method names you intend to write using the should[Behaviour]_when[Condition] naming convention from testing-standards.md. Ask the developer to confirm before creating any files. Only write the test file after the developer types 'yes'. After writing, run mvn test -Dtest=[TestClassName] -q and report the results."
  }
}
```

### Test Hook 2

1. Create a new Java class in `src/main/java/com/training/service/`:

```bash
cat > src/main/java/com/training/service/PricingService.java << 'EOF'
package com.training.service;

public class PricingService {
    public double calculateDiscount(double price, int quantity) {
        return quantity > 10 ? price * 0.1 : 0.0;
    }
}
EOF
```

2. Save the file — the hook fires automatically
3. In the Kiro chat panel, the agent proposes test method names
4. Type `yes` to approve — the test file is created and tests run

---

## Step 4 — Hook 3: `security-lint-on-commit`

This hook uses the `promptSubmit` trigger — it fires on **every prompt you submit** and appends security-awareness context to it. The most practical use is combining it with a phrase pattern check.

> **Note on `promptSubmit`:** This trigger fires on every prompt, not just "ready to commit." The hook appends its prompt to yours. The most effective pattern is to make the hook instruction conditional — it only acts meaningfully when the user's prompt contains commit-related language.

Create `.kiro/hooks/security-lint-on-commit.kiro.hook`:

```bash
touch .kiro/hooks/security-lint-on-commit.kiro.hook
code .kiro/hooks/security-lint-on-commit.kiro.hook
```

Paste this content:

```json
{
  "name": "Security Lint on Commit",
  "description": "Runs a targeted security scan on staged files when the developer's prompt indicates they are ready to commit.",
  "version": "1",
  "when": {
    "type": "promptSubmit"
  },
  "then": {
    "type": "askAgent",
    "prompt": "If the user's prompt contains words like 'commit', 'ready to commit', 'push', or 'stage', run a security scan on the staged files before proceeding with any other action. To do this: run git diff --cached --name-only to get staged files, read each staged Java file, and check for: hardcoded passwords or API keys, SQL string concatenation in queries, REST controller methods missing @Valid on @RequestBody parameters, email or PII written to log statements. Report findings with file name, line number, and severity (CRITICAL/HIGH/MEDIUM). If any CRITICAL or HIGH issues are found, output BLOCKED and do not proceed until the developer confirms they have fixed the issues. If no CRITICAL or HIGH issues are found, output CLEAR and continue with the original request. If the user's prompt has nothing to do with committing or pushing, ignore this instruction entirely and just respond to the original prompt normally."
  }
}
```

### Test Hook 3

1. Stage a file with a deliberate hardcoded secret:

```bash
# Add temporarily to InventoryService.java for testing:
# private static final String DB_PASSWORD = "mypassword123";
git add src/main/java/com/training/service/InventoryService.java
```

2. In the Kiro chat panel, type: `I'm ready to commit these changes`
3. The hook fires — the security scan should detect the hardcoded password and output `BLOCKED`
4. Remove the test password, then type: `ready to commit` again — should output `CLEAR`

---

## Step 5 — Hook 4: `pr-description-on-push`

This hook fires after a spec task is completed (`postTaskExecution`). It generates a pull request description from the current branch diff and the completed spec task context.

Create `.kiro/hooks/pr-description-on-push.kiro.hook`:

```bash
touch .kiro/hooks/pr-description-on-push.kiro.hook
code .kiro/hooks/pr-description-on-push.kiro.hook
```

Paste this content:

```json
{
  "name": "PR Description on Push",
  "description": "Generates a pull request description from the current branch diff and the completed spec task when a spec task is marked complete.",
  "version": "1",
  "when": {
    "type": "postTaskExecution"
  },
  "then": {
    "type": "askAgent",
    "prompt": "A spec task has just been marked as complete. Generate a pull request description by: (1) running git diff main...HEAD --stat to see which files changed, (2) running git diff main...HEAD to read the actual changes, (3) reading the spec file in .kiro/specs/ to identify the completed task and which acceptance criteria it addresses. Then produce a PR description with these sections: Summary (2-3 sentences understandable by a non-technical stakeholder), Spec Task Completed (task ID and name), What Changed (per file: what changed and why), Acceptance Criteria Addressed (list from spec, mark each as addressed or still pending), How to Test (numbered steps to verify the feature), and Reviewer Checklist (constructor injection used, @Valid on request bodies, tests cover error cases, no hardcoded credentials, Javadoc on public methods). Output the description to the chat panel so the developer can copy it."
  }
}
```

### Test Hook 4

1. Open the Kiro spec runner (left sidebar → Spec icon)
2. Open your `inventory-service` spec
3. Mark one task as complete by clicking its checkbox
4. The hook fires — a PR description appears in the chat panel
5. Verify it references the correct spec task and acceptance criteria

---

## Step 6 — Hook 5: `doc-sync-on-stop`

This hook uses `userTriggered` — it only fires when you manually click the play button in the Kiro Hooks panel. It does a comprehensive documentation sync at the end of a work session.

Create `.kiro/hooks/doc-sync-on-stop.kiro.hook`:

```bash
touch .kiro/hooks/doc-sync-on-stop.kiro.hook
code .kiro/hooks/doc-sync-on-stop.kiro.hook
```

Paste this content:

```json
{
  "name": "Doc Sync on Stop",
  "description": "Performs a comprehensive documentation sync at the end of a work session. Updates Javadoc, README endpoint table, and SpringDoc OpenAPI annotations for all changed files. Run manually before pushing a branch.",
  "version": "1",
  "when": {
    "type": "userTriggered"
  },
  "then": {
    "type": "askAgent",
    "prompt": "Perform an end-of-session documentation sync. Steps: (1) Run git diff main...HEAD --name-only to get all changed files in this branch. (2) For each changed Java file: check if all public methods have accurate Javadoc, check if any method signature changed so Javadoc needs updating, add or update Javadoc where needed. (3) For each changed controller file: check if SpringDoc @Operation and @ApiResponse annotations are present for all endpoints, add any that are missing. (4) Read README.md and compare its API Endpoints table to the current controller files: add missing endpoints, remove endpoints that no longer exist, update descriptions for endpoints that changed. Before making any changes, present a plan listing exactly what you intend to change in each file and wait for the developer to type 'yes' to proceed. After completing, report exactly what was changed file by file. Rules: never change business logic — documentation only; always show the plan before applying changes; skip trivial getters and setters."
  }
}
```

### Test Hook 5

1. In the Kiro panel, navigate to the **Agent Hooks** section
2. Find **"Doc Sync on Stop"** and click the **play button** (▶) next to it
3. The hook fires and the agent presents its sync plan in the chat panel
4. Review the plan — type `yes` to approve
5. Verify that README.md and Javadoc are updated

---

## Step 7 — Verify All Hooks Are Active

In Kiro IDE, open the **Agent Hooks** section in the Kiro panel. You should see all 5 hooks listed:

| Hook name | File | Status |
|---|---|---|
| Javadoc on Save | `javadoc-on-save.kiro.hook` | ✅ Active |
| Test on New Class | `test-on-new-class.kiro.hook` | ✅ Active |
| Security Lint on Commit | `security-lint-on-commit.kiro.hook` | ✅ Active |
| PR Description on Push | `pr-description-on-push.kiro.hook` | ✅ Active |
| Doc Sync on Stop | `doc-sync-on-stop.kiro.hook` | ✅ Active |

**If a hook does not appear:**

```bash
# Validate JSON syntax for each file
for f in .kiro/hooks/*.kiro.hook; do
  echo "Checking $f..."
  python3 -m json.tool "$f" > /dev/null && echo "  ✅ Valid JSON" || echo "  ❌ Invalid JSON"
done
```

Common mistakes that prevent hooks from loading:
- File has `.md` extension instead of `.kiro.hook`
- Trailing comma after the last key in a JSON object (invalid JSON)
- Using single quotes instead of double quotes in JSON
- Incorrect `"type"` value in the `"when"` block — must be exactly: `fileEdited`, `fileCreated`, `promptSubmit`, `postTaskExecution`, or `userTriggered`

---

## Step 8 — Where to See Hook Output

This is the most common point of confusion in this lab. Hook output does **not** appear in the main chat panel body where you normally talk to the agent. It has its own dedicated view. Here is exactly where to look.

---

### Location 1 — Task List (hook currently running)

When a hook fires, Kiro opens a background agent task. To see it:

1. Look at the **top of the Kiro chat panel** — there is a row of small icon buttons above the chat input
2. Click the **Task list** button (it looks like a bullet list icon, usually the second or third icon in that row)
3. You will see a section called **Current Task** — click it
4. The full agent conversation for the hook opens: every file the agent read, every change it made, every decision it took

This is the view to show your candidates when a hook fires and they say "nothing happened." Something almost certainly did happen — it just ran silently in the background. Task list is where it went.

**What it looks like while running:**

```
Current Task — Javadoc on Save
  ✅ Reading: src/main/java/com/training/service/InventoryService.java
  ✅ Writing: src/main/java/com/training/service/InventoryService.java
  → Added Javadoc to 3 methods: createReservation, releaseReservation, getStockLevel
  → 2 methods already had accurate Javadoc: findById, validateRequest
```

---

### Location 2 — History (hook already finished)

If the hook has already completed by the time you go to look for it:

1. In the Kiro chat panel, click the **History** button (clock icon, in the same row of buttons at the top of the panel)
2. Each past hook run appears as a separate entry, labelled with the hook name and a timestamp
3. Click any entry to expand the full agent conversation for that run

History persists for the entire session. You can go back and review what any hook did, including hooks that fired hours ago, even if you were not watching when they triggered.

---

### Location 3 — Agent Hooks Panel (confirm it fired at all)

Before checking Task list or History, first confirm the hook actually triggered:

1. Click the **ghost icon** in the left sidebar to open the Kiro panel
2. Scroll to the **Agent Hooks** section
3. Each hook shows a small status indicator
4. When a hook fires, its indicator briefly lights up or shows a spinner
5. After completion, it shows the last run time (e.g. "Last run: 2 minutes ago")

If the last run time does not update after you trigger the event, the hook did not fire. This means the trigger condition was not met — check the file pattern or trigger type.

---

### Location 4 — The actual files changed (for `askAgent` hooks)

For hooks that write files (like `javadoc-on-save` and `test-on-new-class`), the most direct confirmation is checking the file itself:

```bash
# See what changed after saving a Java file
git diff src/main/java/com/training/service/InventoryService.java
```

If the hook ran successfully, you will see Javadoc additions in the diff. If there is no diff, either the file already had accurate Javadoc (the hook ran but found nothing to change) or the hook did not fire.

---

### Reading a Hook Run End-to-End

Here is what a full hook run looks like in the Task list view, using `javadoc-on-save` as the example:

```
Hook fired: Javadoc on Save
Trigger: fileEdited — InventoryService.java was saved

Agent action:
  1. Reading file: src/main/java/com/training/service/InventoryService.java
  2. Checking public methods for missing Javadoc...
     - createReservation(CreateReservationRequest): ❌ Missing Javadoc
     - releaseReservation(UUID): ❌ Missing Javadoc  
     - getStockLevel(UUID): ✅ Javadoc already present and accurate
     - findById(UUID): Skipped (private method)
  3. Writing updated file with Javadoc added to 2 methods
  
Result: Added Javadoc to createReservation and releaseReservation.
        getStockLevel already had accurate documentation.
```

---

### What to Do If You See Nothing

Work through this checklist in order:

**1. Hook not in the Hooks panel at all**
- The file has `.md` extension instead of `.kiro.hook`
- The JSON is invalid — run `python3 -m json.tool .kiro/hooks/your-hook.kiro.hook`
- The `"type"` value is wrong — must be one of: `fileEdited`, `fileCreated`, `fileDeleted`, `promptSubmit`, `agentStop`, `preToolUse`, `postToolUse`, `preTaskExecution`, `postTaskExecution`, `userTriggered`

**2. Hook is in the panel but does not fire**
- For `fileEdited`/`fileCreated`: the file path does not match the `"patterns"` glob — test with `python3 -c "import fnmatch; print(fnmatch.fnmatch('src/main/java/com/training/service/InventoryService.java', 'src/main/java/**/*.java'))"`
- For `promptSubmit`: the hook fires on every prompt — if you do not see it, you may be looking in the wrong place (check Task list, not the main chat)
- For `userTriggered`: you must click the ▶ play button next to the hook in the Agent Hooks panel — it will not fire automatically

**3. Hook fires but output is empty**
- The agent ran but had nothing to do (e.g. all Javadoc was already present)
- Check History to see the agent's reasoning — it will explain why it made no changes

**4. Hook fires but changes are wrong**
- The hook prompt needs to be more specific — edit the `"prompt"` field in the `.kiro.hook` file and save; changes take effect immediately without restarting Kiro

---

### Quick Reference Table

| What you want to see | Where to look |
|---|---|
| Hook running right now | Chat panel top → **Task list** → Current Task |
| Hook that already finished | Chat panel top → **History** → select the hook run |
| Whether the hook fired at all | Kiro panel → **Agent Hooks** → check last run time |
| What files the hook changed | Terminal → `git diff` |
| Why the hook is not appearing | Run JSON validator → check `"type"` value |
| Why the hook is not firing | Check file pattern glob matches your file path |

---

## Step 9 — Commit the Hooks

```bash
# Verify all 5 files exist with the correct extension
ls -la .kiro/hooks/
# Expected:
# javadoc-on-save.kiro.hook
# test-on-new-class.kiro.hook
# security-lint-on-commit.kiro.hook
# pr-description-on-push.kiro.hook
# doc-sync-on-stop.kiro.hook

git add .kiro/hooks/
git commit -m "feat: Layer 4 — 5-hook automation pipeline

Hook 1: javadoc-on-save (fileEdited, src/main/**/*.java)
  → askAgent: adds Javadoc to public methods on every save

Hook 2: test-on-new-class (fileCreated, src/main/**/*.java)
  → askAgent: proposes test skeleton, waits for approval before writing

Hook 3: security-lint-on-commit (promptSubmit)
  → askAgent: scans staged files when commit language detected, BLOCKED/CLEAR verdict

Hook 4: pr-description-on-push (postTaskExecution)
  → askAgent: generates PR description from diff + spec task on task completion

Hook 5: doc-sync-on-stop (userTriggered)
  → askAgent: syncs Javadoc, README, and OpenAPI annotations on demand"

git push origin main
```

---

## Hook Format Reference

Every Kiro hook file follows this JSON structure:

```json
{
  "name": "Human-readable name shown in the Hooks panel",
  "description": "What this hook does — one sentence",
  "version": "1",
  "when": {
    "type": "triggerType",
    "patterns": ["optional/glob/pattern/**/*.java"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "The instruction sent to the agent when the hook fires."
  }
}
```

**Valid `when.type` values for Kiro IDE:**

| Type | When it fires | Supports `patterns`? |
|---|---|---|
| `fileEdited` | A file matching the pattern is saved/edited | ✅ Yes |
| `fileCreated` | A file matching the pattern is created | ✅ Yes |
| `fileDeleted` | A file matching the pattern is deleted | ✅ Yes |
| `promptSubmit` | The user submits any prompt | ❌ No |
| `agentStop` | The agent finishes responding | ❌ No |
| `preToolUse` | Before the agent invokes a tool | ❌ No |
| `postToolUse` | After the agent invokes a tool | ❌ No |
| `preTaskExecution` | Before a spec task starts | ❌ No |
| `postTaskExecution` | After a spec task completes | ❌ No |
| `userTriggered` | Manual — you click the play button | ❌ No |

**Valid `then.type` values:**

| Type | What it does | Uses credits? |
|---|---|---|
| `askAgent` | Sends a prompt to the AI agent | ✅ Yes |
| `shellCommand` | Runs a shell command directly | ❌ No |

---

## Lab Completion Criteria

- [ ] `.kiro/hooks/` directory exists with all 5 files using the `.kiro.hook` extension
- [ ] All 5 hook files contain valid JSON (verified with `python3 -m json.tool`)
- [ ] All 5 hooks appear in the Kiro IDE **Agent Hooks** panel
- [ ] `javadoc-on-save` fires when a `.java` file is saved — Javadoc added (verified)
- [ ] `test-on-new-class` fires when a new class is created — test plan proposed (verified)
- [ ] `security-lint-on-commit` fires on a commit-related prompt — security findings reported (verified)
- [ ] `pr-description-on-push` fires when a spec task completes — PR description generated (verified)
- [ ] `doc-sync-on-stop` fires on manual trigger — sync plan presented (verified)
- [ ] All 5 hooks committed and pushed