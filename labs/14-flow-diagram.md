Prompt: I want to generate a mermaid diagram based flowchart of my whole project from entry point to all API's, so that any new developer onboarded can understand my flow


# Project Architecture Flowchart

> A visual guide to the full request flow — from application startup to every API endpoint.
> Intended for new developers onboarding to this service.

---

## Application Entry Point & Startup

```mermaid
flowchart TD
    A([🚀 JVM Start]) --> B[UserServiceApplication.main]
    B --> C[SpringApplication.run]
    C --> D[Spring Boot Auto-Configuration]
    D --> E1[SecurityConfig\nBCryptPasswordEncoder\nSecurityFilterChain]
    D --> E2[AwsConfig\nSqsClient]
    D --> E3[Spring Data JPA\nPostgreSQL DataSource]
    D --> E4[EnableScheduling\nScheduler Thread Pool]
    E4 --> F[ReservationExpirationService\nfixedRate = 5 min]
```

---

## Security Filter — Every Inbound Request

```mermaid
flowchart TD
    REQ([HTTP Request]) --> SF[Spring Security\nSecurityFilterChain]
    SF --> CHK{Path?}
    CHK -->|POST /api/v1/users/register| PERMIT[Permit All\nNo auth required]
    CHK -->|Any other path| AUTH{Authenticated?}
    AUTH -->|No| R401[401 Unauthorized]
    AUTH -->|Yes| CONT[Continue to Controller]
    PERMIT --> CONT
```

---

## User API — POST /api/v1/users/register

```mermaid
flowchart TD
    REQ([POST /api/v1/users/register\nUserRegistrationRequest JSON]) --> VAL{Bean Validation\n@Valid}
    VAL -->|Invalid fields| EH1[GlobalExceptionHandler\nMethodArgumentNotValidException]
    EH1 --> R400[400 Bad Request\nfield errors map]

    VAL -->|Valid| UC[UserController\n.registerUser]
    UC --> US[UserService\n.registerUser]

    US --> DB1{UserRepository\n.existsByEmail?}
    DB1 -->|true| EH2[GlobalExceptionHandler\nEmailAlreadyExistsException]
    EH2 --> R409[409 Conflict]

    DB1 -->|false| HASH[BCryptPasswordEncoder\n.encode password]
    HASH --> SAVE[UserRepository\n.save User entity]
    SAVE --> PGUSERS[(PostgreSQL\nusers table)]
    PGUSERS --> DTO[Map to\nUserRegistrationResponse]
    DTO --> R201[201 Created\nid, name, email, createdAt]
```

---

## Inventory API — GET /api/v1/inventory/products/{productId}

```mermaid
flowchart TD
    REQ([GET /api/v1/inventory/products/productId]) --> IC[InventoryController\n.getProductStock]
    IC --> IS[InventoryService\n.getProductStock]
    IS --> DB{InventoryRepository\n.findByProductId?}
    DB -->|Not found| EH[GlobalExceptionHandler\nInventoryNotFoundException]
    EH --> R404[404 Not Found]
    DB -->|Found| MAP[Map to InventoryResponse\ntotal / reserved / available qty]
    MAP --> R200[200 OK\nInventoryResponse]
```

---

## Inventory API — PUT /api/v1/inventory/products/{productId}

```mermaid
flowchart TD
    REQ([PUT /api/v1/inventory/products/productId\n?quantity=N]) --> IC[InventoryController\n.updateProductStock]
    IC --> IS[InventoryService\n.updateProductStock]

    IS --> LOCK{InventoryRepository\n.findByProductIdWithLock\npessimistic write lock}
    LOCK -->|Exists| UPD[Inventory.updateTotalQuantity\nadjust reserved if needed]
    LOCK -->|Not found| NEW[Create new\nInventory record]

    UPD --> SAVE[InventoryRepository.save]
    NEW --> SAVE
    SAVE --> PG[(PostgreSQL\ninventory table)]
    PG --> EVT[InventoryEventPublisher\n.publishInventoryChangeEvent\nSTOCK_UPDATED]
    EVT --> SQS[(AWS SQS\nFIFO Queue)]
    SQS --> R200[200 OK\nInventoryResponse]
```

