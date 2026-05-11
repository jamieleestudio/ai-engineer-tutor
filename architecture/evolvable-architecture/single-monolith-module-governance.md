# Single Monolith Module Governance

## 1. Purpose

This document defines the governance rules for the **initial architecture stage: a single monolith with multiple logical or physical modules**.

The focus is not microservice governance. The goal is to keep the monolith:

- modular
- evolvable
- easy to refactor
- safe to scale in complexity

This document applies **before** service extraction, distributed transactions, service discovery, or other microservice concerns are introduced.

---

## 2. Scope

This governance model assumes:

- one codebase
- one runtime unit by default
- one JVM process by default
- one database instance by default
- multiple business modules with clear ownership

The architecture target is:

```text
interfaces -> application -> domain
                     |
                     -> infrastructure
```

Cross-module collaboration must remain explicit and controlled, even though everything is still deployed as one monolith.

---

## 3. Core Principles

### 3.1 Monolith Does Not Mean No Boundaries

Running in one process does not justify direct access across modules.

Each module must preserve:

- clear responsibilities
- explicit dependencies
- isolated data ownership
- stable contracts

---

### 3.2 Logical Boundaries Come First

Modules may start as package-level boundaries or as separate build modules.

The key requirement is not the packaging shape, but whether the team can clearly answer:

- what this module owns
- what this module exposes
- what this module must not access directly

---

### 3.3 Business Collaboration Must Be Explicit

When one module needs another module's capability, it must call through the target module's **api** contract.

Allowed collaboration:

```text
order.application -> payment.api
```

Forbidden collaboration:

```text
order.domain -> payment.domain
order.interfaces -> payment.infrastructure
order.application -> payment.repository
```

---

## 4. Module Structure

Each business module should follow the same internal structure:

```text
order
├─ api
├─ application
├─ domain
├─ infrastructure
└─ interfaces
```

Layer responsibilities:

| Layer | Responsibility | May Depend On | Must Not Depend On |
|-------|----------------|---------------|--------------------|
| `api` | Public contract: DTOs, commands, queries, facades | none | Spring MVC, persistence, business implementation |
| `application` | Use case orchestration, transaction boundary, cross-module coordination | current module `domain`, current module `infrastructure`, other module `api` | other module `domain`, Controller, repository of another module |
| `domain` | Core business model, invariants, domain services | none | Spring, persistence, HTTP, MQ, other module |
| `infrastructure` | Persistence, third-party clients, technical adapters | current module `domain`, current module `application` | other module internals |
| `interfaces` | Controllers, MQ consumers, scheduled triggers, protocol mapping | current module `application`, current module `api` | `domain`, `infrastructure`, other module internals |

---

## 5. Cross-Module Dependency Rules

### 5.1 Mandatory Rule

**A module may only depend on another module through that module's `api` layer.**

Example:

```text
order -> payment.api
inventory -> order.api
```

Not allowed:

```text
order -> payment.application
order -> payment.domain
order -> payment.infrastructure
```

This rule keeps module collaboration explicit and makes later extraction possible without redesigning internal dependencies.

---

### 5.2 Cross-Module Orchestration Belongs to Application

If a use case touches multiple modules, orchestration belongs to the initiating module's `application` layer.

Example:

- `order.application` creates an order
- then calls `payment.api` to reserve payment
- then calls `inventory.api` to reserve stock

This orchestration must not appear in:

- Controller
- Domain entity
- Repository
- Infrastructure adapter

---

### 5.3 Domain Must Stay Local

The `domain` layer is always module-private in a monolith.

A domain model may not:

- import another module's domain type
- invoke another module's domain service
- access another module's repository

If cross-module interaction is required, expose an `api` contract instead.

---

## 6. Data Ownership Rules

### 6.1 Every Table Has an Owner Module

Even in a shared database, every table must have exactly one owner module.

The owner module is responsible for:

- schema evolution
- writes
- invariants
- data cleanup rules

Other modules may read through:

- the owner module's `api`
- a dedicated query interface
- approved read-only reporting access if explicitly defined

Other modules must not write another module's tables directly.

---

### 6.2 Avoid Cross-Module Direct Table Access

Forbidden by default:

- `order` repository reading `payment_*` tables directly
- `inventory` repository updating `order_*` tables directly
- cross-module foreign keys that enforce runtime coupling

If reporting requires multi-module joins, do it through:

- application-level composition
- dedicated query services
- reporting views explicitly marked as read-only

Do not use repository shortcuts to break ownership.

---

### 6.3 Shared Database Is an Infrastructure Detail, Not a Modeling Shortcut

Using one database instance does not mean modules share one data model.

The team should still maintain:

- table ownership
- schema naming conventions
- migration ownership
- clear write boundaries

---

## 7. Transaction Rules

### 7.1 Transaction Boundary Lives in Application

The `application` layer defines transaction boundaries.

Recommended rule:

- one use case
- one application service method
- one clear transaction boundary

Controllers must not manage transactions directly.

---

### 7.2 Avoid Large Cross-Module Transactions

Because this is still a monolith, a single transaction can technically update multiple modules.

