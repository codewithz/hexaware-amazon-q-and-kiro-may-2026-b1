# Lab 3 — Spec-Driven Development: Inventory Management Service

**Module:** 1.5 — Spec-Driven Development (Layer 2)
**Duration:** 60 minutes

**Deliverable:** A complete, approved spec committed to `.kiro/specs/inventory-service/`

---

## Objective

In this lab you will run the complete spec workflow for the first time. You write one prompt. The agent generates the requirements, architecture, and task plan. You review, refine, and approve each stage.

The spec you produce in this lab is the direct input for Capstone 1 — you will implement it using the agent immediately after.

By the end you will have:

- Written a feature prompt and reviewed Stage 1 requirements (EARS user stories)
- Refined the architecture artefacts in Stage 2 (DB schema, API endpoints, data flow)
- Approved the sequenced task plan in Stage 3
- A committed spec that the agent will use to drive implementation

**Estimated time per part:**

| Part | Activity | Time |
|------|----------|------|
| Part 1 | Write the feature prompt | ~5 min |
| Part 2 | Stage 1 — Requirements review | ~15 min |
| Part 3 | Stage 2 — Architecture review | ~20 min |
| Part 4 | Stage 3 — Task plan review | ~10 min |
| Part 5 | Save and commit the spec | ~10 min |

---

## Prerequisites

- [ ] Lab 2 complete — steering files loaded and verified
- [ ] Kiro IDE open (or Amazon Q Developer with `/dev` access)
- [ ] You are on a feature branch:

```bash
git checkout -b feature/inventory-spec
```

---

## What is a Spec?

A spec in the context of agentic development is a structured document that contains:

1. **Requirements** — EARS user stories with acceptance criteria
2. **Design** — technical architecture decisions (DB schema, API design, data flow)
3. **Tasks** — a dependency-ordered implementation plan the agent will execute

The spec answers: *"Before a single line of code is written, do we all agree on exactly what we're building and how?"*

Unlike traditional specs written in a wiki and never updated, these specs are:

- Stored in the repository alongside the code
- Version-controlled with Git
- Read by the agent as context during implementation
- Updated when constraints change (not abandoned)

---

## Part 1 — Write the Feature Prompt (~5 min)

### Step 1 — Open the spec workflow

**In Kiro:**

- Open the Chat panel
- Click the **Spec** button above the input box (or type `#spec` to reference the spec context)
- The panel will prompt you for a feature description

**In Amazon Q Developer:**

- In the chat panel type: `/dev` followed by your prompt

### Step 2 — Enter the feature prompt

Type or paste the following prompt exactly:

```
Build an Inventory Management Service for our e-commerce platform.

The service must:
- Track product stock levels, with separate tracking of: total quantity on hand,
  reserved quantity (held for pending orders), and available quantity (total minus reserved)
- Support stock reservation when an order is placed: deduct from available,
  add to reserved — fail if available stock is insufficient
- Release reservations automatically when an order is cancelled or expires
- Publish an inventory change event to Amazon SQS whenever available quantity changes
- Expose a REST API following our team API design standards
- Support bulk stock updates for warehouse restocking operations
- Allow administrators to view current stock levels for any product

The service will be called by:
- Order Service: creates and releases reservations
- Warehouse Management System: sends bulk stock updates
- Admin Dashboard: reads current stock levels

Constraints:
- Use the tech stack and libraries from our steering files
- Follow the testing standards defined in steering files
- All endpoints must follow our API design standards
```

Press **Enter** and wait. The agent will generate Stage 1.

---

## Part 2 — Stage 1: Requirements Review (~15 min)

### What is EARS?

> **EARS** stands for **Easy Approach to Requirements Syntax**. It is a structured way of writing requirements so that every story has an unambiguous trigger and a deterministic system response — eliminating the vague language ("should", "may", "handles") that causes misunderstandings between engineers and between humans and agents.

EARS stories follow one of these patterns:

| Pattern | Use when… |
|---------|-----------|
| **WHEN** [trigger] **THE SYSTEM SHALL** [behaviour] | An event occurs |
| **IF** [condition] **THE SYSTEM SHALL** [behaviour] | A precondition is checked |
| **WHERE** [context] **THE SYSTEM SHALL** [behaviour] | An environmental constraint applies |

Patterns can be combined: `WHEN … IF … THE SYSTEM SHALL …`

### Step 3 — Review the generated requirements

The agent should produce stories similar to these. Read each one and evaluate it against the EARS pattern — not just whether the content sounds right, but whether the trigger and the behaviour are both specific enough to be testable.

**Expected stories (verify each one exists):**

