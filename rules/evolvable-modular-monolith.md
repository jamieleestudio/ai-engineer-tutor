# Evolvable Modular Monolith Architecture Rules for AI

This document serves as the **primary instruction set** for AI assistants when generating, refactoring, or analyzing code within this project.

**CRITICAL INSTRUCTION**: You must adhere to these rules strictly. Any deviation constitutes an architecture violation.

---

## 1. Core Philosophy & Strategy

- **Deployment Mode**: Monolithic deployment (single Spring Boot app), but **Microservice-Ready structure**.
- **Communication**: All cross-module communication MUST go through explicit API modules (`xxx-api`).
- **Evolution Strategy**: Start with a single module (`xxx`). Extract `xxx-api` ONLY when another module needs to call it.
- **Dependency Direction**: `Upper Layer` -> `Lower Layer`, `Implementation` -> `Contract`.

## 2. Physical Structure (Maven Modules)

### Rule 2.1: Module Types
For a business domain (e.g., `user`):

| Module Type | Suffix | Content Allowed | Content Forbidden |
| :--- | :--- | :--- | :--- |
| **API Module** | `-api` | Interfaces (`Service`), DTOs, Enums, Events | Implementation logic, Spring Context, Persistence, Web (HttpServlet) |
| **Impl Module** | (none) | Domain logic, DB access, Controller, Listeners | Direct dependency on other Impl modules |

### Rule 2.2: Dependency Graph
- `xxx` (Impl) -> depends on -> `xxx-api` (Self API)
- `xxx` (Impl) -> depends on -> `yyy-api` (Other API)
- **FORBIDDEN**: `xxx` (Impl) -> depends on -> `yyy` (Other Impl)

## 3. Logical Layering (Inside Implementation Module)

You MUST structure the `src/main/java/com/example/{module}` directory as follows:

```text
com.example.{module}
├── interfaces          (Adapters/Entry Points)
│   ├── web             -> @RestController (HTTP for Frontend/App)
│   ├── facade          -> Service Impl of `xxx-api` (RPC for Backend)
│   └── listener        -> MQ Consumers
├── application         (Orchestration)
│   └── service         -> Business Flows, DTO<>Entity Mapping
├── domain              (Core Business Logic - PURE JAVA)
│   ├── model           -> Entities, Aggregates, VOs
│   ├── service         -> Domain Services (Complex Rules)
│   └── repository      -> Repository Interfaces ONLY
└── infrastructure      (Technical Implementation)
    ├── persistence     -> JPA/MyBatis Impl, POs
    ├── client          -> Feign/RPC Clients
    └── config          -> Spring Configurations
```

### Rule 3.1: Layer Dependencies
1.  `Interfaces` -> `Application`
2.  `Application` -> `Domain`
3.  `Infrastructure` -> `Domain` (Inversion of Control)
4.  **STRICTLY FORBIDDEN**: `Domain` -> `Infrastructure`, `Domain` -> `Application`, `Domain` -> `Interfaces`.

## 4. Coding Conventions

### 4.1 Naming Standards
| Object Type | Suffix | Location | Note |
| :--- | :--- | :--- | :--- |
| **RPC Interface** | `Service` | `xxx-api` | e.g., `UserReadService` |
| **RPC Input** | `Request` | `xxx-api` | Must be Serializable |
| **RPC Output** | `DTO` | `xxx-api` | Anemic model, NO Entities |
| **Web Controller** | `Controller` | `interfaces.web` | BFF pattern |
| **Web Input** | `Request` | `interfaces.web` | HTTP Request Body |
| **Web Output** | `Response` | `interfaces.web` | HTTP Response Body |
| **App Input** | `Command`/`Query` | `application` | Application Layer Input |
| **Domain Event** | `Event` | `domain/api` | Integration events go to API |

### 4.2 API Design
- **RPC APIs** (`xxx-api`):
    - MUST NOT expose `Entity` objects.
    - MUST NOT throw checked exceptions (use `Result<T>` or RuntimeException).
    - Prefer separating `ReadService` and `WriteService`.
- **Web APIs** (RESTful):
    - **Resource-Oriented**: URLs MUST represent resources (nouns), not actions (verbs). E.g., `POST /users` (good), `POST /createUser` (bad).
    - **HTTP Methods**: Use standard methods strictly (GET for retrieval, POST for creation, PUT/PATCH for update, DELETE for removal).
    - **Status Codes**: Return explicit status codes (201 Created, 204 No Content, 400 Bad Request, 404 Not Found).
    - **Stateless**: The server must not store client state between requests.

## 5. AI Code Generation Workflows

### Scenario A: Creating a New Feature (Single Module)
1.  Define Domain Model (`domain.model`).
2.  Define Repository Interface (`domain.repository`).
3.  Implement Persistence (`infrastructure.persistence`).
4.  Define Application Service (`application.service`) to orchestrate.
5.  Expose via Web Controller (`interfaces.web`).
6.  **Do NOT create an API module** unless explicitly requested or required by another module.

### Scenario B: Cross-Module Call (e.g., Order calls User)
1.  Check if `user-api` exists. If not, **create it**.
2.  Move `UserReadService` interface and DTOs from `user` to `user-api`.
3.  Refactor `user` implementation to implement the interface in `interfaces.facade`.
4.  Add dependency `implementation project(':user-api')` to `order`.
5.  Inject `UserReadService` in `order`.

## 6. Architecture Guard Rails (Self-Correction)

Before outputting code, verify:
1.  [ ] Did I import any class from `com.example.othermodule` (except `api`)? -> **Fix: Use API module.**
2.  [ ] Did I use an `@Entity` in a Controller return type? -> **Fix: Use DTO.**
3.  [ ] Is `Domain` depending on `Spring` (except basic annotations)? -> **Fix: Remove dependency.**
4.  [ ] Is business logic leaking into `Controller`? -> **Fix: Move to Application Service.**

## 7. ArchUnit Enforcement

The following ArchUnit rules MUST be implemented in the project's test suite to automatically reject architectural violations.

### 7.1 Layered Architecture Definition
```java
layeredArchitecture()
    .consideringOnlyDependenciesInAnyPackage("com.example..")
    .layer("Interfaces").definedBy("..interfaces..")
    .layer("Application").definedBy("..application..")
    .layer("Domain").definedBy("..domain..")
    .layer("Infrastructure").definedBy("..infrastructure..")
    
    .whereLayer("Interfaces").mayNotBeAccessedByAnyLayer()
    .whereLayer("Application").mayOnlyBeAccessedByLayers("Interfaces")
    .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Infrastructure")
    .whereLayer("Infrastructure").mayNotBeAccessedByAnyLayer(); // In strict mode
```

### 7.2 Module Isolation
Classes in an implementation module `com.example.moduleA..` MUST NOT depend on classes in `com.example.moduleB..` UNLESS the target class is in `com.example.moduleB.api..`.

### 7.3 Domain Purity
Classes residing in `..domain..` package:
- MUST NOT depend on `org.springframework..` (except `stereotype` or `transaction.annotation`).
- MUST NOT depend on `javax.persistence..` or `jakarta.persistence..` (if using strict DDD separation).
- MUST NOT depend on `..infrastructure..` or `..interfaces..`.

### 7.4 Naming & Location
- Classes annotated with `@RestController` MUST reside in `..interfaces.web..`.
- Classes annotated with `@Repository` or implementing `Repository` interfaces MUST reside in `..infrastructure.persistence..`.
- Classes ending with `DTO` MUST NOT reside in `..domain..`.
- Classes ending with `Entity` MUST reside in `..domain.model..` (or `infrastructure` depending on mapping strategy).
