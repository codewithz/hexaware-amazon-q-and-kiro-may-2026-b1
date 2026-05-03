# Kiro Hooks for Enterprise: Governing New Code Without Touching the Existing Codebase

**Presentation Plan of Action**
**Context:** Large Enterprise Spring Boot Application — Already in Production

---

## Executive Summary

> **Problem:** A large enterprise Java/Spring Boot application is in production with thousands of existing methods, tests, and documentation. The team wants to adopt AI-assisted test and documentation generation using Kiro Hooks — but cannot risk touching, overwriting, or regressing anything already in production.
>
> **Solution:** Implement a **Baseline-First Kiro Hook Strategy** — where the existing codebase is snapshotted as a protected baseline, and Kiro Hooks are configured to govern only net-new code added from this point forward.

---

## The Core Principle

```
Everything written BEFORE today → Protected. Kiro does not touch it.
Everything written AFTER today  → Governed. Kiro Hooks apply automatically.
```

This is not a limitation — it is a **deliberate governance decision** that makes Kiro adoption safe, incremental, and reversible.

---

## Part 1 — Understanding Kiro Hooks (Quick Recap)

Kiro Hooks are event-driven AI agents that activate when a file is created or modified. They live inside the project repository under:

```
.kiro/
├── hooks/          ← Hook definitions (what the agent does)
├── steering/       ← Shared rules and context (how the agent behaves)
└── specs/          ← Feature specifications (what is being built)
```

Each hook is a Markdown file with:
- A **trigger** — which file or file pattern activates it
- **Instructions** — what the AI agent should do when triggered
- **Constraints** — hard boundaries the agent must never cross

---

## Part 2 — The Enterprise Challenge

When adopting Kiro Hooks in an existing production system, three risks must be mitigated:

| Risk | Description |
|------|-------------|
| **Regression Risk** | AI regenerates or modifies tests for existing methods, breaking coverage contracts |
| **Overwrite Risk** | AI rewrites existing Javadoc or documentation, losing human-authored context |
| **Scope Creep Risk** | Hook triggers on files it should not, causing unintended changes across the codebase |

All three risks are eliminated by the **Baseline Protection Strategy** described below.

---

## Part 3 — The Baseline Protection Strategy

### Step 1: Freeze the Existing Codebase as a Baseline

Before any hook is activated, a one-time **baseline snapshot** is created. This is a governed Markdown file committed to the repository that lists all existing classes and their methods as of the cutover date.

**File:** `.kiro/steering/baseline-registry.md`

```markdown
# Enterprise Baseline Registry
# Cutover Date: [DATE]
# Approved By: Architecture Guild
# Change Process: PR with two senior engineer approvals required

## IMPORTANT
Methods listed below are GRANDFATHERED.
No Kiro Hook — test generation, documentation, or review —
may create, modify, or delete anything associated with these methods.

---

## com.example.service.OrderService

| Method Signature | Status | Notes |
|-----------------|--------|-------|
| createOrder(OrderRequest) | Protected | Core billing path |
| cancelOrder(String orderId) | Protected | Saga participant |
| getOrderStatus(String orderId) | Protected | High-traffic read path |
| ... (all existing methods) | Protected | |

## com.example.service.InventoryService

| Method Signature | Status | Notes |
|-----------------|--------|-------|
| reserveStock(String sku, int qty) | Protected | |
| releaseStock(String sku, int qty) | Protected | |
| ... | Protected | |

## [Repeat for every existing class]
```

> **This file is the single source of truth.** It is committed to Git on Day 1 and only changes via a Pull Request with explicit approvals.

---

### Step 2: Define the Hook with Baseline Awareness

Every Kiro Hook in the enterprise setup follows a **Read-Filter-Act** pattern:

```
1. READ    → What changed in this file?
2. FILTER  → Is this method/class in the baseline registry? If yes, STOP.
3. ACT     → Only proceed for methods NOT in the baseline.
```

---

## Part 4 — Hook Definitions

### Hook 1: Automated Test Generation

