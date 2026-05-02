# Lab 1 — Your First Agentic Feature with Amazon Q Developer

**Module:** 1.2 — Q Developer Setup & Agentic Capabilities  
**Duration:** 30 minutes  
**Time Slot:** 9:30 – 10:00 AM  
**Deliverable:** A working User Registration feature implemented entirely by the `/dev` agent

---

## Objective

In this lab you will use Amazon Q Developer's `/dev` agentic mode to implement a complete user registration feature from a single natural language prompt. The goal is **not** to write code — it is to learn how to direct an agent and evaluate what it produces.

By the end you will have:
- Used `/dev` for the first time
- Reviewed and accepted an agent-generated implementation
- Understood the difference between chat mode and agentic mode

---

## Prerequisites

Before starting this lab, confirm the following:

- [ ] Amazon Q Developer extension is installed in VS Code or IntelliJ
- [ ] You are signed in with your AWS Builder ID or IAM Identity Center credentials
- [ ] You can see the Amazon Q icon in the left sidebar (VS Code) or the Q panel (IntelliJ)
- [ ] Your training repository is open in the IDE
- [ ] Java 17 and Maven are installed (`java -version` and `mvn -version` should both respond)

If any of the above are missing, complete the prerequisites setup before continuing.

---

## Background — Chat Mode vs Agentic Mode

Amazon Q Developer has two distinct modes:

| Mode | What it does | When to use |
|---|---|---|
| **Chat** (`Q` panel) | Answers questions, explains code, gives suggestions inline | Quick questions, code review, learning |
| **Agentic** (`/dev`) | Reads your codebase, writes files, runs builds, iterates on errors | Implementing features, scaffolding, refactoring |

In chat mode, you write the code. In agentic mode (`/dev`), the agent writes the code, builds it, reads the error, fixes it, and iterates — without you typing a single line.

This lab uses **agentic mode**.

---

## Project Setup

### Step 1 — Clone or initialise the training repository

If you do not already have the training repository open:

```bash
# Clone the training repository provided by your instructor
git clone <your-training-repo-url> ai-sdlc-training
cd ai-sdlc-training
```

If starting from scratch, create a new Spring Boot project:

```bash
# Use Spring Initializr via curl
curl https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.2.0 \
  -d baseDir=user-service \
  -d groupId=com.training \
  -d artifactId=user-service \
  -d name=user-service \
  -d dependencies=web,data-jpa,postgresql,security,validation,lombok \
  -o user-service.zip

unzip user-service.zip
cd user-service

Use this link: 

https://start.spring.io/#!type=maven-project&language=java&platformVersion=4.0.6&packaging=jar&configurationFileFormat=properties&jvmVersion=17&groupId=com.hexaware&artifactId=user-service&packageName=com.hexaware&dependencies=web,validation,lombok,security,data-jpa,postgresql
```

### Step 2 — Open the project in your IDE

```bash
# VS Code
code .

# IntelliJ — open via File > Open
```

### Step 3 — Verify Q Developer is active

In VS Code: open the **Chat** panel by clicking the Amazon Q icon (speech bubble with a Q) in the left activity bar. You should see a text input at the bottom.

Type the following and press Enter:
```
What Java version is this project using?
```

Q Developer should respond by inspecting your `pom.xml` and telling you the Java version. If it does not respond, check your sign-in status.

---

## Running Lab 1

### Step 4 — Open the /dev agent

In the Q Developer chat panel, type `/dev` and press **Space**. The panel will switch to agentic mode — you will see the input change to indicate it is in `/dev` context.

> **Note:** Do not press Enter until you have typed your full prompt.

### Step 5 — Type the feature prompt

Type the following prompt exactly (or paste it):

```
Implement a user registration feature for this Spring Boot application.

Requirements:
- POST /api/v1/users/register endpoint
- Accept: firstName, lastName, email, password (all required)
- Validate that email is a valid format and is unique in the database
- Hash the password using BCrypt before saving (never store plain text)
- Return 201 Created with the created user's id, firstName, lastName, email, and createdAt
- Return 409 Conflict if the email already exists
- Return 400 Bad Request with field-level error messages if validation fails

Use the existing Spring Boot + JPA setup. Create the User entity, UserRepository, 
UserService, and UserController. Follow standard layered architecture.
```

Press **Enter**.

### Step 6 — Watch the agent work

The agent will now:

1. **Read your project structure** — it scans `pom.xml`, existing source files, and configuration
2. **Plan the implementation** — you will see it describe what it will create
3. **Generate files** — it creates each file one by one:
   - `User.java` (entity)
   - `UserRepository.java`
   - `RegisterUserRequest.java` (DTO with validation annotations)
   - `RegisterUserResponse.java` (DTO)
   - `UserService.java`
   - `UserController.java`
   - Potentially a `GlobalExceptionHandler.java`