```
US-01: WHEN a warehouse admin submits a stock update for a product,
       THE SYSTEM SHALL update the total quantity on hand and recalculate
       available quantity as (total - reserved).

US-02: WHEN the Order Service submits a reservation request for a product quantity,
       IF available quantity is sufficient,
       THE SYSTEM SHALL deduct the quantity from available stock,
       add it to reserved quantity, and return a reservation ID.

US-03: WHEN the Order Service submits a reservation request for a product quantity,
       IF available quantity is insufficient,
       THE SYSTEM SHALL reject the request with a 409 Conflict response
       and return the current available quantity.

US-04: WHEN the Order Service releases a reservation,
       THE SYSTEM SHALL deduct the quantity from reserved stock and
       add it back to available stock.

US-05: WHEN available quantity changes for any product,
       THE SYSTEM SHALL publish an InventoryChangedEvent to the
       configured Amazon SQS queue within 5 seconds.

US-06: WHEN an admin requests the current stock level for a product,
       THE SYSTEM SHALL return total quantity, reserved quantity,
       and available quantity.

US-07: WHEN a bulk stock update is submitted,
       THE SYSTEM SHALL process all items atomically — either all items
       update successfully or none do.
```

### Step 4 — Identify what is missing

Look for the concurrent reservation scenario — this is a common edge case that agents often miss on first generation.

> **Why this matters:** Without a concurrency requirement, two orders can simultaneously read the same available stock level, both pass the sufficiency check, and both succeed — leaving you with negative stock. This is not a theoretical edge case; it is the first failure mode that appears under real load.

⚠️ **If US-08 is missing, request it:**

```
What happens if two orders attempt to reserve the last 3 units of the same
product at exactly the same time? Add a requirement for concurrent reservation safety.
```

The agent should add a story like:

```
US-08: WHEN two concurrent reservation requests compete for the same product,
       THE SYSTEM SHALL use optimistic locking to ensure only one succeeds
       when combined reservations would exceed available quantity.
```

### Step 5 — Approve Stage 1

Once you are satisfied with the requirements (at minimum US-01 through US-07 plus the concurrency story), approve Stage 1 by clicking **Approve** or typing:

```
The requirements look good. Proceed to the architecture design stage.
```

---

## Part 3 — Stage 2: Architecture Review (~20 min)

### Step 6 — Review the database schema

The agent should produce a `ProductStock` entity similar to this:

```sql
CREATE TABLE product_stock (
    id              BIGSERIAL PRIMARY KEY,
    product_id      VARCHAR(255) NOT NULL UNIQUE,
    total_quantity  INTEGER NOT NULL DEFAULT 0,
    reserved_qty    INTEGER NOT NULL DEFAULT 0,
    available_qty   INTEGER NOT NULL DEFAULT 0,
    version         BIGINT NOT NULL DEFAULT 0,   -- optimistic locking
    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE TABLE stock_reservation (
    id              BIGSERIAL PRIMARY KEY,
    reservation_id  UUID NOT NULL UNIQUE,
    product_id      VARCHAR(255) NOT NULL,
    quantity        INTEGER NOT NULL,
    order_id        VARCHAR(255) NOT NULL,
    status          VARCHAR(50) NOT NULL,  -- ACTIVE, RELEASED, EXPIRED
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL,
    expires_at      TIMESTAMP WITH TIME ZONE
);
```

**Check for:**

- [ ] `version` column exists on `product_stock` — this is the optimistic locking field for US-08
- [ ] `available_qty` is a stored column, not a computed view (for query performance)
- [ ] `stock_reservation` table tracks reservation lifecycle (ACTIVE → RELEASED / EXPIRED)

> **Why the `version` column matters:**
> The SQL column alone is not enough. The JPA entity must also annotate this field with `@Version`. Without the annotation, the ORM will not use the column for concurrency checking — it will exist in the schema but have no effect at runtime. Verify **both** the SQL schema and the entity class.

⚠️ **If `version` is missing from the schema**, ask:

```
The product_stock table is missing the version column for optimistic locking.
Add it as a BIGINT with a default of 0, and make sure the JPA entity uses
@Version on this field.
```

⚠️ **If the schema is correct but you want to confirm the JPA entity**, ask:

```
Show me the ProductStock JPA entity class. Confirm the version field
is annotated with @Version.
```

### Step 7 — Review the API endpoints

