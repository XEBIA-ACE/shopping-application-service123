# AGENTS.md — Shopping Application Service

## 1. Stack

| Technology | Role |
|---|---|
| **Node.js 20 LTS** | Runtime for API Gateway / BFF layer |
| **Express.js 4.x** | HTTP routing, middleware, session management, auth endpoints |
| **Spring Boot 3.x (Java 21)** | Core microservices: Product, Wishlist, Search orchestration |
| **React 18.x** | Frontend SPA (Vite-based) |
| **Elasticsearch 8.x** | Full-text product keyword search, relevance ranking |
| **MongoDB 7.x** | Wishlist storage, user session/profile documents |
| **MySQL 8.x** | Product catalogue, pricing, availability (relational integrity) |
| **Mongoose 8.x** | ODM for MongoDB in Node.js layer |
| **Sequelize 6.x** | ORM for MySQL in Node.js layer (auth/user tables) |
| **Spring Data JPA** | ORM for MySQL in Spring Boot services |
| **Spring Data Elasticsearch** | Elasticsearch client for Spring Boot Search service |
| **JWT (jsonwebtoken / Spring Security)** | Stateless auth tokens |
| **bcrypt** | Password hashing in Node.js auth service |
| **Jest + Supertest** | Unit & integration tests for Node.js/Express |
| **JUnit 5 + Mockito** | Unit & integration tests for Spring Boot |
| **React Testing Library + Vitest** | Frontend component and hook tests |
| **Docker + docker-compose** | Container orchestration for all services |
| **GitHub Actions** | CI pipeline |

---

## 2. Project Structure

