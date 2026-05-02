# Lab 2 — Authoring Steering Files

**Module:** 1.4 — Authoring Steering Files (Layer 5)  
**Duration:** 45 minutes  

**Deliverable:** Three steering files committed to `.kiro/steering/` — verified as loaded and active

---

## Objective

Steering files are persistent, version-controlled instructions that the agent reads before every interaction. They eliminate the need to repeat your architecture decisions, library preferences, and coding standards in every prompt.

In this lab you will author three steering files:
- `architecture.md` — tech stack, approved libraries, forbidden patterns
- `testing-standards.md` — naming conventions, AAA pattern, Testcontainers rules
- `api-design.md` — URL conventions, error format, HTTP status codes, versioning

By the end you will have:
- Three steering files with correct YAML front matter
- Verified that the agent reads and applies steering file content
- Committed the files to Git so they travel with the repository

---

## Prerequisites

- [ ] Workshop 1 complete — governance branch exists
- [ ] Amazon Q Developer or Kiro IDE open with your training repository
- [ ] You are on a feature branch (not `main`)

Create a new branch for this lab:
```bash
git checkout -b feature/steering-files
```

---

> ### 💡 Why does this lab use `.kiro` when I installed Amazon Q?
>
> **Amazon Q Developer has been rebranded to Kiro.** The CLI, IDE extension, and all
> folder conventions now use the Kiro name. This happened gradually, which is why you
> may still see "Amazon Q" in some places (console, docs, billing) while the tooling
> itself already says "Kiro."
>
> | Old name (Amazon Q era) | New name (Kiro era) |
> |---|---|
> | Amazon Q Developer CLI | Kiro CLI (`kiro` / `q` both work) |
> | `.amazonq/` project folder | `.kiro/` project folder |
> | `.amazonq/rules/` | `.kiro/steering/` |
> | `~/.aws/amazonq/` global config | `~/.kiro/` global config |
>
> **Your existing `.amazonq/` folders still work** — Kiro reads both locations and
> prefers `.kiro/` when both exist. All new content you create in this workshop goes
> into `.kiro/` because that is the current standard.
>
> If your IDE title bar or chat panel still says "Amazon Q" rather than "Kiro," you
> are on an older extension version. The labs work either way — just use whichever
> chat panel is visible to you. The folder paths (`.kiro/steering/`) are the same.

---

## Understanding Steering Files

Steering files are Markdown files with a YAML front matter block that the agent reads on every invocation. The front matter controls when the file is applied.

```
---
inclusion: always
---
```

There are three inclusion modes:

| Mode | When the file is read |
|---|---|
| `always` | On every single agent interaction |
| `fileMatch: "src/**/*.java"` | Only when the agent is working on Java files in src/ |
| `manual` | Only when explicitly referenced with `#filename` |

For this lab, all three steering files use `always` — they apply globally.

> ### ⚠️ Front matter syntax — Amazon Q vs Kiro
>
> Older Amazon Q documentation shows the front matter key as `alwaysApply: true`.
> **Kiro uses `inclusion: always` instead.** Using the old syntax means the file will
> not be activated automatically — this is the most common reason steering files appear
> to be ignored.
>
> ```markdown
> # ❌ Old Amazon Q syntax — does NOT work in Kiro
> ---
> alwaysApply: true
> ---
>
> # ✅ Correct Kiro syntax
> ---
> inclusion: always
> ---
> ```
>
> If your verification prompts (Step 6) return wrong answers, check this first.

---

## Step 1 — Create the .kiro/steering directory

```bash
mkdir -p .kiro/steering
```

> **Note:** If your project already has a `.amazonq/rules/` folder from a previous
> session, leave it in place. Kiro will continue to read it. You do not need to delete
> or migrate it — just create the new `.kiro/steering/` path alongside it.

---

## Step 2 — Create architecture.md

Create the file `.kiro/steering/architecture.md`:

```bash
touch .kiro/steering/architecture.md
```

Open it and add the following content. **Read each section carefully — these become the agent's persistent knowledge of your project:**

````markdown
---
inclusion: always
---

# Architecture Standards

## Tech Stack

This project uses the following technology stack. Do not suggest alternatives 
unless explicitly asked.

