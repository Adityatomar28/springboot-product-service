#springboot-product-service


A Spring Boot REST API backend for an e-commerce application, built using a layered architecture. The service manages products and categories, supporting both an external product source (FakeStore API) and a self-hosted database-backed implementation via Spring Data JPA.

## Overview

- **Type:** Spring Boot REST API
- **Architecture:** Layered Architecture
- **Purpose:** Manage products and categories by exposing REST APIs. Initially fetches products from an external FakeStore API, and later supports storing and retrieving products from its own database using Spring Data JPA.

## Tech Stack

| Technology | Purpose |
|---|---|
| Java | Programming language |
| Spring Boot | Build REST APIs |
| Spring MVC | Handle HTTP requests |
| Spring Data JPA | Database access |
| Hibernate | ORM framework |
| MySQL | Database |
| Flyway | Database versioning |
| Maven | Build tool |
| RestTemplate | Call external APIs |
| DTO Pattern | Request/Response objects |
| Exception Handling | Global error handling (via `@ControllerAdvice`) |

## Architecture

```
                Client
                   │
            HTTP Request
                   │
          ProductController
                   │
        ProductService Interface
           /                 \
SelfProductService    FakeStoreProductService
                   │
            ProductRepository
                   │
                Hibernate
                   │
                 MySQL
```

## Project Structure

```
src
│
├── controller       # REST API endpoints
├── service           # Business logic (interface + implementations)
├── repository        # Spring Data JPA repositories
├── model              # JPA entities
├── dto                  # Request/response DTOs
├── exception        # Custom exceptions
├── advise             # Global exception handling (@ControllerAdvice)
├── configuration  # Spring beans (RestTemplate, ModelMapper, etc.)
├── resources        # application.properties, Flyway migrations
```

## Module Breakdown

### Controller (`controller`)
Exposes REST endpoints and contains no business logic. Responsibilities are limited to receiving requests, validating input, calling the service layer, and returning responses.

Example endpoints:
- `GET /products`
- `GET /products/{id}`
- `POST /products`
- `PUT /products/{id}`
- `DELETE /products/{id}`

### Service (`service`)
Contains all business logic, including validation, fetching from the database, calling the FakeStore API, and throwing exceptions when a resource isn't found.

The `ProductService` interface acts as an abstraction so the controller can switch implementations without changes, following the Dependency Inversion Principle:

```
Controller → ProductService (interface) → Implementation
```

- **`FakeStoreProductService`** — Calls `https://fakestoreapi.com` via `RestTemplate` and converts the JSON response into DTOs.
- **`SelfProductService`** — Uses `ProductRepository` to read/write from the application's own database.

### Repository (`repository`)
Spring Data JPA repositories (e.g. `ProductRepo extends JpaRepository<Product, Long>`). No SQL is written manually; Hibernate generates queries automatically.

### Model (`model`)
JPA entities representing database tables:
- **`Product`** — id, title, description, price, category
- **`Category`** — related to Product via a many-products-to-one-category relationship
- **`BaseModel`** — common fields (`id`, `createdAt`, `updatedAt`) extended by other entities to avoid duplicate code

### DTO (`dto`)
Data Transfer Objects used instead of exposing entities directly to clients. This protects internal/sensitive fields (e.g. `supplierCost`, `internalNotes`, `createdBy`) and provides:
- Security
- Loose coupling
- Validation
- Independent API design

Flow: `Entity → DTO → Client`

### Configuration (`configuration`)
Defines Spring beans such as `RestTemplate`, `ModelMapper`, and `ObjectMapper` as singletons.

### Exception & Advise (`exception`, `advise`)
Custom exceptions (e.g. `ProductNotFoundException`) are thrown instead of generic exceptions, and handled globally via `@ControllerAdvice` rather than repeating try/catch blocks in every controller.

Example error response:
```json
{
  "message": "Product not found"
}
```

### Resources (`resources`)
Contains `application.properties` (database config, server port, Hibernate config) and Flyway migration scripts under `db/migration` (e.g. `V1__.sql`, `V2__addNewCol.sql`). On startup, Flyway checks which migrations have already run and executes only new ones — no manual table creation needed.

### Projections
Uses Spring Data Projections to fetch only required columns (e.g. `id`, `title`) instead of `SELECT *`, improving query performance.

## Request Flow

```
Client
  ↓
HTTP Request
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
Hibernate
  ↓
MySQL
  ↓
Repository
  ↓
Service
  ↓
Controller
  ↓
JSON Response
```

## Key Design Notes

- **Why an interface for `ProductService`?** It abstracts the implementation, letting the controller stay unaware of whether data comes from FakeStore or the local database — adhering to the Dependency Inversion Principle.
- **Why use DTOs instead of exposing entities?** Prevents leaking internal-only fields, decouples the API contract from the database schema, and allows independent validation per request type.
- **Why Flyway?** Ensures database schema changes are versioned and applied automatically and consistently, which is critical in production environments.