**File:** `.kiro/hooks/test-generator.md`

```markdown
# Hook: Enterprise Test Generator

## Trigger
- On save of any `.java` file under `src/main/java/`

## Pre-Flight Checks (MUST complete before any action)
1. Read `.kiro/steering/enterprise-policy.md`
2. Read `.kiro/steering/baseline-registry.md`
3. Read `.kiro/steering/security-exclusions.md`

## Instructions

### Step 1 — Identify Changed Methods
- Extract all method signatures from the saved Java file
- Compare every signature against the baseline-registry.md

### Step 2 — Apply Baseline Filter
- If a method signature matches any entry in baseline-registry.md → SKIP IT COMPLETELY
- Do not reference it, do not read its existing tests, do not touch its test file sections
- Proceed ONLY with methods that are NOT listed in the baseline

### Step 3 — Spec Check
- For each new (non-baseline) method, check if a spec exists in `.kiro/specs/`
- If a spec exists → generate tests that validate the spec's acceptance criteria
- If no spec exists → output a warning to the developer:
  "⚠️ No spec found for [MethodName]. Please create a spec in .kiro/specs/ before tests can be generated."
- Do NOT generate tests without a corresponding spec

### Step 4 — Generate Tests
- Locate the corresponding test file (e.g., `OrderServiceTest.java`)
- Check if a `@Test` method already exists for this new method → if yes, skip
- If no test exists → append a new JUnit 5 test block at the END of the test file
- Never insert tests between existing test methods
- Never modify any existing `@Test` method

### Step 5 — Tag and Label
- Every AI-generated test must include:
  - `@Tag("ai-generated")` annotation
  - A comment header: `// AI-Generated | Hook: test-generator | Review Required`

## Hard Constraints
- NEVER modify any test annotated with `@Tag("human-reviewed")`
- NEVER delete any existing test method
- NEVER regenerate tests for baseline methods under any circumstance
- NEVER generate tests for methods listed in security-exclusions.md
- If uncertain about a method's scope → SKIP and notify, never guess
```

---

### Hook 2: Automated Documentation Generation

**File:** `.kiro/hooks/doc-generator.md`

```markdown
# Hook: Enterprise Documentation Generator

## Trigger
- On save of any `.java` file under `src/main/java/`

## Pre-Flight Checks
1. Read `.kiro/steering/enterprise-policy.md`
2. Read `.kiro/steering/baseline-registry.md`

## Instructions

### Step 1 — Identify New or Undocumented Methods
- Extract all method signatures from the saved file
- Filter out ALL methods present in baseline-registry.md
- From remaining methods, identify those with missing or empty Javadoc

### Step 2 — Generate Javadoc
For each new method without existing Javadoc:
- Generate a complete Javadoc block including:
  - `@param` for every parameter
  - `@return` description
  - `@throws` for declared exceptions
  - One-line summary sentence
- Insert the Javadoc block immediately above the method declaration
- Never modify existing Javadoc blocks

### Step 3 — Update Module README (if applicable)
- If the new method represents a significant new capability, append a one-line entry
  to the module's `README.md` under a `## Recent Additions` section
- Never modify existing README content above this section

## Hard Constraints
- NEVER modify Javadoc for methods in baseline-registry.md
- NEVER overwrite existing Javadoc — only add where it is missing
- NEVER remove `@deprecated` or `@since` tags from any method
```

---

### Hook 3: Code Review Agent

**File:** `.kiro/hooks/code-review.md`

```markdown
# Hook: Enterprise Code Review Agent

## Trigger
- On save of any `.java` file under `src/main/java/`

## Pre-Flight Checks
1. Read `.kiro/steering/enterprise-policy.md`
2. Read `.kiro/steering/baseline-registry.md`
3. Read `.kiro/steering/coding-standards.md`

## Instructions

### Step 1 — Scope to New Code Only
- Identify methods NOT present in baseline-registry.md
- Review ONLY those methods — ignore everything else in the file