```
shopping-app/
├── AGENTS.md                          # This file
├── tasks.md                           # Agent-generated task tracker (created before coding)
├── docker-compose.yml                 # Orchestrates all services + infrastructure
├── docker-compose.override.yml        # Local dev overrides (ports, volumes, hot-reload)
├── .env.example                       # Template for all environment variables
├── .github/
│   └── workflows/
│       ├── ci.yml                     # Main CI pipeline
│       └── pr-checks.yml              # Lint + test gate on PRs
│
├── gateway/                           # Node.js + Express API Gateway / BFF
│   ├── Dockerfile
│   ├── package.json
│   ├── jest.config.js
│   ├── .env.example
│   ├── src/
│   │   ├── app.js                     # Express app factory (no listen() here)
│   │   ├── server.js                  # Entry point — calls app.listen()
│   │   ├── config/
│   │   │   ├── db.js                  # Sequelize (MySQL) + Mongoose (MongoDB) init
│   │   │   ├── jwt.js                 # JWT secret, expiry config
│   │   │   └── env.js                 # Validated env vars via envalid
│   │   ├── middleware/
│   │   │   ├── authMiddleware.js      # JWT verification, attach req.user
│   │   │   ├── errorHandler.js        # Centralised error response formatter
│   │   │   ├── rateLimiter.js         # express-rate-limit config
│   │   │   └── requestLogger.js       # Morgan/pino request logging
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   │   ├── auth.routes.js     # POST /auth/register, /auth/login, /auth/logout
│   │   │   │   ├── auth.controller.js
│   │   │   │   ├── auth.service.js    # bcrypt, JWT sign/verify, session logic
│   │   │   │   ├── auth.model.js      # Sequelize User model (MySQL)
│   │   │   │   └── auth.validator.js  # Joi/Zod request validation schemas
│   │   │   ├── proxy/
│   │   │   │   ├── product.proxy.js   # http-proxy-middleware → Spring Product service
│   │   │   │   ├── search.proxy.js    # http-proxy-middleware → Spring Search service
│   │   │   │   └── wishlist.proxy.js  # http-proxy-middleware → Spring Wishlist service
│   │   │   └── health/
│   │   │       └── health.routes.js   # GET /health
│   │   └── utils/
│   │       ├── logger.js              # Pino logger singleton
│   │       └── asyncHandler.js        # Wraps async route handlers, forwards errors
│   └── tests/
│       ├── unit/
│       │   ├── auth.service.test.js
│       │   └── authMiddleware.test.js
│       └── integration/
│           ├── auth.routes.test.js    # Supertest against real Express app
│           └── health.routes.test.js
│
├── services/
│   │
│   ├── product-service/               # Spring Boot — Product catalogue
│   │   ├── Dockerfile
│   │   ├── pom.xml
│   │   └── src/
│   │       ├── main/
│   │       │   ├── java/com/shopping/product/
│   │       │   │   ├── ProductServiceApplication.java
│   │       │   │   ├── config/
│   │       │   │   │   ├── SecurityConfig.java        # JWT filter chain (stateless)
│   │       │   │   │   └── SwaggerConfig.java
│   │       │   │   ├── controller/
│   │       │   │   │   └── ProductController.java     # GET /products, GET /products/{id}
│   │       │   │   ├── service/
│   │       │   │   │   └── ProductService.java
│   │       │   │   ├── repository/
│   │       │   │   │   └── ProductRepository.java     # JPA repository (MySQL)
│   │       │   │   ├── model/
│   │       │   │   │   └── Product.java               # JPA entity
│   │       │   │   ├── dto/
│   │       │   │   │   ├── ProductRequestDto.java
│   │       │   │   │   └── ProductResponseDto.java
│   │       │   │   └── exception/
│   │       │   │       ├── GlobalExceptionHandler.java
│   │       │   │       └── ProductNotFoundException.java
│   │       │   └── resources/
│   │       │       ├── application.yml
│   │       │       └── application-test.yml
│   │       └── test/java/com/shopping/product/
│   │           ├── controller/ProductControllerTest.java
│   │           ├── service/ProductServiceTest.java
│   │           └── repository/ProductRepositoryIntegrationTest.java
│   │
│   ├── search-service/                # Spring Boot — Elasticsearch keyword search
│   │   ├── Dockerfile
│   │   ├── pom.xml
│   │   └── src/
│   │       ├── main/
│   │       │   ├── java/com/shopping/search/
│   │       │   │   ├── SearchServiceApplication.java
│   │       │   │   ├── config/
│   │       │   │   │   └── ElasticsearchConfig.java
│   │       │   │   ├── controller/
│   │       │   │   │   └── SearchController.java      # GET /search?q=&page=&size=
│   │       │   │   ├── service/
│   │       │   │   │   └── SearchService.java
│   │       │   │   ├── repository/
│   │       │   │   │   └── ProductSearchRepository.java # ElasticsearchRepository
│   │       │   │   ├── model/
│   │       │   │   │   └── ProductDocument.java        # @Document index mapping
│   │       │   │   └── dto/
│   │       │   │       └── SearchResponseDto.java
│   │       │   └── resources/
│   │       │       ├── application.yml
│   │       │       └── application-test.yml
│   │       └── test/java/com/shopping/search/
│   │           ├── controller/SearchControllerTest.java
│   │           └── service/SearchServiceTest.java
│   │
│   └── wishlist-service/              # Spring Boot — Wishlist (MongoDB)
│       ├── Dockerfile
│       ├── pom.xml
│       └── src/
│           ├── main/
│           │   ├── java/com/shopping/wishlist/
│           │   │   ├── WishlistServiceApplication.java
│           │   │   ├── config/
│           │   │   │   └── SecurityConfig.java
│           │   │   ├── controller/
│           │   │   │   └── WishlistController.java    # GET|POST|DELETE /wishlists/{userId}/items
│           │   │   ├── service/
│           │   │   │   └── WishlistService.java
│           │   │   ├── repository/
│           │   │   │   └── WishlistRepository.java    # MongoRepository
│           │   │   ├── model/
│           │   │   │   └── Wishlist.java              # @Document collection mapping
│           │   │   └── dto/
│           │   │       ├── WishlistItemRequestDto.java
│           │   │       └── WishlistResponseDto.java
│           │   └── resources/
│           │       ├── application.yml
│           │       └── application-test.yml
│           └── test/java/com/shopping/wishlist/
│               ├── controller/WishlistControllerTest.java
│               └── service/WishlistServiceTest.java
│
└── frontend/                          # React 18 SPA (Vite)
    ├── Dockerfile
    ├── package.json
    ├── vite.config.ts
    ├── vitest.config.ts
    ├── tsconfig.json
    ├── index.html
    └── src/
        ├── main.tsx                   # React DOM root
        ├── App.tsx                    # Router setup
        ├── api/
        │   ├── authApi.ts             # Axios calls to /auth/*
        │   ├── productApi.ts          # Axios calls to /products/*
        │   ├── searchApi.ts           # Axios calls to /search
        │   └── wishlistApi.ts         # Axios calls to /wishlists/*
        ├── components/
        │   ├── auth/
        │   │   ├── LoginForm.tsx
        │   │   └── RegisterForm.tsx
        │   ├── product/
        │   │   ├── ProductCard.tsx
        │   │   └── ProductList.tsx
        │   ├── search/
        │   │   └── SearchBar.tsx
        │   └── wishlist/
        │       └── WishlistPanel.tsx
        ├── hooks/
        │   ├── useAuth.ts
        │   ├── useSearch.ts
        │   └── useWishlist.ts
        ├── store/
        │   ├── authSlice.ts           # Redux Toolkit slice
        │   ├── searchSlice.ts
        │   └── store.ts
        ├── pages/
        │   ├── HomePage.tsx
        │   ├── LoginPage.tsx
        │   ├── ProductDetailPage.tsx
        │   └── WishlistPage.tsx
        └── tests/
            ├── components/
            │   ├── LoginForm.test.tsx
            │   ├── ProductCard.test.tsx
            │   └── WishlistPanel.test.tsx
            └── hooks/
                ├── useAuth.test.ts
                └── useSearch.test.ts
```