The agent should define these endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/stock/{productId}` | Get current stock levels |
| `POST` | `/api/v1/stock/reservations` | Create a reservation |
| `DELETE` | `/api/v1/stock/reservations/{reservationId}` | Release a reservation |
| `PUT` | `/api/v1/stock/bulk-update` | Bulk stock update |
| `GET` | `/api/v1/stock` | List all products with stock (paginated) |

**Check for:**

- [ ] All paths start with `/api/v1/` — matches the `api-design.md` steering file
- [ ] `POST /api/v1/stock/reservations` returns **201 Created** with a `Location` header
- [ ] `DELETE /api/v1/stock/reservations/{reservationId}` returns **204 No Content**
- [ ] The bulk update uses `PUT` (idempotent), not `POST`

> **Why `PUT` for bulk update?** `PUT` is idempotent — sending the same request twice produces the same result. `POST` is not. For a restocking operation, idempotency means a retry due to a network timeout cannot accidentally double-update stock levels.

⚠️ **If any endpoint uses the wrong HTTP method or status code**, correct it:

```
The reservation creation endpoint should return 201 Created with a Location
header pointing to /api/v1/stock/reservations/{reservationId}. Update the design.
```

### Step 8 — Review the data flow for reservation creation

The agent should describe a data flow similar to:

```
POST /api/v1/stock/reservations
  → InventoryController.createReservation()
  → StockReservationService.reserve()
  → [transaction begins]
  → ProductStockRepository.findByProductIdWithPessimisticLock()
     OR ProductStockRepository.findByProductId() + @Version optimistic lock
  → Check available_qty >= requested quantity
  → If insufficient: throw InsufficientStockException → 409 Conflict
  → If sufficient:
      → Deduct from available_qty, add to reserved_qty
      → Save ProductStock (version incremented)
      → Create StockReservation record
      → [transaction commits]
      → Publish InventoryChangedEvent to SQS (after commit, not inside transaction)
      → Return 201 Created with ReservationResponse
```

**Check for:**

- [ ] SQS event is published **after** the transaction commits — not inside it
- [ ] Concurrency is handled (optimistic or pessimistic locking — either is acceptable, see callout below)
- [ ] Transaction boundary is at the service layer, not the controller

> **Optimistic vs. pessimistic locking — which should you accept?**
>
> Both handle concurrent reservations correctly. The tradeoff is:
>
> | | Optimistic (`@Version`) | Pessimistic (`SELECT … FOR UPDATE`) |
> |-|------------------------|--------------------------------------|
> | **How it works** | Detects conflict at commit; retries or fails | Holds a DB row lock for the duration of the transaction |
> | **Best for** | Low-to-medium contention (most e-commerce products) | High contention (flash sales, limited-edition drops) |
> | **Risk** | Retry storms under very high load | Lock waits and potential deadlocks |
>
> For this service, **optimistic locking is the preferred default**. Accept pessimistic locking only if the agent provides a clear justification.

> **Why SQS must be published after commit:**
> If the event fires inside the transaction and the transaction subsequently rolls back (e.g. due to a DB error), downstream systems have already been notified of an inventory change that never happened. Publishing after commit means the event reflects state that is durably persisted. This is a correctness rule, not a performance preference.

### Step 9 — Approve Stage 2

Once satisfied with the architecture:

```
Architecture looks correct. I've verified the optimistic locking column,
the correct API status codes, and the SQS event publishing after commit.
Proceed to generate the task plan.
```

---

## Part 4 — Stage 3: Task Plan Review (~10 min)

### Step 10 — Review the task sequencing

The agent will generate an ordered list of implementation tasks. The order matters — each task should depend only on tasks already completed.

**Verify this order (roughly):**

```
Task 1:  Set up Maven project structure and dependencies
Task 2:  Create ProductStock JPA entity with @Version field
Task 3:  Create StockReservation JPA entity
Task 4:  Create ProductStockRepository and StockReservationRepository
Task 5:  Implement StockReservationService with reservation logic
Task 6:  Implement SQS event publishing (InventoryEventPublisher)
Task 7:  Implement bulk stock update in StockManagementService
Task 8:  Create InventoryController with all endpoints
Task 9:  Add GlobalExceptionHandler for InsufficientStockException and others
Task 10: Write unit tests for StockReservationService
Task 11: Write MockMvc controller tests for InventoryController
Task 12: Write integration tests with Testcontainers (PostgreSQL + LocalStack SQS)
Task 13: Add OpenAPI annotations and verify Swagger UI
```

> **What is LocalStack?** LocalStack is a local emulator for AWS services. Rather than running tests against a real SQS queue in AWS, Testcontainers spins up a LocalStack container that behaves identically. This keeps integration tests fast, free, and runnable offline. You do not need to set up LocalStack manually — the Testcontainers configuration in the test code handles it.

**Check for ordering problems:**

- Tests (Tasks 10–12) must come **after** implementation (Tasks 2–9)
- Repository layer (Task 4) must come **before** service layer (Task 5)
- Entity layer (Tasks 2–3) must come **before** repository layer (Task 4)

The correct mental model is: **entities → repositories → services → controllers → tests**

⚠️ **If the order is wrong**, ask:

```
Task 10 (unit tests) appears before Task 5 (service implementation).
Reorder — all implementation tasks must precede all test tasks.
```

### Step 11 — Approve Stage 3

```
Task plan looks good and sequencing is correct. Approve and save the spec.
```

---

## Part 5 — Save and Commit the Spec (~10 min)

### Step 12 — Save the spec files

Kiro will automatically save the spec to `.kiro/specs/inventory-service/`. Verify the files exist:

```bash
ls .kiro/specs/inventory-service/
# Expected output:
# requirements.md
# design.md
# tasks.md
```

If using Q Developer, create the directory and save manually:

```bash
mkdir -p .kiro/specs/inventory-service
```

Then ask the agent:

```
Save the approved spec to .kiro/specs/inventory-service/ as three separate files:
requirements.md, design.md, and tasks.md
```

### Step 13 — Confirm your branch and commit

First, confirm you are on the correct branch before committing:

```bash
git branch --show-current
# Expected: feature/inventory-spec
```

Then commit:

```bash
git add .kiro/specs/
git commit -m "spec: inventory management service — approved spec

