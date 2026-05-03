# Lab 8 — Kiro Agent Skills: Building Reusable Instruction Packages

**Day 2 | Layer 3 Extension | Duration: 60 minutes**
**Prerequisites:** Lab 5 (Custom Agents), Lab 6 (Hook Pipeline), Kiro IDE signed in

---

## Overview

In this lab you will build three **Agent Skills** for the Inventory Management Service project. Skills are portable instruction packages that Kiro activates on-demand — loaded only when your request matches the skill's description, not upfront. This keeps the agent's context focused while giving it access to deep, specialised knowledge exactly when it is needed.

By the end of this lab you will have:

- A workspace skill for **Spring Boot code review** (`.kiro/skills/spring-code-review/`)
- A workspace skill for **API documentation generation** (`.kiro/skills/api-doc-generator/`)
- A global skill for **personal commit message standards** (`~/.kiro/skills/commit-standards/`)
- Practised importing a community skill from GitHub
- Understood the distinction between Skills, Steering, and Hooks

---

## What Are Skills?

### The problem Skills solve

Every AI agent has a context window — a limit to how much information it can hold at once. The naive solution is to load everything upfront: all your architecture rules, all your team conventions, all your deployment procedures, all your code review checklists. But more context does not mean better results. Too much information overwhelms the agent, slows its responses, and reduces quality.

Skills solve this with **progressive disclosure**: give the agent a catalogue of what it knows, but only load the full details when those details are actually needed.

At startup, Kiro reads only the **name and description** of each skill — a few words, nothing more. When you ask for something that matches a skill's description, Kiro loads the full skill instructions at that moment. When the task is done, those instructions leave the context. The next task starts fresh.

### What a Skill actually is

A Skill is a folder on disk containing a required `SKILL.md` file and optional supporting files:

```
my-skill/
├── SKILL.md           ← required — instructions + frontmatter
├── scripts/           ← optional — executable scripts the agent can run
├── references/        ← optional — detailed docs the agent can read
└── assets/            ← optional — templates the agent can use
```

The `SKILL.md` file has two parts:

**Frontmatter** — metadata Kiro reads at startup (always loaded):

```markdown
---
name: spring-code-review
description: Review Spring Boot Java code for correctness and security.
             Use when reviewing a class or preparing for a PR.
---
```

**Body** — the actual instructions (loaded only when the skill activates):

```markdown
## Review Checklist

### 1. Architecture Alignment
- Are dependencies injected via constructor?
- Is business logic absent from controllers?
...
```

The `description` field is the critical piece. It is Kiro's **activation signal** — the agent compares your prompt against every skill's description and loads the full instructions only when there is a match. Write it like a trigger condition, not a label:

| Weak description | Strong description |
|---|---|
| "helps with code review" | "Review Spring Boot Java code for correctness, security, and architecture compliance. Use when reviewing a class, controller, or service before committing." |
| "for documentation" | "Generate OpenAPI annotations and Markdown endpoint summaries. Use when adding a new REST endpoint or preparing API docs for a release." |
| "commit messages" | "Write a Conventional Commits compliant commit message. Use when preparing a git commit or reviewing staged changes before committing." |

### How Skills activate — two modes

**Automatic activation** — Kiro matches your prompt text against skill descriptions and activates the best match. You do not have to name the skill:

> "Can you review the InventoryController before I merge?" → activates `spring-code-review`

**Manual activation via slash command** — Type `/` in the Kiro chat input to see all available skills as commands. Selecting one loads it immediately, regardless of your prompt text:

> `/spring-code-review Review InventoryController.java`

### Skill scope — workspace vs global

Skills can live in two places, and the location determines who can use them:

| Scope | Directory | Available in |
|---|---|---|
| **Workspace** | `.kiro/skills/<name>/` | This project only — committed to Git, shared with the team |
| **Global** | `~/.kiro/skills/<name>/` | Every project on your machine — personal, not committed |

If a workspace skill and a global skill share the same name, the **workspace skill wins**. This lets you define global defaults and override them per-project.

### Skills are portable