That should be treated as a controlled exception, not the default design.

Use cross-module transactions only when:

- the use case is truly atomic
- the modules are part of the same consistency boundary
- separating the operation would make the model incorrect

Do not use a large transaction only because it is convenient.

---

### 7.3 Prefer Local Invariants Over Global Coupling

If multiple modules always change together, re-evaluate whether they are actually separate modules.

Frequent cross-module atomic updates usually indicate:

- an incorrect boundary
- a missing shared capability
- an orchestration model that is too coupled

---

## 8. Shared Code Governance

### 8.1 `shared` Is a Last Resort

The `shared` module must remain small and technical.

Allowed in `shared`:

- base technical utilities
- common configuration helpers
- generic value objects with no business ownership
- common testing utilities

Not allowed in `shared`:

- business rules
- module-specific DTOs
- domain services
- cross-module orchestration logic
- anything created only to avoid choosing proper ownership

---

### 8.2 Promote Reuse Carefully

Before moving code into `shared`, verify:

1. the code is truly generic
2. it has no hidden business meaning
3. at least two modules need it for the same reason

If not, keep it in the owning module.

---

## 9. Boot and Assembly Rules

### 9.1 Boot Modules Assemble, Business Modules Do Not Start Themselves

Business modules must not contain their own `@SpringBootApplication`.

Boot modules are responsible for:

- application startup
- environment selection
- module assembly
- client-specific interface scanning

Business modules are responsible for:

- exposing Spring-loadable configuration
- registering required beans
- keeping business code independent from deployment topology

---

### 9.2 Interfaces Are Selected by Client Type

For web, admin, and mobile clients, the boot layer decides which input adapters are enabled.

Example:

```text
boot-web    -> scans ..interfaces.web..
boot-admin  -> scans ..interfaces.admin..
boot-mobile -> scans ..interfaces.mobile..
```

Business logic must stay unchanged regardless of which client boot is selected.

---

### 9.3 No Hidden Bean Coupling Across Modules

Do not rely on accidental component scanning to wire module internals together.

Prefer:

- explicit configuration classes
- starter-style assembly
- constructor injection through public contracts

Avoid:

- direct injection of another module's internal service
- scanning broad packages that bypass module boundaries

---

## 10. API Contract Rules

### 10.1 API Is the Module Boundary

The `api` layer defines what other modules are allowed to know.

It may contain:

- facades
- commands
- queries
- response DTOs
- domain-neutral identifiers

It must not leak:

- JPA entities
- repositories
- controller models
- infrastructure types

---

### 10.2 API Should Be Stable and Minimal

Expose only what other modules truly need.

Do not turn `api` into a mirror of internal application services.

Good `api` design:

- use-case oriented
- small surface area
- explicit naming
- independent from transport protocol

---

## 11. Error Handling Rules

### 11.1 Business Errors Must Be Predictable

Modules should expose clear business failures through documented exception types or result models.

Examples:

- order already cancelled
- payment already completed
- stock not sufficient

Avoid leaking low-level technical exceptions across module boundaries.

---

### 11.2 Interfaces Translate Protocol Concerns

The `interfaces` layer is responsible for mapping:

- HTTP status codes
- request validation errors
- protocol-specific error payloads

Application and domain layers should not depend on HTTP semantics.

---

## 12. Testing Rules

### 12.1 Architecture Tests Are Mandatory

Use ArchUnit or equivalent rules to verify:

- layer boundaries
- cross-module dependency rules
- domain purity
- forbidden imports

---

### 12.2 Behavior Tests Must Follow Module Boundaries

Each module should have:

- application service tests
- domain behavior tests
- API contract tests where useful

Cross-module scenarios should be tested at the application level, not by bypassing module contracts.

---

### 12.3 Do Not Test Through Internals of Other Modules

If module `order` depends on `payment`, tests in `order` should use `payment.api` or test doubles based on `payment.api`.

Tests must not depend on:

- `payment.domain`
- `payment.repository`
- `payment` internal fixtures

Otherwise test code will silently break the architecture.

---

## 13. Recommended Enforcement

The following rules should be automated:

- `..interfaces..` must not depend on `..domain..` or `..infrastructure..`
- `..domain..` must not depend on Spring or persistence APIs
- one module must not depend on another module's non-`api` packages
- controllers must only call application services
- repositories must remain inside the owning module

Architecture governance should be checked in CI, not only in code review.

---

## 14. Common Anti-Patterns

Avoid these patterns in a single monolith with multiple modules:

- treating package structure as documentation only
- letting controllers orchestrate multiple modules
- exposing repositories across modules
- putting business logic into `shared`
- using shared database tables as an excuse for shared ownership
- injecting another module's internal Spring service directly
- creating cross-module utility shortcuts that bypass `api`

---

## 15. Summary

For the initial monolith stage, the most important governance goal is simple:

> **Keep module boundaries real before making deployment boundaries possible.**

If the team can preserve:

- explicit module APIs
- local domain models
- clear transaction ownership
- strict data ownership
- disciplined shared code usage

then the monolith will remain maintainable and evolvable without prematurely adopting microservice complexity.