### Step 2 — Review Against Standards
For each new method, check:
- Naming conventions (camelCase, meaningful names)
- Exception handling (no swallowed exceptions, no bare catch blocks)
- Null safety (appropriate null checks or Optional usage)
- Single Responsibility (method does one thing)
- Logging (appropriate log levels, no sensitive data in logs)
- Security patterns (no hardcoded credentials, input validation present)

### Step 3 — Output Findings
- Output findings as inline comments in the format:
  `// [REVIEW] <finding> | Severity: LOW | MEDIUM | HIGH`
- Do not modify the code itself — only add review comments
- Group findings at the TOP of the new method block

## Hard Constraints
- NEVER add review comments to baseline methods
- NEVER modify any logic — comments only
- NEVER flag baseline code as needing changes
```

---

## Part 5 — Governance Steering Files

### Enterprise Policy

**File:** `.kiro/steering/enterprise-policy.md`

```markdown
# Enterprise Kiro Hook Policy

## Ownership & Approvals
| Artifact | Owner | Change Process |
|----------|-------|----------------|
| baseline-registry.md | Architecture Guild | PR + 2 senior approvals |
| Hook definitions (.kiro/hooks/) | Tech Lead | PR + 1 senior approval |
| Steering files (.kiro/steering/) | Architecture Guild | PR + 2 senior approvals |
| Spec files (.kiro/specs/) | Feature Team | PR + 1 team lead approval |

## AI-Generated Content Policy
- All AI-generated tests start as `@Tag("ai-generated")`
- AI-generated tests must be reviewed and approved within 5 business days
- Approved tests are upgraded to `@Tag("human-reviewed")` by the reviewer
- AI-generated content that is not reviewed within 5 days is flagged in the next sprint review

## Baseline Evolution Policy
- A method is removed from the baseline (graduates) when:
  1. It receives a new AI-generated test that has been human-reviewed
  2. A tech lead explicitly removes it from baseline-registry.md via PR
- Methods are ADDED to the baseline only on the initial cutover date
- No new methods are added to the baseline after cutover — all new methods are governed

## Scope Boundaries
- Hooks apply to: `src/main/java/`
- Hooks exclude: `src/main/java/legacy/`, `generated/`, `deprecated/`
- Security-sensitive classes: see `security-exclusions.md`
```

---

### Security Exclusions

**File:** `.kiro/steering/security-exclusions.md`

```markdown
# Security Exclusion Registry
# These classes and methods are excluded from ALL Kiro Hook activity.
# Changes to this file require Security Team + Architecture Guild approval.

## Excluded Classes (No Hook Activity Permitted)
- com.example.security.*
- com.example.auth.*
- com.example.crypto.*
- com.example.payment.*

## Reason
These classes contain security-sensitive logic. AI-generated tests or documentation
for these classes present an unacceptable risk of exposing implementation details
or creating tests that could be used to probe security boundaries.

## Process for Security Class Coverage
All test and documentation work for excluded classes must be performed manually
by engineers with appropriate security clearance, following the Security Testing SOP.
```

---

## Part 6 — Repository Structure

The complete `.kiro/` directory layout for the enterprise setup:

```
.kiro/
├── hooks/
│   ├── test-generator.md         ← Generates JUnit 5 tests for new methods
│   ├── doc-generator.md          ← Generates Javadoc for new methods
│   └── code-review.md            ← Reviews new methods against standards
│
├── steering/
│   ├── enterprise-policy.md      ← Ownership, approvals, AI content policy
│   ├── baseline-registry.md      ← THE protected snapshot (ALL existing methods)
│   ├── security-exclusions.md    ← Classes excluded from all hook activity
│   └── coding-standards.md       ← Java/Spring Boot standards the review hook uses
│
└── specs/
    └── [feature-name]/
        ├── requirements.md        ← EARS-format user stories
        ├── design.md              ← Technical design
        └── tasks.md               ← Implementation checklist
```

---

## Part 7 — Day 1 Onboarding Workflow

The following steps are executed ONCE to activate the strategy:

```
Day 1 Onboarding Checklist
──────────────────────────