Stage 1: 8 EARS user stories including concurrent reservation safety (US-08)
Stage 2: ProductStock + StockReservation schema, 5 REST endpoints,
         SQS event flow with post-commit publishing
Stage 3: 13-task implementation plan, dependency-ordered

Ready for Capstone 1 implementation
Lab 3 — Day 1"

git push origin feature/inventory-spec
```

> **Why `spec:` at the start of the commit message?** This follows the [Conventional Commits](https://www.conventionalcommits.org/) standard — a lightweight convention for structuring commit messages so that tooling (changelogs, CI pipelines, release notes) can parse them automatically. The prefix `spec:` signals that this commit contains specification artefacts, not implementation code. Other prefixes you will see in this course: `feat:`, `fix:`, `test:`, `chore:`.

---

## Lab Completion Checklist

Run through these in order. Each item is a concrete verification, not just a re-read of the steps above.

**Stage 1**

- [ ] All 8 user stories present (US-01 through US-08)
- [ ] US-08 explicitly addresses concurrent reservation — not just "handle errors"
- [ ] Stage 1 approved

**Stage 2**

- [ ] `product_stock` table has a `version BIGINT` column
- [ ] JPA entity confirmed to have `@Version` annotation on the version field
- [ ] `POST /api/v1/stock/reservations` returns 201 + `Location` header
- [ ] `DELETE /api/v1/stock/reservations/{reservationId}` returns 204
- [ ] Bulk update uses `PUT`, not `POST`
- [ ] SQS publish happens after transaction commit in the data flow diagram
- [ ] Stage 2 approved

**Stage 3**

- [ ] Entity tasks precede repository tasks
- [ ] Repository tasks precede service tasks
- [ ] All test tasks follow all implementation tasks
- [ ] Stage 3 approved

**Spec files**

- [ ] `ls .kiro/specs/inventory-service/` shows exactly three files: `requirements.md`, `design.md`, `tasks.md`
- [ ] `git log --oneline -1` shows your commit on `feature/inventory-spec`

---

## Key Takeaways

- **Your job is to review, not to write.** The agent generates the spec from one prompt; your skill is in evaluating what it produces — catching omissions, correcting errors, and enforcing standards before a single line of code is written.

- **The concurrency story (US-08) is the difference between a spec and a naive list.** The agent will usually produce the happy path correctly. Edge cases — especially timing and contention — require your active scrutiny.

- **The `version` column must exist in both the schema and the JPA entity.** A column without `@Version` provides no runtime protection. Verify both.

- **SQS after commit is a correctness rule.** Events published inside a transaction can notify downstream systems of updates that subsequently roll back. Always verify the data flow positions the publish call after the transaction boundary.

- **Task ordering reflects real dependency structure.** If tests appear before their implementation targets, the agent will attempt to run tests against code that does not yet exist. Catch this in the plan before implementation begins.

---

## What's Next

After the break, Lab 4 will use the spec you just approved to run a security scan and generate tests — before you write a single line of implementation code. This reverses the typical order: in spec-driven development, you define quality gates before building, not after.

Capstone 1 immediately after will implement the full spec using the agent. Keep the spec open — you will use it as the implementation guide.