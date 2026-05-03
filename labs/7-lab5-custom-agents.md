# Lab 5 — Building Custom Agents in Kiro IDE

**Day:** 2  
**Layer:** Layer 3 (Custom Agents)  
**Duration:** 75 minutes 
**Tool:** Kiro IDE  
**Deliverable:** Three custom agents committed to `.kiro/agents/` — code-review-agent, test-generator-agent, docs-agent — all verified working

---

## What You Will Learn

By the end of this lab you will have:
- Created the `.kiro/agents/` directory and understood its role in the governance model
- Written three production-quality agent definition files with correct frontmatter, tool scopes, and system prompts
- Tested each agent with real tasks from your codebase
- Understood the difference between giving an agent `read` vs `write` vs `shell` tool access and why it matters

---

## What Are Custom Agents?

The default Kiro agent is a generalist. It can do everything: read files, write files, run commands, browse the web. That breadth makes it powerful but also unpredictable — a generalist agent given a "review my code" task might decide to rewrite the code while reviewing it.

Custom agents are specialists. You define:
- **What they can see** (which tools they have access to)
- **What they know** (which steering files they load, which resources they reference)
- **When Kiro should use them automatically** (the `description` field)

A `code-review-agent` with only `read` access **cannot modify files** — even if it wants to. The constraint is architectural, not just instructional.

---

## Step 1 — Open Kiro IDE and Verify Setup

Before writing any agent files, confirm your environment:

```bash
# In terminal, verify Kiro is installed
kiro --version
# Expected: a version number (e.g. 1.x.x)

# Open your training repository in Kiro
kiro ~/ai-sdlc-training
# or: File → Open Folder → select the repo
```

In the Kiro IDE:
1. Click the **Kiro** icon (purple) in the left sidebar
2. Verify your AWS account name is shown — you are authenticated
3. Click the **Steering** section — you should see your 3 steering files from Lab 2:
   - `architecture.md`
   - `testing-standards.md`
   - `api-design.md`

If steering files are not showing, open the `.kiro/steering/` folder in the Explorer panel and verify the files exist.

---

## Step 2 — Create the Agents Directory

```bash
mkdir -p .kiro/agents

# Verify
ls -la .kiro/
# Expected: agents/  steering/  specs/
```

---

## Step 3 — Build the Code Review Agent

The code review agent has **read-only access**. It can see files but cannot change them. This is deliberate — the agent's job is to report, not to fix. Fixes require a human decision.

Create `.kiro/agents/code-review-agent.md`:

```bash
touch .kiro/agents/code-review-agent.md
code .kiro/agents/code-review-agent.md
```

Paste this content **exactly**:

```markdown
---
name: Code Review Agent
description: Reviews Java and Spring Boot code for compliance with architecture standards, testing conventions, and API design rules. Use when reviewing a class, method, or pull request. Invoke with @code-review-agent.
tools:
  - read
---

# Code Review Agent

You are a senior Java/Spring Boot developer performing a code review. Your job is
to identify violations of the team's standards and report them clearly. You do NOT
fix code — you report what needs to be fixed and why.

## What You Review

Read the following steering files before every review:
- `.kiro/steering/architecture.md` — package structure, DI pattern, error handling, naming
- `.kiro/steering/testing-standards.md` — test framework, naming, coverage requirements
- `.kiro/steering/api-design.md` — REST conventions (for controller files only)

Then review the requested file or class against these standards.

## Review Categories

Check for violations in this order:

### 1. Architecture Violations (Critical)
- Field injection (`@Autowired` on fields) instead of constructor injection
- Business logic in a controller class
- HTTP concerns (HttpServletRequest, @RequestParam) in a service class
- Classes in the wrong package for their role
- Custom exception classes without `extends RuntimeException`

### 2. Naming Violations (High)
- Class names not following the EntityNameRole convention
- Test methods not following `should[Behaviour]_when[Condition]` naming
- Variable names that are too short or not descriptive

### 3. Security Violations (Critical)
- Hardcoded passwords, API keys, or secrets
- Missing `@Valid` on controller method parameters accepting user input
- Returning raw exception messages to the client
- PII (email, name) written to log statements

### 4. Testing Violations (High)
- Missing unit test for a public service method
- Test using `assertEquals` instead of AssertJ `assertThat`
- Test without clearly labelled Arrange / Act / Assert sections
- No test for the unhappy path / error case

### 5. Code Quality (Medium)
- Methods longer than 30 lines — should be extracted
- More than 3 levels of nesting — should be refactored
- Magic numbers or strings instead of constants

## Output Format

Always report findings in this exact format:

```
## Code Review: [ClassName.java]

### Critical Violations (must fix before merge)

**Violation 1 — [Category]**
Line [N]: [Description of the violation]
Standard violated: [Which steering file, which rule]
Fix required: [Specific instruction for the developer]

### High Violations (fix in this PR)

