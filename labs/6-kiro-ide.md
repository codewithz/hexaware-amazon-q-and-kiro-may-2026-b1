# Module 2.1 — Kiro IDE Orientation


**Day:** Day 2  
**Goal:** Every participant has Kiro IDE open, connected to their AWS account, and has verified their Day 1 steering files are loaded

---

## What Is Kiro IDE?

Kiro is a purpose-built IDE for agentic software development, built on the VS Code engine. It looks and behaves like VS Code — same keybindings, same extension ecosystem, same file explorer — but adds three capabilities that VS Code with Q Developer cannot provide natively:

| Capability | VS Code + Q Developer | Kiro IDE |
|---|---|---|
| Custom agents with tool scope | ❌ | ✅ `.kiro/agents/` |
| Hook pipeline (event-driven automation) | ❌ | ✅ `.kiro/hooks/` |
| Steering files (persistent agent memory) | `.amazonq/rules/` (limited) | ✅ `.kiro/steering/` (first-class) |
| Spec-driven workflow | Manual | ✅ Native spec runner |
| Amazon Q Developer integration | ✅ | ✅ Same underlying model |

Everything you built on Day 1 works in Kiro — the same steering files, the same specs, the same codebase.

---

## First Launch Checklist

Work through these steps at the start of the session. The trainer will do this live.

### Step 1 — Open Kiro

Launch Kiro from your Applications folder or terminal:

```bash
kiro
```

On first launch you will see the **Welcome** panel.

### Step 2 — Import VS Code Settings

Kiro detects your VS Code installation and offers to import:
- Extensions
- Keybindings
- Themes and fonts
- User snippets

Click **Import from VS Code**. This takes 30–60 seconds.

> If you skip this, you can do it later: **File → Preferences → Import from VS Code**

### Step 3 — Sign In with AWS

Kiro authenticates with your AWS account the same way Q Developer does.

1. Click the **Kiro** icon in the left sidebar (the purple icon)
2. Click **Sign In**
3. Choose **AWS Builder ID** (for individual) or **IAM Identity Center** (for enterprise SSO)
4. Complete the browser authentication flow

**Verify sign-in:** The Kiro sidebar should show your account name and region. The chat panel should be active.

> **Kiro Pro check:** If your organisation has Amazon Q Developer Pro, sign in with your IAM Identity Center credentials — Kiro Pro is included. If you're on Builder ID (free), you have 50 monthly agentic interactions.

### Step 4 — Open Your Training Repository

```
File → Open Folder → [select your training repo from Day 1]
```

This is the same repository where you created steering files and specs on Day 1.

### Step 5 — Verify Steering Files Are Loaded

Open the Kiro chat panel and type:

```
What steering files do you have loaded for this project?
```

Kiro should respond listing your three steering files:
- `tech-stack.md`
- `architecture.md`
- `testing-standards.md`

If Kiro does not list them, check that they exist in `.kiro/steering/`:

```bash
ls .kiro/steering/
# tech-stack.md  architecture.md  testing-standards.md
```

### Step 6 — Explore the Kiro Interface

Take 5 minutes to locate:

| UI Element | Location | Purpose |
|---|---|---|
| Kiro Chat | Left sidebar → Kiro icon | Main agent chat panel |
| Agent selector | Top of chat panel | Switch between custom agents |
| Spec runner | Left sidebar → Spec icon | View and run your spec tasks |
| Hooks panel | Left sidebar → Hooks icon | View active hooks and last run status |
| Agent files | Explorer → `.kiro/` folder | Browse agents, hooks, steering, specs |

---

## How Kiro Differs in Daily Use

### Asking the Agent

In Kiro, you talk to the agent the same way as in Q Developer — through the chat panel. The difference is context:

**Q Developer (Day 1):**
```
Build a REST endpoint for inventory reservation
```
The agent has no memory of your standards unless you paste them every time.

**Kiro (Day 2):**
```
Build a REST endpoint for inventory reservation
```
The agent has already read your steering files. It knows your package structure, your naming conventions, your error format, your testing requirements — without you repeating them.

### Invoking Custom Agents

In Kiro, you can direct a task to a specific custom agent using `@agent-name`:

```
@code-review-agent Review the InventoryService class I just wrote
```

Kiro will automatically route to your `code-review-agent.md` agent definition, which you will build in Lab 5.

### The Spec Runner

If you have an open spec (from Lab 3 or Lab 7), the spec runner shows:
- Each approved task
- Status: pending / in-progress / complete
- Which agent is assigned
- A **Run** button to execute the task

---

## Common Issues at First Launch

| Issue | Fix |
|---|---|
| Kiro chat panel is greyed out | Not signed in — complete Step 3 |
| Steering files not listed | Check `.kiro/steering/` exists and has `.md` files |
| Missing extensions from VS Code | Re-run **Import from VS Code** from Preferences |
| Sign-in loops in browser | Clear browser cookies for `aws.amazon.com`, try again |
| "No active workspace" error | Open a folder first (Step 4), then sign in |

---

## Orientation Completion Criteria

Before the session moves to Module 2.2, verify:

- [ ] Kiro open and authenticated
- [ ] Repository from Day 1 loaded
- [ ] Steering files confirmed loaded (Kiro can list them)
- [ ] Explorer shows `.kiro/` folder with `steering/` and `specs/` subfolders
- [ ] Kiro chat responds to a test message