| Layer | Technology | Version |
|---|---|---|
| Language | Java | 17 (LTS) |
| Framework | Spring Boot | 3.2.x |
| Database | PostgreSQL | 15+ |
| ORM | Spring Data JPA with Hibernate | Managed by Spring Boot |
| Messaging | Amazon SQS | AWS SDK v2 |
| Testing | JUnit 5 + Mockito + Testcontainers | Managed by Spring Boot |
| Build | Maven | 3.9+ |
| API Docs | SpringDoc OpenAPI (Swagger UI) | 2.x |

## Project Structure

Follow strict layered architecture. Every feature must have all four layers:

```
src/
  main/
    java/
      com/training/[service]/
        controller/     ← REST controllers only — no business logic
        service/        ← All business logic lives here
        repository/     ← Spring Data JPA interfaces only
        entity/         ← JPA entities
        dto/            ← Request and response DTOs (separate classes)
        exception/      ← Custom exception classes
        config/         ← Spring configuration classes
  test/
    java/
      com/training/[service]/
        controller/     ← Controller tests (MockMvc)
        service/        ← Service unit tests (Mockito)
        integration/    ← Full-stack integration tests (Testcontainers)
```

## Approved Libraries

Only use libraries from this list. Do not add new dependencies without 
discussion. If you need something not on this list, say so and explain why.

- `spring-boot-starter-web` — REST controllers
- `spring-boot-starter-data-jpa` — JPA and Hibernate
- `spring-boot-starter-validation` — Bean Validation (Jakarta)
- `spring-boot-starter-security` — Spring Security
- `spring-boot-starter-actuator` — Health checks and metrics
- `postgresql` — PostgreSQL JDBC driver
- `software.amazon.awssdk:sqs` — AWS SQS SDK v2
- `lombok` — Boilerplate reduction (@Data, @Builder, @RequiredArgsConstructor)
- `springdoc-openapi-starter-webmvc-ui` — OpenAPI / Swagger UI
- `mapstruct` — DTO ↔ Entity mapping (do not use manual mapping code)
- `testcontainers` — Integration test infrastructure
- `testcontainers:postgresql` — PostgreSQL Testcontainer
- `testcontainers:localstack` — AWS LocalStack for SQS integration tests

## Forbidden Patterns

Never use these patterns — if you see them in existing code, flag them:

- ❌ `@Autowired` on fields — always use constructor injection
- ❌ Business logic in `@Controller` or `@RestController` classes
- ❌ Direct SQL strings in service or controller classes (use JPA repositories)
- ❌ `System.out.println` — use `@Slf4j` and `log.info()` / `log.debug()`
- ❌ Catching and swallowing exceptions (`catch (Exception e) {}`)
- ❌ Storing passwords or secrets in plain text or in `.properties` files committed to Git
- ❌ `Optional.get()` without an `.isPresent()` check — use `.orElseThrow()`
- ❌ `@Transactional` on `@RestController` classes — put it on the service layer
````

---

## Step 3 — Create testing-standards.md

Create `.kiro/steering/testing-standards.md`:

```bash
touch .kiro/steering/testing-standards.md
```

Add the following content:

````markdown
---
inclusion: always
---

# Testing Standards

## Test Pyramid

Every feature must have tests at all three levels:

1. **Unit tests** — test individual classes in isolation with Mockito
2. **Controller tests** — test REST layer with MockMvc (no real database)
3. **Integration tests** — test the full stack with real PostgreSQL via Testcontainers

Aim for: 70% unit tests, 20% controller tests, 10% integration tests.

## Naming Convention

All test methods must follow this pattern:

```
methodName_stateUnderTest_expectedBehaviour
```

Examples:
```java
register_withValidRequest_returns201Created()
register_withDuplicateEmail_returns409Conflict()
register_withMissingFirstName_returns400WithFieldError()
findById_whenUserNotFound_throwsUserNotFoundException()
```

Do not use vague names like `testRegister()` or `happyPath()`.

## AAA Pattern

Every unit test must follow the Arrange-Act-Assert pattern with blank lines 
between sections and comments marking each section:

```java
@Test
void register_withValidRequest_returns201Created() {
    // Arrange
    RegisterUserRequest request = RegisterUserRequest.builder()
        .firstName("Jane")
        .lastName("Smith")
        .email("jane@example.com")
        .password("SecurePass123!")
        .build();
    
    User savedUser = User.builder()
        .id(1L)
        .firstName("Jane")
        .lastName("Smith")
        .email("jane@example.com")
        .createdAt(Instant.now())
        .build();
    
    when(userRepository.existsByEmail("jane@example.com")).thenReturn(false);
    when(userRepository.save(any(User.class))).thenReturn(savedUser);

    // Act
    RegisterUserResponse response = userService.register(request);

    // Assert
    assertThat(response.id()).isEqualTo(1L);
    assertThat(response.email()).isEqualTo("jane@example.com");
    verify(userRepository).save(any(User.class));
}
```