□ Step 1: Run baseline capture script
          → Scans all src/main/java/ files
          → Extracts every class + method signature
          → Generates baseline-registry.md

□ Step 2: Architecture Guild reviews baseline-registry.md
          → Confirms completeness
          → Adds ownership annotations for sensitive methods

□ Step 3: Commit baseline-registry.md to main branch via PR
          → Requires 2 Architecture Guild approvals
          → This is a one-time, irreversible commitment

□ Step 4: Commit all .kiro/ hook and steering files via PR
          → Tech Lead approval required

□ Step 5: Team walkthrough (30 min)
          → How hooks trigger
          → How to read AI-generated test output
          → How to promote @Tag("ai-generated") to @Tag("human-reviewed")
          → How to create a spec before adding a new method

□ Step 6: Pilot with one feature team for 2 weeks
          → Monitor hook output quality
          → Gather feedback on false positives (hook touching baseline)
          → Tune instructions if needed

□ Step 7: Roll out to all feature teams
```

---

## Part 8 — Lifecycle of a New Method

Here is the end-to-end journey of a net-new method from this point forward:

```
Developer adds Method 21 to OrderService.java
                │
                ▼
        Developer saves the file
                │
                ▼
    Kiro Hook: test-generator fires
                │
        Check baseline-registry.md
                │
     Method 21 NOT in baseline ✓
                │
        Does a spec exist?
         /              \
        YES              NO
         │                │
         ▼                ▼
   Generate test      Notify developer:
   from spec AC's     "Create spec first"
         │
         ▼
   Append to OrderServiceTest.java
   Tagged: @Tag("ai-generated")
         │
         ▼
   Developer reviews the test
         │
         ▼
   Developer approves → changes tag to
   @Tag("human-reviewed")
         │
         ▼
   Tech Lead optionally adds Method 21
   to baseline-registry.md via PR
   (only when team agrees it is stable)
```

---

## Part 9 — Answering the Hard Questions

**Q: What if a developer modifies an existing (baseline) method?**

> The hook reads the diff, detects that the modified method is in the baseline, and skips it entirely. No tests are generated or modified. The developer is responsible for updating existing tests manually — just as they do today.

**Q: What if the baseline-registry.md is incomplete on Day 1?**

> Any method NOT in the registry is treated as "new" and falls under hook governance. This is why the baseline review on Day 1 is critical. However, if a missed method is discovered, it can be added to the registry via the standard PR process before any hook output is committed.

**Q: Can a developer opt out for a specific file?**

> Yes. The enterprise policy can include an opt-out comment convention:
> `// kiro:skip` at the top of a class tells all hooks to ignore that file entirely.

**Q: What about test files themselves — can the hook accidentally modify them?**

> Hook triggers are scoped to `src/main/java/` only. Test files in `src/test/java/` are not triggers. The hook only *writes* to test files; it never reads a test file as a trigger.

**Q: How does the baseline evolve over time?**

> It does not grow — it only shrinks (graduates). Methods leave the baseline when they receive a human-reviewed AI test. No new methods are ever added to the baseline after Day 1 cutover.

---

## Part 10 — Summary

| Dimension | Approach |
|-----------|----------|
| **Protection mechanism** | Baseline registry committed to Git on Day 1 |
| **Hook scope** | Net-new methods only — baseline methods are invisible to hooks |
| **Spec requirement** | Tests are only generated when a corresponding spec exists |
| **AI content tagging** | All AI output tagged `@Tag("ai-generated")` until human-reviewed |
| **Security** | Sensitive classes fully excluded from all hook activity |
| **Governance** | All registry and hook changes go through PR with named approvals |
| **Rollout** | Pilot with one team → validate → expand |
| **Reversibility** | Hooks can be disabled per file with `// kiro:skip` |

---

> **The bottom line:** Kiro Hooks give the team forward-looking AI governance without any risk to the existing production codebase. The baseline registry is the safety contract. Everything else is automation.