---

## Inventory API — PUT /api/v1/inventory/bulk-update

```mermaid
flowchart TD
    REQ([PUT /api/v1/inventory/bulk-update\nBulkStockUpdateRequest JSON]) --> VAL{Bean Validation\n@Valid}
    VAL -->|Invalid| R400[400 Bad Request]
    VAL -->|Valid| IC[InventoryController\n.bulkUpdateStock]
    IC --> IS[InventoryService\n.bulkUpdateStock]

    IS --> LOOP[For each StockUpdateItem]
    LOOP --> UPD[updateProductStock\nproductId, quantity]
    UPD -->|Success| SC[successfulUpdates++]
    UPD -->|Exception| FC[failedUpdates++\nadd error message]
    SC --> LOOP
    FC --> LOOP

    LOOP -->|All items processed| RESP[BulkStockUpdateResponse\ntotal / successful / failed / errors]
    RESP --> R200[200 OK]
```

---

## Inventory API — POST /api/v1/inventory/reservations

```mermaid
flowchart TD
    REQ([POST /api/v1/inventory/reservations\nStockReservationRequest JSON]) --> VAL{Bean Validation\n@Valid}
    VAL -->|Invalid| R400[400 Bad Request]
    VAL -->|Valid| IC[InventoryController\n.reserveStock]
    IC --> IS[InventoryService\n.reserveStock]

    IS --> CHK1{StockReservationRepository\n.existsByOrderIdAndStatus ACTIVE?}
    CHK1 -->|true| EH1[GlobalExceptionHandler\nIllegalStateException]
    EH1 --> R409[409 Conflict\nDuplicate reservation]

    CHK1 -->|false| LOCK[InventoryRepository\n.findByProductIdWithLock\npessimistic write lock]
    LOCK -->|Not found| EH2[GlobalExceptionHandler\nInventoryNotFoundException]
    EH2 --> R404[404 Not Found]

    LOCK -->|Found| AVAIL{Inventory.canReserve\navailableQty >= requested?}
    AVAIL -->|No| EH3[GlobalExceptionHandler\nInsufficientStockException]
    EH3 --> R400B[400 Bad Request]

    AVAIL -->|Yes| RSV[Inventory.reserveStock\nreservedQty += quantity]
    RSV --> SAVEINV[InventoryRepository.save]
    SAVEINV --> SAVERSV[StockReservationRepository.save\nstatus=ACTIVE, expiresAt set]
    SAVERSV --> PG[(PostgreSQL\ninventory +\nstock_reservations)]
    PG --> EVT[InventoryEventPublisher\nSTOCK_RESERVED event]
    EVT --> SQS[(AWS SQS\nFIFO Queue)]
    SQS --> R201[201 Created\nStockReservationResponse]
```

---

## Inventory API — DELETE /api/v1/inventory/reservations

```mermaid
flowchart TD
    REQ([DELETE /api/v1/inventory/reservations\nStockReleaseRequest JSON]) --> VAL{Bean Validation\n@Valid}
    VAL -->|Invalid| R400[400 Bad Request]
    VAL -->|Valid| IC[InventoryController\n.releaseReservation]
    IC --> IS[InventoryService\n.releaseReservation]

    IS --> FIND{StockReservationRepository\n.findByOrderIdAndStatus ACTIVE?}
    FIND -->|Not found| EH1[GlobalExceptionHandler\nReservationNotFoundException]
    EH1 --> R404[404 Not Found]

    FIND -->|Found| LOCK[InventoryRepository\n.findByProductIdWithLock\npessimistic write lock]
    LOCK -->|Not found| EH2[GlobalExceptionHandler\nInventoryNotFoundException]
    EH2 --> R404B[404 Not Found]

    LOCK -->|Found| REL[Inventory.releaseReservation\nreservedQty -= quantity]
    REL --> SAVEINV[InventoryRepository.save]
    SAVEINV --> UPDRSV[StockReservation.release\nstatus = RELEASED]
    UPDRSV --> SAVERSV[StockReservationRepository.save]
    SAVERSV --> PG[(PostgreSQL)]
    PG --> EVT[InventoryEventPublisher\nSTOCK_RELEASED event]
    EVT --> SQS[(AWS SQS\nFIFO Queue)]
    SQS --> R204[204 No Content]
```