## Testcontainers Rules

- Always use `@Testcontainers` and `@Container` annotations (not manual lifecycle)
- Use a shared container across all integration tests in a class with `static`
- Never hardcode ports — use `container.getMappedPort(5432)`
- Use `@DynamicPropertySource` to inject container config into Spring context
- Use `localstack` for all AWS service integration tests (never call real AWS from tests)

Standard integration test setup:

```java
@SpringBootTest
@Testcontainers
@Transactional
class UserServiceIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserService userService;

    // tests here
}
```

## Test Data

- Never use production data in tests
- Use realistic but fake data (real-looking names, valid email formats)
- Use `@BeforeEach` to reset state between tests
- Do not share mutable state between test methods
````

---

## Step 4 — Create api-design.md

Create `.kiro/steering/api-design.md`:

```bash
touch .kiro/steering/api-design.md
```

Add the following content:

````markdown
---
inclusion: always
---

# API Design Standards

## URL Conventions

All REST endpoints must follow these rules:

- Base path: `/api/v1/` (all endpoints are versioned)
- Resource names are **plural nouns** — never verbs
- Use kebab-case for multi-word resource names

| ✅ Correct | ❌ Wrong |
|---|---|
| `GET /api/v1/users` | `GET /api/v1/getUsers` |
| `POST /api/v1/users` | `POST /api/v1/user/create` |
| `GET /api/v1/users/{id}` | `GET /api/v1/users/getById/{id}` |
| `PUT /api/v1/users/{id}` | `PUT /api/v1/updateUser/{id}` |
| `DELETE /api/v1/users/{id}` | `DELETE /api/v1/users/delete/{id}` |
| `POST /api/v1/stock-reservations` | `POST /api/v1/stockReservation` |

## HTTP Status Codes

Use the correct status code for every response:

| Situation | Status Code |
|---|---|
| Successful read | `200 OK` |
| Successful creation | `201 Created` with `Location` header |
| Successful delete | `204 No Content` |
| Validation error (missing/invalid fields) | `400 Bad Request` |
| Unauthenticated | `401 Unauthorized` |
| Insufficient permissions | `403 Forbidden` |
| Resource not found | `404 Not Found` |
| Business rule conflict (e.g. duplicate) | `409 Conflict` |
| Server error | `500 Internal Server Error` |

## Request and Response Format

All requests and responses use JSON with camelCase field names.

**Successful creation response (201):**
```json
{
  "id": 42,
  "createdAt": "2024-01-15T09:45:00Z",
  "... other resource fields"
}
```

**Location header on creation:**
```
Location: /api/v1/users/42
```

**400 Bad Request (validation errors):**
```json
{
  "status": 400,
  "error": "Validation Failed",
  "timestamp": "2024-01-15T09:45:00Z",
  "errors": [
    {
      "field": "email",
      "message": "must be a valid email address"
    },
    {
      "field": "firstName",
      "message": "must not be blank"
    }
  ]
}
```

**404 Not Found:**
```json
{
  "status": 404,
  "error": "Not Found",
  "message": "User with id 42 not found",
  "timestamp": "2024-01-15T09:45:00Z"
}
```

**409 Conflict:**
```json
{
  "status": 409,
  "error": "Conflict",
  "message": "A user with email 'jane@example.com' already exists",
  "timestamp": "2024-01-15T09:45:00Z"
}
```

## Pagination

All list endpoints that may return more than 20 results must support pagination:

```
GET /api/v1/users?page=0&size=20&sort=createdAt,desc
```

Response format for paginated endpoints:
```json
{
  "content": [ ... ],
  "totalElements": 150,
  "totalPages": 8,
  "size": 20,
  "number": 0,
  "first": true,
  "last": false
}
```

Use Spring Data's `Pageable` and `Page<T>` — do not implement pagination manually.

## Versioning

The current API version is `v1`. When breaking changes are needed:
- Create a `v2` package alongside `v1`
- Do not modify or delete v1 endpoints until all consumers have migrated
- Document the migration path in the OpenAPI spec
````

---

## Step 5 — Load steering files in Kiro (or verify in Q Developer)

### If using Kiro IDE:

1. Open the **Kiro panel** (ghost icon in left sidebar)
2. Expand the **Steering** section
3. You should see all three files listed:
   - `architecture.md`
   - `testing-standards.md`  
   - `api-design.md`
4. Each should show a green indicator meaning it is active

If the files do not appear, try closing and reopening Kiro, or run `Kiro: Reload Steering Files` from the command palette.

### If using Amazon Q Developer (older extension, not yet upgraded to Kiro):

Q Developer reads context from open files. Pin the steering files using the `@workspace` context:

In the chat panel, type:
```
@workspace What are the approved testing libraries for this project?
```

Q Developer will search the workspace including the `.kiro/steering/` directory.

> **Note:** The Amazon Q Developer extension does not have a dedicated Steering panel
> — that UI is a Kiro-only feature. Using `@workspace` achieves the same result: the
> agent reads your steering files before answering. Both approaches produce identical
> output in the verification prompts below.

---

## Step 6 — Verify the steering files are working

Run these verification prompts. The agent must respond using the content from your steering files, not from its general training knowledge.

**Test 1 — Tech stack:**
```
What integration test infrastructure should I use for database tests in this project?
```
✅ Expected: Testcontainers with PostgreSQL container — **not** H2 in-memory

**Test 2 — API design:**
```
What HTTP status code should I return when a resource is successfully created?
```
✅ Expected: `201 Created` with a `Location` header pointing to the new resource

**Test 3 — Code standards:**
```
How should I inject dependencies in Spring Boot service classes in this project?
```
✅ Expected: Constructor injection — **not** `@Autowired` on fields

**Test 4 — Test naming:**
```
How should I name a test for the case where a user registers with an email that already exists?
```
✅ Expected: Pattern `methodName_stateUnderTest_expectedBehaviour`, e.g. `register_withDuplicateEmail_returns409Conflict`

If any answer is wrong, check that:
- The file is saved
- The front matter has `inclusion: always` (not `alwaysApply: true` or other variations)
- The file is in `.kiro/steering/` (not `.kiro/` or a subdirectory of `src/`)

---

## Step 7 — Commit and push

```bash
git add .kiro/steering/
git commit -m "feat: add Layer 5 steering files for agent context

- architecture.md: tech stack, project structure, approved libs, forbidden patterns
- testing-standards.md: naming convention, AAA pattern, Testcontainers setup
- api-design.md: URL conventions, status codes, error format, pagination

All files use inclusion: always — active on every agent interaction
Lab 2 — Day 1"

git push origin feature/steering-files
```

---

## Lab Completion Checklist

- [ ] `.kiro/steering/architecture.md` created with correct front matter and content
- [ ] `.kiro/steering/testing-standards.md` created with correct front matter and content
- [ ] `.kiro/steering/api-design.md` created with correct front matter and content
- [ ] All three files show in the Kiro Steering panel (or verified via `@workspace` in Amazon Q)
- [ ] All four verification prompts returned correct answers from steering content
- [ ] Files committed to Git with descriptive commit message
- [ ] Branch pushed to remote

---

## Common Issues

**"Why is the folder called `.kiro` if we're using Amazon Q?"**  
Amazon Q Developer has been rebranded to Kiro. The `.kiro/` folder is the new
standard that replaces `.amazonq/`. See the callout at the top of this lab for the
full name mapping. Both tools read both folder locations — `.kiro/` takes priority
when both exist.

**Agent ignores steering files and uses its own defaults**  
Check the front matter — it must be `inclusion: always` (not `alwaysApply: true`).
The old Amazon Q documentation used a different key. Using the wrong key means the
file is treated as `manual` inclusion and will not load automatically.

**Files appear in Kiro panel but agent gives wrong answers**  
Confirm the files are not empty and the content is formatted correctly (valid
Markdown, not truncated).

**Test 1 fails — agent suggests H2**  
This often means `testing-standards.md` is not being read. Recheck the inclusion
front matter and the file path.

**I don't see a Steering panel — only a chat window**  
You are on the older Amazon Q Developer extension, not yet upgraded to Kiro. Use
`@workspace` in the chat panel instead (see Step 5). The steering files work
identically — only the UI for browsing them differs.

---

## What's Next

Your steering files are now active. Every agent interaction for the rest of Day 1 and Day 2 will be informed by these files — you will not need to repeat "use Testcontainers" or "return 201 Created" in your prompts. The agent already knows.

After lunch, you will use these steering files as the foundation for Lab 3 — where the agent generates a complete spec for the Inventory Management Service using the architecture and standards you have just defined.