Skills follow the open [Agent Skills standard](https://agentskills.io/specification). This means:

- You can import skills from any public GitHub repository
- You can share your skills with other teams or the community
- Skills work across any AI tool that implements the same standard, not just Kiro

This is the key difference from Steering: steering files are Kiro-specific project context. Skills are portable, shareable, reusable packages.

### What Skills are not

Skills are not Hooks and they are not Agents. Understanding where each fits:

- A **Hook** fires automatically when an IDE event happens (file save, commit). You do not invoke it — it invokes itself.
- A **Custom Agent** is a persistent specialist with defined tool access (`read`, `write`, `shell`). You invoke it with `@agent-name`.
- A **Skill** is an instruction package you invoke deliberately (or Kiro activates by matching). It has no tool restrictions of its own — it runs within whatever context is active.

Think of it this way: an Agent is a person with a job title and a keycard. A Skill is a procedure manual that person picks up when they need it.

---

## Conceptual Foundation — Skills vs Steering vs Hooks

Before writing any files, understand where each mechanism fits.

| Mechanism | Lives in | Loaded | Best for |
|---|---|---|---|
| **Steering** | `.kiro/steering/` | Always (auto/manual/fileMatch) | Project-wide standards that always apply |
| **Skills** | `.kiro/skills/` or `~/.kiro/skills/` | On-demand when description matches | Reusable, portable workflows — share across projects or teams |
| **Hooks** | `.kiro/hooks/` | On IDE events (save, commit) | Automated background tasks triggered by developer actions |

**The key insight:** Steering is always present in the agent's context (consuming tokens). Skills are dormant until needed. If you have 20 team conventions, put the always-relevant ones in steering and the task-specific ones in skills — your agent stays fast and focused.

---

## Part 1 — Inspect the Skills Panel (5 minutes)

**Step 1.1** Open Kiro IDE with your Inventory Management Service project.

**Step 1.2** In the left sidebar, locate the **Kiro panel** and expand the **Agent Steering & Skills** section.

You will see two sub-sections:
- **Steering** — your existing `.kiro/steering/*.md` files from Lab 2
- **Skills** — empty for now

**Step 1.3** Open the integrated terminal (`` Ctrl+` `` on Windows/Linux, `` Cmd+` `` on Mac) and verify your project structure:

```bash
ls .kiro/
```

Expected output:
```
agents/   hooks/   specs/   steering/
```

The `skills/` directory does not exist yet — you will create it in Part 2.

---

## Part 2 — Workspace Skill: Spring Boot Code Review (20 minutes)

This skill will teach Kiro your team's specific code review checklist for Spring Boot services. It will activate automatically when you ask Kiro to review code, check a class, or prepare for a PR.

### Step 2.1 — Create the skill folder structure

```bash
mkdir -p .kiro/skills/spring-code-review/references
mkdir -p .kiro/skills/spring-code-review/assets
```

### Step 2.2 — Create the main SKILL.md

Create `.kiro/skills/spring-code-review/SKILL.md` with the following content:

```markdown
---
name: spring-code-review
description: Review Spring Boot Java code for correctness, security, test coverage, and alignment with project architecture. Use when reviewing a class, controller, service, repository, or any Java file before committing or raising a PR.
---

## Review Checklist

When activated, work through each section below in order.
Reference the files in `references/` for detailed standards.

### 1. Architecture Alignment
- Does this class belong to the correct layer (controller / service / repository)?
- Are dependencies injected via constructor (not field injection)?
- Is there any business logic inside a `@Controller` or `@RestController`? If yes, flag it.
- Does the class follow the package structure defined in `references/architecture-conventions.md`?

### 2. REST API Standards (Controllers only)
- Are HTTP status codes correct? (`201 Created` for POST, `204 No Content` for DELETE)
- Are all endpoints annotated with `@Operation` for OpenAPI documentation?
- Is request body validated with `@Valid` and `@NotNull` / `@NotBlank`?
- Are error responses returned as `ProblemDetail` (RFC 7807)?

### 3. Service Layer Standards
- Is `@Transactional` applied at the service level, not the repository level?
- Are checked exceptions wrapped in a domain-specific `RuntimeException`?
- Is there no direct use of `EntityManager`? (Use Spring Data repositories.)

### 4. Security
- Are any SQL queries constructed via string concatenation? (Flag immediately — SQL injection risk.)
- Is sensitive data (passwords, tokens) logged anywhere?
- Are endpoints appropriately secured (check `SecurityConfig`)?

### 5. Test Coverage
- Does each public method have a corresponding unit test?
- Are tests using `@ExtendWith(MockitoExtension.class)` (not Spring context)?
- Is there at least one integration test for each `@RestController`?

### 6. Output Format

After completing the review, produce a structured report:

    ## Code Review Report — [ClassName]

    ### ✅ Passed
    - List items that meet standards

    ### ⚠️ Warnings (non-blocking)
    - List items to improve before next sprint

    ### ❌ Blockers (must fix before PR merge)
    - List critical issues with line references

    ### Suggested next prompt
    "Fix the blockers in [ClassName] and re-run the spring-code-review skill."
```

> **Why this structure?** The `description` field is what Kiro reads at startup to decide whether to activate this skill. Make it specific — include the action words (review, check, prepare) and the subject (Spring Boot, Java, controller). Vague descriptions like "helps with code" will miss-trigger or never trigger.

### Step 2.3 — Create the architecture conventions reference

Create `.kiro/skills/spring-code-review/references/architecture-conventions.md`:

```markdown
# Architecture Conventions — Inventory Management Service

## Package Structure

    com.example.inventory
    ├── api/              # @RestController classes only
    │   └── dto/          # Request/response DTOs
    ├── domain/           # Business logic (@Service)
    │   └── model/        # Domain entities
    ├── infrastructure/   # @Repository, external clients
    └── config/           # @Configuration classes

## Naming Conventions
- Controllers: `[Resource]Controller.java` (e.g., `InventoryController.java`)
- Services: `[Resource]Service.java` (interface) + `[Resource]ServiceImpl.java`
- Repositories: `[Resource]Repository.java` (extends `JpaRepository`)
- DTOs: `[Resource]Request.java` / `[Resource]Response.java`

## Dependency Rules
- `api/` may depend on `domain/` — never on `infrastructure/`
- `domain/` must not depend on `api/` or `infrastructure/`
- `infrastructure/` may depend on `domain/`
```

### Step 2.4 — Verify the skill appears in the panel

Return to the Kiro panel → **Agent Steering & Skills**. The `spring-code-review` skill should now appear under Skills.

**Hover over it** — you will see the description text. This is exactly what Kiro reads at startup.

### Step 2.5 — Test the skill with a direct slash command

In the Kiro chat panel, type:

```
/spring-code-review
```

You should see `spring-code-review` appear as an autocomplete option. Select it, then add:

```
/spring-code-review Review InventoryController.java
```

Kiro will load the full `SKILL.md` instructions and perform the review against your file. Observe the structured report output.

**Checkpoint:** The report should contain all six sections. If any section is missing, check that your `SKILL.md` frontmatter is valid YAML (no tabs, correct indentation).

---

## Part 3 — Workspace Skill: API Documentation Generator (15 minutes)

This skill automates the generation of OpenAPI-compatible endpoint documentation for any new controller or method you write.

### Step 3.1 — Create the skill folder

```bash
mkdir -p .kiro/skills/api-doc-generator/assets
```

### Step 3.2 — Create SKILL.md

Create `.kiro/skills/api-doc-generator/SKILL.md`:

```markdown
---
name: api-doc-generator
description: Generate OpenAPI documentation annotations and Markdown endpoint summaries for Spring Boot REST controllers and methods. Use when adding a new endpoint, updating an API method signature, or preparing API documentation for a release.
---

## Documentation Generation Process

### Step 1 — Analyse the method signature
Identify:
- HTTP method and path (`@GetMapping`, `@PostMapping`, etc.)
- Path variables and request parameters
- Request body type (if any)
- Return type and possible HTTP status codes

### Step 2 — Add SpringDoc OpenAPI annotations

Add these annotations to the method:

    @Operation(
        summary = "[One-line description of what the endpoint does]",
        description = "[2-3 sentences. Include business context, not just technical details.]"
    )
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Success — [describe the response body]"),
        @ApiResponse(responseCode = "400", description = "Validation error — [describe when this occurs]"),
        @ApiResponse(responseCode = "404", description = "Not found — [describe when this occurs]"),
        @ApiResponse(responseCode = "500", description = "Internal server error")
    })

Use `@Parameter(description = "...")` for every `@PathVariable` and `@RequestParam`.

### Step 3 — Generate Markdown endpoint card

After annotating the code, produce a Markdown endpoint card in this format:

    ## [HTTP Method] [Path]

    **Summary:** [One-line summary]

    **Description:** [2-3 sentences from the @Operation description]

    ### Request
    | Parameter | Type | Required | Description |
    |---|---|---|---|
    | (list path variables, query params, body fields) |

    ### Response
    | Status | Description |
    |---|---|
    | 200 | [success description] |
    | 400 | [validation error] |
    | 404 | [not found] |

    ### Example Request
    (bash)
    curl -X [METHOD] "http://localhost:8080/[path]" \
      -H "Content-Type: application/json" \
      -d '[example JSON body if applicable]'

    ### Example Response
    (json)
    [example response JSON]

Append this card to `docs/api-reference.md`. If the file does not exist, create it with a heading:
`# Inventory Management Service — API Reference`
```

### Step 3.3 — Test the skill

In Kiro chat, type:

```
/api-doc-generator Document the POST /api/inventory endpoint in InventoryController.java
```

Verify that Kiro:
1. Adds `@Operation` and `@ApiResponses` annotations to the correct method
2. Creates or appends to `docs/api-reference.md`
3. Produces a properly formatted Markdown endpoint card

---

## Part 4 — Global Skill: Personal Commit Message Standards (10 minutes)

A global skill lives in `~/.kiro/skills/` and is available in **every** workspace on your machine. This is the right place for personal, cross-project workflows — things you always want, regardless of the project.

### Step 4.1 — Create the global skill

```bash
mkdir -p ~/.kiro/skills/commit-standards
```

Create `~/.kiro/skills/commit-standards/SKILL.md`:

```markdown
---
name: commit-standards
description: Write a Conventional Commits compliant commit message for staged changes. Use when preparing a git commit, writing a commit message, or reviewing staged changes before committing.
---

## Commit Message Process

### Step 1 — Analyse staged changes
Run a mental diff of the staged files. Identify:
- The primary change type
- The scope (which module or feature is affected)
- Whether this is a breaking change

### Step 2 — Select the correct type

| Type | Use when |
|---|---|
| `feat` | A new feature visible to the user or downstream consumers |
| `fix` | A bug fix |
| `refactor` | Code restructure with no behaviour change |
| `test` | Adding or updating tests only |
| `docs` | Documentation only |
| `chore` | Build scripts, CI config, dependency updates |
| `perf` | Performance improvement |

### Step 3 — Write the commit message

Follow this format exactly:

    <type>(<scope>): <imperative summary under 72 chars>

    [Optional body — explain WHY, not WHAT. Wrap at 72 chars.]

    [Optional footer]
    BREAKING CHANGE: <description if applicable>
    Refs: #<issue-number if applicable>

### Examples

    feat(inventory): add low-stock threshold alert endpoint

    Adds GET /api/inventory/alerts that returns all items where
    current stock falls below the configured minimum threshold.
    Threshold is configurable per item via the existing PATCH endpoint.

    Refs: #42

    fix(reservation): prevent double-booking on concurrent requests

    Adds optimistic locking (@Version) to the Reservation entity to
    reject concurrent save operations on the same record.

    Refs: #67

### Step 4 — Output

Produce the complete commit message inside a code block so it can be copied directly.
Then ask: "Would you like me to run `git commit -m` with this message?"
```

### Step 4.2 — Verify global scope

In Kiro, open any **different** project folder. Type `/commit-standards` in chat. The skill should be available even though that project has no `.kiro/skills/` folder.

This demonstrates the global vs workspace scope distinction.

---

## Part 5 — Import a Community Skill from GitHub (5 minutes)

Kiro supports importing skills directly from public GitHub repositories.

### Step 5.1 — Import via the Kiro panel

1. In the Kiro panel → **Agent Steering & Skills**, click the **+** button
2. Select **Import a skill**
3. Choose **GitHub**
4. Paste the following URL (this points to a skill subfolder, not the repo root — required by Kiro):

```
https://github.com/kirodotdev/Kiro/tree/main/skills/pr-review
```

> If this URL is not available, use any public GitHub repository that follows the Agent Skills standard (a folder containing a `SKILL.md`). The format is `https://github.com/<owner>/<repo>/tree/<branch>/<skill-folder>`.

5. Click **Import**

The skill is copied into `.kiro/skills/` (workspace) or `~/.kiro/skills/` (global) depending on your selection.

### Step 5.2 — Inspect the imported skill

Open the imported `SKILL.md` and note:
- The frontmatter structure
- How the description is written (action verbs + subject + context)
- Whether it uses `references/` or `scripts/` subdirectories

This is a model for how to write your own shareable skills.

---

## Part 6 — Skills vs Steering Decision Exercise (5 minutes)

Look at the following list of team conventions. Decide whether each belongs in **Steering** or a **Skill**:

| Convention | Your decision | Reasoning |
|---|---|---|
| "All repositories must extend `JpaRepository`" | ? | |
| "When generating an API doc, use this 6-step process" | ? | |
| "Package structure: `api/`, `domain/`, `infrastructure/`" | ? | |
| "Before any commit, validate the message format" | ? | |
| "All HTTP error responses use `ProblemDetail` (RFC 7807)" | ? | |

**Suggested answers:**

| Convention | Answer | Reasoning |
|---|---|---|
| All repos extend JpaRepository | **Steering** | Always relevant — agent should know this on every generation task |
| API doc 6-step process | **Skill** | Only relevant when documenting — no need to load upfront |
| Package structure | **Steering** | Always relevant — agent should enforce this on every file it creates |
| Commit message format | **Skill** (global) | Task-specific, used across all projects, not needed during coding |
| ProblemDetail error format | **Steering** | Always relevant for any controller or exception handler work |

**Rule of thumb:** If the convention affects every code generation task → Steering. If it's a specific workflow you invoke deliberately → Skill.

---

## Lab Validation Checklist

Before moving on, verify each item:

- [ ] `.kiro/skills/spring-code-review/SKILL.md` exists with valid frontmatter
- [ ] `.kiro/skills/spring-code-review/references/architecture-conventions.md` exists
- [ ] `/spring-code-review` slash command is available in Kiro chat
- [ ] Skill produced a structured review report with all six sections
- [ ] `.kiro/skills/api-doc-generator/SKILL.md` exists
- [ ] `/api-doc-generator` slash command is available
- [ ] Skill generated `docs/api-reference.md` with an endpoint card
- [ ] `~/.kiro/skills/commit-standards/SKILL.md` exists
- [ ] `commit-standards` is available in a different project workspace
- [ ] Community skill successfully imported from GitHub
- [ ] Steering vs Skills decision exercise completed

---

## Key Concepts Summary

| Concept | Detail |
|---|---|
| **Skill location (workspace)** | `.kiro/skills/<skill-name>/SKILL.md` |
| **Skill location (global)** | `~/.kiro/skills/<skill-name>/SKILL.md` |
| **Name constraint** | Lowercase letters, numbers, hyphens only — max 64 characters |
| **Description constraint** | Max 1024 characters — this is Kiro's activation signal |
| **Slash command** | Type `/` in chat to see all available skills as commands |
| **Priority rule** | Workspace skill overrides global skill if names conflict |
| **Portability** | Skills follow the open [Agent Skills standard](https://agentskills.io/specification) — importable from GitHub or any compatible AI tool |
| **On-demand loading** | Kiro loads only name + description at startup; full instructions load only on activation |

---

## Troubleshooting

**Skill does not appear in the Kiro panel**
- Check that the folder name matches the `name` field in frontmatter exactly
- Check that the `SKILL.md` frontmatter uses valid YAML (use spaces, not tabs)
- Reload the Kiro panel using the refresh icon

**Slash command not showing**
- Ensure the skill folder is inside `.kiro/skills/` (not `.kiro/steering/`)
- Restart Kiro if you created the file while Kiro was already open

**Skill activates for the wrong prompts**
- Rewrite the `description` to be more specific — add the exact action verbs and subjects that should trigger it
- Remove generic words like "helps with" and replace with concrete triggers: "Use when reviewing", "Use when generating", "Use when preparing"

**Imported skill not found**
- The GitHub URL must point to a **subfolder** containing `SKILL.md`, not the repository root
- The repository must be public

---



---

*Lab 8 | Day 2 | Kiro Agent Skills | Last updated: May 2026*