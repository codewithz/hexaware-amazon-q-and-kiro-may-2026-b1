# Lab 1 — Your First Agentic Feature with Amazon Q Developer

**Module:** 1.2 — Q Developer Setup & Agentic Capabilities  
**Duration:** 30 minutes  
**Time Slot:** 9:30 – 10:00 AM  
**Deliverable:** A working User Registration feature implemented entirely by the `/dev` agent

---

## Objective

In this lab, you will use Amazon Q Developer's `/dev` agentic mode to implement a complete user registration feature from a single natural language prompt.

The goal is **not** to write code manually. The goal is to learn how to direct an agent, review its work, and evaluate the generated implementation.

By the end of this lab, you will have:

- Used `/dev` for the first time
- Reviewed and accepted an agent-generated implementation
- Understood the difference between chat mode and agentic mode
- Practiced guiding the agent when the generated code needs correction

---

## Prerequisites

Before starting this lab, confirm the following:

- [ ] Amazon Q Developer extension is installed in VS Code or IntelliJ
- [ ] You are signed in with your AWS Builder ID or IAM Identity Center credentials
- [ ] You can see the Amazon Q icon in the left sidebar in VS Code or the Q panel in IntelliJ
- [ ] Your training repository is open in the IDE
- [ ] Java 17 is installed
- [ ] Maven is installed

Verify Java and Maven using the following commands:

```bash
java -version
mvn -version
```

If any of the above are missing, complete the prerequisite setup before continuing.

---

## Background — Chat Mode vs Agentic Mode

Amazon Q Developer has two distinct modes:

| Mode | What it does | When to use |
|---|---|---|
| **Chat** (`Q` panel) | Answers questions, explains code, and gives suggestions inline | Quick questions, code review, learning |
| **Agentic** (`/dev`) | Reads your codebase, writes files, runs builds, and iterates on errors | Implementing features, scaffolding, refactoring |

In chat mode, **you write the code**.

In agentic mode (`/dev`), **the agent writes the code**, builds it, reads errors, fixes problems, and iterates without you typing each line manually.

This lab uses **agentic mode**.

---

# Project Setup

## Step 1 — Clone or Initialise the Training Repository

If you already have the training repository open in your IDE, you can skip this step.

If you do not have the repository yet, clone it using the command below:

```bash
git clone <your-training-repo-url> ai-sdlc-training
cd ai-sdlc-training
```

---

## Option A — Create the Spring Boot Project Using the Command Line

If you are starting from scratch, create a new Spring Boot project using Spring Initializr through `curl`:

```bash
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
```

---

## Option B — Create the Spring Boot Project Using Spring Initializr Link

You can also create the same project using the Spring Initializr web interface.

### Use this link

Open the following link in your browser:

[Create `user-service` Spring Boot Project using Spring Initializr](https://start.spring.io/#!type=maven-project&language=java&platformVersion=4.0.6&packaging=jar&configurationFileFormat=properties&jvmVersion=17&groupId=com.hexaware&artifactId=user-service&packageName=com.hexaware&dependencies=web,validation,lombok,security,data-jpa,postgresql)

### Project Configuration

The link preselects the following configuration:

| Setting | Value |
|---|---|
| Project | Maven |
| Language | Java |
| Packaging | JAR |
| Java Version | 17 |
| Group ID | `com.hexaware` |
| Artifact ID | `user-service` |
| Package Name | `com.hexaware` |
| Dependencies | Spring Web, Validation, Lombok, Spring Security, Spring Data JPA, PostgreSQL Driver |

After opening the link:

1. Review the selected options.
2. Click **Generate**.
3. Extract the downloaded ZIP file.
4. Open the extracted `user-service` folder in your IDE.

---

## Step 2 — Open the Project in Your IDE

For VS Code:

```bash
code .
```

For IntelliJ:

1. Open IntelliJ IDEA.
2. Select **File > Open**.
3. Choose the project folder.
4. Wait for Maven dependencies to load.

---

## Step 3 — Verify Amazon Q Developer Is Active

In VS Code:

1. Click the Amazon Q icon in the left activity bar.
2. Open the **Chat** panel.
3. Confirm that you can see the chat input box.

Ask Q Developer the following question:

```text
What Java version is this project using?
```

Q Developer should inspect your `pom.xml` and tell you the Java version.

If it does not respond, check your sign-in status and re-authenticate if required.

---

# Running Lab 1

## Step 4 — Open the `/dev` Agent

In the Q Developer chat panel, type:

```text
/dev
```

Then press **Space**.

The panel will switch to agentic mode. You should see the input change to indicate that it is now in `/dev` context.

> **Note:** Do not press Enter until you have typed or pasted the full feature prompt.

---

## Step 5 — Type the Feature Prompt

Type or paste the following prompt exactly:

```text
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

---

## Step 6 — Watch the Agent Work

The agent will now perform several actions:

1. **Read your project structure**  
   It scans `pom.xml`, existing source files, and configuration.

2. **Plan the implementation**  
   It describes what it will create.

3. **Generate files**  
   It may create files such as:

   - `User.java`
   - `UserRepository.java`
   - `RegisterUserRequest.java`
   - `RegisterUserResponse.java`
   - `UserService.java`
   - `UserController.java`
   - `GlobalExceptionHandler.java`

4. **Run the build**  
   It compiles the project using Maven.

5. **Fix errors**  
   If the build fails, it reads the error and attempts to correct the issue automatically.

Do not interrupt the process. Watch what it does carefully because you will review the result in the next step.

**Expected duration:** 3–6 minutes depending on your machine and Amazon Q Developer tier.

---

## Step 7 — Review the Generated Implementation

Once the agent stops, open each generated file and review the implementation.

### Review `User.java` Entity

Check the following:

- [ ] Has `@Entity`
- [ ] Has `@Table(name = "users")`
- [ ] Has `@Id` with `@GeneratedValue`
- [ ] Has `email`, `firstName`, `lastName`, `passwordHash`, and `createdAt` fields
- [ ] `createdAt` is populated automatically using `@CreationTimestamp` or equivalent logic
- [ ] Email field has `@Column(unique = true)`

---

### Review `RegisterUserRequest.java` DTO

Check the following:

- [ ] Has `@NotBlank` on `firstName`
- [ ] Has `@NotBlank` on `lastName`
- [ ] Has `@NotBlank` on `password`
- [ ] Has `@Email` and `@NotBlank` on `email`
- [ ] Does not expose or contain a `passwordHash` field

---

### Review `UserService.java`

Check the following:

- [ ] Uses `BCryptPasswordEncoder` or an injected `PasswordEncoder`
- [ ] Checks for duplicate email before saving
- [ ] Throws a specific exception for duplicate email
- [ ] Does not use a generic `RuntimeException` for duplicate email
- [ ] Never stores the raw password
- [ ] Stores only the BCrypt hash

---

### Review `UserController.java`

Check the following:

- [ ] Endpoint is mapped to `POST /api/v1/users/register`
- [ ] Uses `@Valid` on the request body parameter
- [ ] Returns `ResponseEntity` with status `201 Created`
- [ ] Does not return the password hash in the response

---

### Review `GlobalExceptionHandler.java`, If Generated

Check the following:

- [ ] Catches `MethodArgumentNotValidException`
- [ ] Returns HTTP `400 Bad Request` with field-level validation errors
- [ ] Catches the duplicate email exception
- [ ] Returns HTTP `409 Conflict` for duplicate email

---

## Step 8 — Ask Q to Fix a Problem

If you notice any issue during the review, ask the agent to fix it using the chat panel.

Do not edit the file manually.

For example, if the response body returns the password hash, ask:

```text
The response body is returning the passwordHash field. Remove it from the 
response — the API should never expose hashed passwords.
```

Watch the agent make the change.

This is the core skill of the lab: **directing the agent through language instead of manually changing the code yourself**.

---

## Step 9 — Run the Application

Start the application:

```bash
mvn spring-boot:run
```

If PostgreSQL is not available locally, the application may fail to connect to the database. That is expected.

For this lab, the most important checkpoint is that the project **compiles without errors**.

If your instructor has configured H2 in-memory database for the lab, the full application should start and you can test the endpoint.

Use the following request:

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

```http
HTTP/1.1 201 Created
```

```json
{
  "id": 1,
  "firstName": "Jane",
  "lastName": "Smith",
  "email": "jane.smith@example.com",
  "createdAt": "2024-01-15T09:45:00Z"
}
```

---

## Step 10 — Commit the Implementation

Commit the generated implementation:

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

# Lab Completion Checklist

- [ ] Used `/dev` agent mode instead of chat mode to implement the feature
- [ ] Entity, Repository, Service, and Controller layers were generated
- [ ] Password is hashed with BCrypt
- [ ] Plain text password is never stored
- [ ] Endpoint is `POST /api/v1/users/register`
- [ ] API returns `201 Created` for successful registration
- [ ] API returns `400 Bad Request` for validation errors
- [ ] API returns `409 Conflict` for duplicate email
- [ ] Project compiles without errors using `mvn compile`
- [ ] Implementation is committed to Git

---

# Discussion Questions

After completing the lab, discuss the following:

1. What did the agent do well without being told explicitly?
2. What did you have to correct or guide it on?
3. How long would this implementation take if you had written it manually?
4. What parts of the generated code would you review most carefully before merging to production?

These questions will be discussed during the debrief before the break.

---

# Troubleshooting

## The `/dev` Command Is Not Available

Make sure you are signed in to Amazon Q Developer.

In VS Code, check the bottom status bar for Q sign-in status. Re-authenticate if required.

---

## The Agent Generated Files in the Wrong Package

Ask the agent:

```text
Move all the generated files into the com.training.userservice package.
```

If you used the Spring Initializr link from this lab, your package may be:

```text
com.hexaware
```

In that case, ask the agent:

```text
Move all the generated files into the com.hexaware package.
```

---

## The Build Fails with a Missing Dependency

Check that `spring-boot-starter-security` is present in `pom.xml`.

If it is missing, ask the agent:

```text
Add the Spring Security dependency to pom.xml and configure it to allow unauthenticated access to the registration endpoint.
```

---

## The Agent Stops Mid-Implementation

Ask the agent to continue:

```text
/dev continue
```

Or describe the remaining work clearly:

```text
Continue implementing the UserService — the UserController has been created but the service class is missing.
```