---

## Scheduled Job — Reservation Expiration (every 5 min)

```mermaid
flowchart TD
    SCHED([Spring Scheduler\nfixedRate = 300000ms]) --> RES[ReservationExpirationService\n.processExpiredReservations]
    RES --> IS[InventoryService\n.processExpiredReservations]
    IS --> QUERY[StockReservationRepository\n.findExpiredReservations\nstatus=ACTIVE AND expiresAt < now]
    QUERY --> LOOP[For each expired reservation]
    LOOP --> LOCK[InventoryRepository\n.findByProductIdWithLock]
    LOCK -->|null| SKIP[Skip — log warning]
    LOCK -->|Found| REL[Inventory.releaseReservation]
    REL --> SAVEINV[InventoryRepository.save]
    SAVEINV --> EXP[StockReservation.expire\nstatus = EXPIRED]
    EXP --> SAVERSV[StockReservationRepository.save]
    SAVERSV --> EVT[InventoryEventPublisher\nSTOCK_RELEASED event]
    EVT --> SQS[(AWS SQS\nFIFO Queue)]
    SQS --> LOOP
    LOOP -->|Done| LOG[Log: N reservations processed]
```

---

## Error Handling — GlobalExceptionHandler

```mermaid
flowchart LR
    EX1[EmailAlreadyExistsException] --> GEH[GlobalExceptionHandler\n@RestControllerAdvice]
    EX2[InsufficientStockException] --> GEH
    EX3[InventoryNotFoundException] --> GEH
    EX4[ReservationNotFoundException] --> GEH
    EX5[MethodArgumentNotValidException] --> GEH
    EX6[Exception catch-all] --> GEH

    GEH --> R409[409 Conflict]
    GEH --> R400[400 Bad Request]
    GEH --> R404[404 Not Found]
    GEH --> R500[500 Internal Server Error]
```

---

## Full Component Map

```mermaid
flowchart TD
    subgraph Entry
        MAIN[UserServiceApplication\n@SpringBootApplication\n@EnableScheduling]
    end

    subgraph Config
        SEC[SecurityConfig]
        AWS[AwsConfig → SqsClient]
    end

    subgraph Controllers
        UC[UserController\nPOST /api/v1/users/register]
        IC[InventoryController\nGET/PUT/POST/DELETE\n/api/v1/inventory/...]
    end

    subgraph Services
        US[UserService]
        IS[InventoryService]
        IEP[InventoryEventPublisher]
        RES[ReservationExpirationService\nScheduled]
        PS[PricingService\ncalculateDiscount\ncalculateTax]
    end

    subgraph Repositories
        UR[UserRepository]
        IR[InventoryRepository]
        SRR[StockReservationRepository]
    end

    subgraph Entities
        UE[User]
        IE[Inventory]
        SRE[StockReservation]
    end

    subgraph External
        PG[(PostgreSQL)]
        SQS[(AWS SQS FIFO)]
    end

    subgraph ExceptionHandling
        GEH[GlobalExceptionHandler\n@RestControllerAdvice]
    end

    MAIN --> Config
    MAIN --> Controllers
    MAIN --> RES

    UC --> US
    IC --> IS

    US --> UR
    IS --> IR
    IS --> SRR
    IS --> IEP
    RES --> IS

    IEP --> SQS

    UR --> PG
    IR --> PG
    SRR --> PG

    UR -.->|maps to| UE
    IR -.->|maps to| IE
    SRR -.->|maps to| SRE

    Controllers -.->|throws| GEH
    Services -.->|throws| GEH
```
