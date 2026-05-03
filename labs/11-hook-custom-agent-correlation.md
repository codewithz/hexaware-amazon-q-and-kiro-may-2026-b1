# Lab: Connecting Hooks to Custom Agents in Kiro IDE

**Duration:** 30 minutes  
**Level:** Beginner  
**Prerequisites:** Kiro IDE installed, Java/Spring Boot project open

---

## Objectives

By the end of this lab you will be able to:

- Explain how a Kiro hook invokes a custom agent
- Identify why the `prompt` field is the critical link between a hook and an agent
- Write a hook prompt that correctly targets a named custom agent
- Avoid the common mistake of hooks that trigger but never reach the intended agent

---

## Background: How Hooks Find Agents

In Kiro IDE, a **hook** detects an event and a **custom agent** defines the behaviour. But the hook does not reference the agent by a dedicated field like `"agent": "docs-agent"`. Instead, the connection is made entirely through the `prompt` field inside the `then` block.

When a hook fires, Kiro reads the `prompt` and uses it to determine which agent to invoke. If your prompt does not mention the agent by name, Kiro will not route the request to your custom agent — it will fall back to a generic response or do nothing useful.

This is the single most common reason a hook appears to fire but produces no result: **the prompt does not name the agent**.

---

## The Two Files That Work Together

| File | Location | Purpose |
|---|---|---|
| Agent definition | `.kiro/agents/docs-agent.md` | Defines what the agent does and how |
| Hook definition | `.kiro/hooks/javadoc-on-save.json` | Defines when to run and what to ask |

The `name` field in the agent's front matter and the agent name written inside the hook's `prompt` must match exactly.

---

## Part 1 — The Agent Definition

File: `.kiro/agents/docs-agent.md`

```markdown
---
name: docs-agent
description: Keeps Javadoc, README, and API documentation in sync with the codebase.
tools: ["read", "write"]
---

# Docs Agent

You are a technical writer embedded in a Java/Spring Boot development team.
...
```

The `name: docs-agent` in the front matter is the identifier you must use in your hook prompt. The agent will not be called unless this name appears in the prompt.

---

## Part 2 — The Hook Definition (Broken Version)

This is what a hook looks like when it fires but never reaches the custom agent:

```json
{
  "enabled": true,
  "name": "Javadoc on Save",
  "when": {
    "type": "fileEdited",
    "patterns": ["src/main/java/**/*.java"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "Review the saved file and add or update Javadoc for all public classes and methods. Update README.md if any REST endpoints were added or modified. Add SpringDoc annotations to controllers if missing."
  }
}
```

**What goes wrong:** The prompt describes a task but does not name `docs-agent`. Kiro has no way to know which agent to invoke. The hook fires on every Java save, but your custom agent never runs.

---

## Part 3 — The Hook Definition (Working Version)

Adding the agent name to the prompt is the fix:

```json
{
  "enabled": true,
  "name": "Javadoc on Save",
  "description": "Invokes docs-agent to update Javadoc, README, and SpringDoc annotations on Java file save.",
  "version": "1",
  "when": {
    "type": "fileEdited",
    "patterns": ["src/main/java/**/*.java"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "Invoke the docs-agent to review the saved file. The docs-agent should add or update Javadoc for all public classes and methods, update README.md if any REST endpoints were added or modified, and add SpringDoc OpenAPI annotations to controllers if missing."
  }
}
```

**What changed:** The prompt now opens with `Invoke the docs-agent`. Kiro reads this, resolves `docs-agent` against `.kiro/agents/docs-agent.md`, and routes the full task to your custom agent and its instruction set.

---

## Part 4 — Execution Flow

When you save `src/main/java/com/example/UserController.java`:

```
1. Kiro detects a fileEdited event on UserController.java

2. Hook pattern matches: src/main/java/**/*.java ✓

3. Kiro reads the prompt:
   "Invoke the docs-agent to review the saved file..."

4. Kiro resolves "docs-agent" → .kiro/agents/docs-agent.md ✓

5. docs-agent reads its instruction body:
   - Javadoc template and rules
   - README endpoint table format
   - SpringDoc annotation rules
   - Constraints (no business logic changes, no private methods, etc.)

6. docs-agent reads UserController.java using its read tool

7. docs-agent writes updated Javadoc and annotations using its write tool

8. If endpoints changed, docs-agent updates README.md
```

Without step 3 naming the agent, the chain breaks at step 4 and your custom agent is never loaded.

---

## Part 5 — The Prompt Is the Contract

The prompt in the hook serves two purposes simultaneously:

**Routing** — it tells Kiro which custom agent to invoke  
**Tasking** — it tells that agent what to do for this specific trigger

A well-written hook prompt follows this pattern:

```
Invoke the [agent-name] to [task description specific to this trigger].
```

Examples:

```
"Invoke the docs-agent to review the saved file and update all Javadoc."

"Invoke the security-agent to scan the modified controller for injection vulnerabilities."

"Invoke the test-agent to generate unit tests for any new public methods in the saved file."
```

Each prompt names the agent first, then scopes the task to what is relevant for that particular hook trigger.

---

## Part 6 — Exercise

You have a custom agent at `.kiro/agents/review-agent.md` with `name: review-agent`. It performs code review on Java service classes.

Write the `then` block for a hook that invokes `review-agent` whenever a file is saved under `src/main/java/com/example/service/`:

```json
"then": {
  "type": "askAgent",
  "prompt": "..."
}
```

**Checklist before you save your hook:**

- [ ] Does the prompt name `review-agent` explicitly?
- [ ] Does the prompt describe what to review, not just that something changed?
- [ ] Is the agent file saved at `.kiro/agents/review-agent.md`?
- [ ] Does the agent's front matter contain `name: review-agent`?

If all four are true, your hook will correctly invoke your custom agent.

---

## Summary

| Concept | Key Point |
|---|---|
| Agent name | Defined in the front matter of `.kiro/agents/<name>.md` |
| Hook-to-agent link | Made through the `prompt` field — there is no separate `agent` field |
| Most common failure | Prompt describes a task but does not name the agent |
| Fix | Start the prompt with `Invoke the <agent-name> to...` |

The agent defines the rules. The hook defines the trigger. The prompt is the wire that connects them.