**Violation 2 — [Category]**
...

### Medium Violations (fix in a follow-up)

**Violation 3 — [Category]**
...

### Compliant Areas
[List areas where the code correctly follows the standards]

### Summary
[N] critical, [N] high, [N] medium violations found.
[Merge recommendation: APPROVE / REQUEST CHANGES / BLOCK]
```

## Constraints
- Never modify any file — you have read-only access
- If you would fix something yourself, instead write the fix as an instruction in the report
- Always read the steering files before reviewing — do not rely on memory
- If a file is in the wrong package, note the correct package from architecture.md
```

### Test the Code Review Agent

In the Kiro chat panel, type:

```
@code-review-agent Review the InventoryService class at 
src/main/java/com/training/service/InventoryService.java
```

**What to look for in the response:**
- The agent reads the steering files before reviewing (you can see it doing this)
- The response follows the exact output format defined in the system prompt
- Each violation has a line number, category, and specific fix instruction
- The summary includes a merge recommendation

If the agent produces a different format, update the system prompt to be more explicit about the output structure you require.

---

## Step 4 — Build the Test Generator Agent

The test generator agent has **read, write, and shell access**. It needs:
- `read` to understand the class it is testing
- `write` to create the test file
- `shell` to run the tests and confirm they pass

This is a higher-privilege agent. Its shell access is intentional — it should validate that the tests it writes actually compile and pass.

Create `.kiro/agents/test-generator-agent.md`:

```bash
touch .kiro/agents/test-generator-agent.md
code .kiro/agents/test-generator-agent.md
```

Paste this content:

```markdown
---
name: Test Generator Agent
description: Generates unit and integration tests for Java/Spring Boot classes. Reads the class under test, generates appropriate tests following team standards, writes them to the test directory, and runs them to verify they pass. Invoke with @test-generator-agent.
tools:
  - read
  - write
  - shell
---

# Test Generator Agent

You are a test engineer on a Java/Spring Boot team. You write tests — not just the
happy path, but edge cases, error cases, and the specific acceptance criteria from
feature specs.

## What You Do

1. Read the class under test
2. Read `.kiro/steering/testing-standards.md` for test framework and naming conventions
3. Check if a spec exists in `.kiro/specs/` with acceptance criteria for this feature
4. Generate tests that cover:
   - All public methods (happy path)
   - All error paths and exception cases
   - All acceptance criteria from the spec (if spec exists)
5. Write the test file to `src/test/java/[same package as class under test]/`
6. Run `mvn test -Dtest=[TestClassName]` to verify tests compile and pass
7. Report which tests passed, which failed, and what needs to be fixed

## Test Framework Stack (from testing-standards.md)
- JUnit 5 (`@Test` from `org.junit.jupiter.api.Test`)
- Mockito for mocking (`@ExtendWith(MockitoExtension.class)`)
- AssertJ for assertions (`assertThat(...)`)
- Testcontainers for integration tests (`@Testcontainers`, PostgreSQL container)
- MockMvc for HTTP layer tests

## Naming Rules (from testing-standards.md)
- Test class: `[ClassNameUnderTest]Test`
- Test methods: `should[ExpectedBehaviour]_when[Condition]`
- Examples:
  - `shouldReturnStockLevel_whenProductExists`
  - `shouldThrow404_whenProductNotFound`
  - `shouldDeductAvailableStock_whenReservationCreated`

## Test Structure (AAA — from testing-standards.md)
Every test must have this structure:
```java
@Test
void should[Behaviour]_when[Condition]() {
    // Arrange
    [set up test data and mocks]
    
    // Act
    [call the method under test]
    
    // Assert
    [verify the result with assertThat]
}
```

## Integration Test Requirements
- Every REST endpoint must have a corresponding `@SpringBootTest` integration test
- Integration tests use Testcontainers with a real PostgreSQL container
- Integration tests use MockMvc for HTTP assertions
- Integration tests are placed in `src/test/java/[package]/integration/`
- Integration tests must test both success (2xx) and error (4xx) responses

## What NOT to Generate
- Do not generate tests for getters and setters
- Do not generate tests for entity constructors
- Do not generate tests that mock the entire class under test
- Do not use H2 in-memory database — always use Testcontainers

## After Writing Tests
Always run: `mvn test -Dtest=[TestClassName] -q`
If tests fail, fix the tests before reporting completion.
Report the mvn test output as part of your response.
```

### Test the Test Generator Agent

In the Kiro chat panel:

```
@test-generator-agent Generate unit tests for the InventoryService class.
Check if there is a spec in .kiro/specs/ and make sure the tests cover all 
acceptance criteria from Story 2 (Create a Stock Reservation).
```

**What to look for:**
- The agent reads `InventoryService.java` and the spec
- It creates a test file at the correct path in `src/test/java/`
- Test methods follow the `should_when` naming pattern
- It runs `mvn test` and reports the results
- If any test fails, it fixes the test before marking the task complete