4. **Run the build** — it compiles the project with `mvn compile` or `mvn test`
5. **Fix errors** — if the build fails, it reads the error and makes corrections automatically

Do not interrupt this process. Watch what it does — you will evaluate it in the next step.

**Expected duration:** 3–6 minutes depending on your machine and Q tier.

---

### Step 7 — Review the generated implementation

Once the agent stops, open each generated file and check the following:

#### User.java (Entity)
- [ ] Has `@Entity` and `@Table(name = "users")`
- [ ] Has `@Id` with `@GeneratedValue`
- [ ] Has `email`, `firstName`, `lastName`, `passwordHash`, `createdAt` fields
- [ ] `createdAt` is populated automatically (`@CreationTimestamp` or equivalent)
- [ ] Email field has `@Column(unique = true)`

#### RegisterUserRequest.java (DTO)
- [ ] Has `@NotBlank` on firstName, lastName, password
- [ ] Has `@Email` and `@NotBlank` on email
- [ ] Does NOT expose or contain a `passwordHash` field

#### UserService.java
- [ ] Uses `BCryptPasswordEncoder` (or `PasswordEncoder` injected via Spring)
- [ ] Checks for duplicate email before saving
- [ ] Throws a specific exception for duplicate email (not a generic RuntimeException)
- [ ] Never stores the raw password — only the BCrypt hash

#### UserController.java
- [ ] Endpoint is mapped to `POST /api/v1/users/register`
- [ ] Uses `@Valid` on the request body parameter
- [ ] Returns `ResponseEntity` with status `201 Created`
- [ ] Does NOT return the password hash in the response

#### Global Exception Handler (if generated)
- [ ] Catches `MethodArgumentNotValidException` and returns 400 with field errors
- [ ] Catches duplicate email exception and returns 409

---

### Step 8 — Ask Q to fix a problem

If you notice any issue in the review above (e.g. the password hash is being returned in the response, or the endpoint URL is wrong), **ask the agent to fix it** using the chat panel. Do not edit the file manually.

Example:
```
The response body is returning the passwordHash field. Remove it from the 
response — the API should never expose hashed passwords.
```

Watch the agent make the change. This is the core skill: **directing the agent via language, not by typing code yourself**.

---

### Step 9 — Run the application

Start the application to verify it compiles and runs:

```bash
mvn spring-boot:run
```

If PostgreSQL is not available locally, the application will fail to connect to the database — this is expected. The important thing is that **the project compiles without errors**.

If using an H2 in-memory database for the lab (check with your instructor), the full application will start and you can test the endpoint:

```bash
curl -X POST http://localhost:8080/api/v1/users/register \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Jane",
    "lastName": "Smith",
    "email": "jane.smith@example.com",
    "password": "SecurePass123!"
  }'
```

Expected response:
```json
HTTP/1.1 201 Created

{
  "id": 1,
  "firstName": "Jane",
  "lastName": "Smith",
  "email": "jane.smith@example.com",
  "createdAt": "2024-01-15T09:45:00Z"
}
```

---

### Step 10 — Commit the implementation

```bash
git add .
git commit -m "feat: add user registration feature via Q Developer /dev agent

- POST /api/v1/users/register
- BCrypt password hashing
- Email uniqueness validation
- Field-level validation error responses
- 201 Created / 409 Conflict / 400 Bad Request responses

Generated using Amazon Q Developer /dev agent — Lab 1"
```

---

## Lab Completion Checklist

- [ ] Used `/dev` agent mode (not chat mode) to implement the feature
- [ ] All four layers generated: Entity, Repository, Service, Controller
- [ ] Password is hashed with BCrypt — never stored in plain text
- [ ] Endpoint is `POST /api/v1/users/register`
- [ ] Returns 201, 400, and 409 responses correctly
- [ ] Project compiles without errors (`mvn compile` exits 0)
- [ ] Implementation committed to Git

---

## Discussion Questions

After completing the lab, consider:

1. What did the agent do well without being told explicitly?
2. What did you have to correct or guide it on?
3. How long would this implementation take if you had written it manually?
4. What parts of the generated code would you want to review most carefully before merging to production?

These questions will be discussed during the debrief before the break.

---

## Troubleshooting

**The /dev command is not available**  
Make sure you are signed in to Amazon Q Developer. In VS Code, check the bottom status bar for the Q sign-in status. Re-authenticate if needed.

**The agent generated files in the wrong package**  
Ask the agent: *"Move all the generated files into the `com.training.userservice` package."*

**The build fails with a missing dependency**  
Check that `spring-boot-starter-security` is in `pom.xml`. If it is missing, ask the agent: *"Add the Spring Security dependency to pom.xml and configure it to allow unauthenticated access to the registration endpoint."*

**The agent stops mid-implementation**  
Type `/dev continue` or simply describe the remaining work: *"Continue implementing the UserService — the UserController has been created but the service class is missing."*