---

## 3. Required Workflow

The agent **must** follow these steps in order. Do not skip or reorder steps.

### Step 1 — Read All Specifications
- Read every story/spec file provided in the repository before writing any code.
- Identify all functional requirements, acceptance criteria, and data contracts.
- Note every API endpoint, request/response shape, and database schema requirement.

### Step 2 — Create `tasks.md`
- Create `tasks.md` at the repository root before writing any implementation code.
- Structure it with the following sections:

```markdown
# tasks.md

## Discovered Requirements
- [ ] <requirement derived from spec>

## Implementation Tasks
- [ ] TASK-001: Scaffold gateway package.json and Express app skeleton
- [ ] TASK-002: Implement User model (Sequelize/MySQL)
- [ ] TASK-003: Implement auth.service.js — register, login, logout
- [ ] TASK-004: Implement JWT middleware
- [ ] TASK-005: Scaffold product-service Spring Boot project
- [ ] TASK-006: Implement Product JPA entity and repository
- [ ] TASK-007: Implement ProductService and ProductController
- [ ] TASK-008: Scaffold search-service Spring Boot project
- [ ] TASK-009: Implement ProductDocument Elasticsearch mapping
- [ ] TASK-010: Implement SearchService and SearchController
- [ ] TASK-011: Scaffold wishlist-service Spring Boot project
- [ ] TASK-012: Implement Wishlist MongoDB document and repository
- [ ] TASK-013: Implement WishlistService and WishlistController
- [ ] TASK-014: Scaffold React frontend with Vite
- [ ] TASK-015: Implement auth pages and Redux slice
- [ ] TASK-016: Implement product listing and detail pages
- [ ] TASK-017: Implement search bar and results
- [ ] TASK-018: Implement wishlist panel
- [ ] TASK-019: Write unit tests — gateway (≥90% coverage)
- [ ] TASK-020: Write unit tests — product-service (≥90% coverage)
- [ ] TASK-021: Write unit tests — search-service (≥90% coverage)
- [ ] TASK-022: Write unit tests — wishlist-service (≥90% coverage)
- [ ] TASK-023: Write unit tests — frontend (≥90% coverage)
- [ ] TASK-024: Write integration tests for all services
- [ ] TASK-025: Configure docker-compose
- [ ] TASK-026: Configure GitHub Actions CI

## Validation Checklist
- [ ] All unit test suites pass
- [ ] Coverage ≥ 90% in every service
- [ ] docker-compose up --build starts all services cleanly
- [ ] /health endpoint returns 200
- [ ] No hardcoded secrets or credentials
```

- Mark tasks `[x]` as they are completed.

### Step 3 — Implement
- Implement tasks in the order listed in `tasks.md`.
- Complete each module fully (model → repository → service → controller → routes → tests) before moving to the next.
- Never leave a file in a broken/incomplete state when switching modules.
- Commit logical units: one task = one commit with message `TASK-XXX: <description>`.

### Step 4 — Test
- After each service is implemented, run its test suite and confirm it passes before continuing.
- **Node.js/Express gateway:** `cd gateway && npm test -- --coverage`
- **Spring Boot services:** `cd services/<name> && mvn test`
- **Frontend:** `cd frontend && npm run test -- --coverage`
- Fix all failures before proceeding to the next task.

### Step 5 — Validate
- Run `docker-compose up --build` and confirm all containers start without errors.
- Verify the `/health` endpoint of each service returns HTTP 200.
- Confirm coverage thresholds are met in every service (≥90%).
- Tick every item in the `tasks.md` Validation Checklist.
- Run a final lint pass: `npm run lint` (gateway + frontend), `mvn checkstyle:check` (Spring Boot services).

---

## 4