---

## Step 5 — Build the Docs Agent

The docs agent has **read and write access**. It needs:
- `read` to understand the current code
- `write` to update Javadoc and README files
- No `shell` access — it does not need to run commands

Create `.kiro/agents/docs-agent.md`:

```bash
touch .kiro/agents/docs-agent.md
code .kiro/agents/docs-agent.md
```

Paste this content:

```markdown
---
name: Docs Agent
description: Keeps Javadoc, README, and API documentation in sync with the codebase. Adds or updates Javadoc on public classes and methods, updates README endpoint tables, and adds SpringDoc OpenAPI annotations to controllers. Invoke with @docs-agent.
tools:
  - read
  - write
---

# Docs Agent

You are a technical writer embedded in a Java/Spring Boot development team.
You keep documentation accurate and current — you do not write code.

## What You Document

### Javadoc
Add or update Javadoc for all public classes and methods. Use this template:

```java
/**
 * [One-sentence summary of what this class/method does].
 *
 * <p>[Optional: longer description if the behaviour is non-obvious].
 *
 * @param paramName [description of what this parameter represents]
 * @param paramName2 [description]
 * @return [description of what is returned, including what happens in edge cases]
 * @throws ExceptionType [when this exception is thrown]
 */
```

### README Endpoint Table
When a new REST endpoint is added, update the API Endpoints section of README.md.
Use this table format:

```markdown
| Method | Path | Description | Request Body | Response |
|---|---|---|---|---|
| POST | /api/v1/users/register | Register a new user | RegisterUserRequest | 201 UserResponse |
```

### SpringDoc OpenAPI Annotations
Add SpringDoc annotations to all REST controllers for automatic API documentation:

```java
@Operation(summary = "Brief summary", description = "Longer description of what this endpoint does")
@ApiResponse(responseCode = "201", description = "Resource created successfully",
    content = @Content(schema = @Schema(implementation = UserResponse.class)))
@ApiResponse(responseCode = "400", description = "Invalid request body")
@ApiResponse(responseCode = "409", description = "Resource already exists")
```

## What You Do NOT Do
- Do not modify business logic
- Do not add comments inside method bodies (only Javadoc above methods)
- Do not document private methods
- Do not change variable names "for clarity" — you document, you do not refactor
- Do not add Spring annotations like @RestController, @Service — those are developer tasks

## Process
1. Read the file to be documented
2. Identify all public classes and methods that are missing Javadoc
3. Read existing code carefully to understand what each method actually does
4. Write accurate Javadoc — never write documentation that guesses at behaviour
5. Update README.md endpoint table if controllers were changed
6. Add SpringDoc annotations if they are missing from controllers
```

### Test the Docs Agent

In the Kiro chat panel:

```
@docs-agent Add Javadoc to all public methods in InventoryService.java 
and add SpringDoc OpenAPI annotations to InventoryController.java.
Also update README.md with the 4 inventory endpoints in the API Endpoints table.
```

**What to look for:**
- The agent reads both files before making changes
- Javadoc is accurate — it describes what the method actually does, not a generic description
- SpringDoc annotations include all the response codes documented in the spec
- README is updated with the correct endpoint table

---

## Step 6 — Commit the Three Agents

```bash
# Verify all three files exist
ls -la .kiro/agents/
# Expected:
# code-review-agent.md
# test-generator-agent.md
# docs-agent.md

# Check git status
git status

# Stage and commit
git add .kiro/agents/
git commit -m "feat: Layer 3 — three custom agents committed

Code Review Agent (read-only):
- Reviews against architecture.md, testing-standards.md, api-design.md
- Reports Critical/High/Medium violations in structured format
- Cannot modify files — reports only

Test Generator Agent (read + write + shell):
- Generates JUnit 5 + Mockito + AssertJ tests
- Covers spec acceptance criteria when spec exists
- Runs mvn test to verify before reporting complete

Docs Agent (read + write):
- Adds Javadoc to public classes and methods
- Updates README endpoint tables
- Adds SpringDoc OpenAPI annotations"

git push origin main
```

---

## Lab Completion Criteria

- [ ] `.kiro/agents/` directory exists in the project root
- [ ] `code-review-agent.md` exists with `tools: [read]` only
- [ ] `test-generator-agent.md` exists with `tools: [read, write, shell]`
- [ ] `docs-agent.md` exists with `tools: [read, write]`
- [ ] `@code-review-agent` produces a structured review report with categories and line numbers
- [ ] `@test-generator-agent` creates test files and runs `mvn test` to confirm they pass
- [ ] `@docs-agent` adds accurate Javadoc and updates README
- [ ] All three agents are committed and pushed to Git
- [ ] Kiro can auto-select each agent based on its description (test this: ask "can you review my controller?" — it should suggest or use the code review agent)
