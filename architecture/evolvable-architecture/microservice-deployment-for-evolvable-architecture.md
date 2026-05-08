# Skill: Microservice Deployment for Evolvable Architecture

## 1. Purpose

This skill defines **how to evolve a modular monolith architecture into microservices in a controlled, low-risk way**.
It is designed for teams that already have:

* Clear module boundaries
* Layered architecture (4-layer model)
* Multi-end (Admin / User / Mobile) support
* Independent deployment requirements

The goal is **evolution, not revolution**.

---

## 2. Core Philosophy

### 2.1 Architecture ≠ Deployment

* Architecture defines **code structure and dependencies**
* Deployment defines **runtime topology**

A system can be:

* Architecturally modular
* Operationally monolithic

This skill assumes **architecture-first**, deployment follows.

---

### 2.2 Microservices Are an Outcome, Not a Starting Point

Microservices emerge when:

* Modules are already independent
* Dependencies are explicit and directional
* APIs are stable and intentional

If these conditions are not met, splitting services increases complexity.

---

## 3. Baseline Architecture (Before Microservices)

### 3.1 Project Shape

```
root
 ├─ boot
 │   ├─ boot-all
 │   ├─ boot-admin
 │   ├─ boot-user
 │   └─ boot-mobile
 │
 ├─ order
 │   ├─ order-api
 │   ├─ order-application
 │   ├─ order-domain
 │   └─ order-infrastructure
 │
 ├─ payment
 └─ inventory
```

---

### 3.2 Layer Responsibilities

| Layer          | Responsibility                                        |
| -------------- | ----------------------------------------------------- |
| API            | Public contracts (DTOs, Facades, RPC/REST interfaces) |
| Application    | Use case orchestration, cross-module coordination     |
| Domain         | Core business rules and invariants                    |
| Infrastructure | DB, MQ, cache, third-party integrations               |

---

## 4. Module Collaboration Rules

### 4.1 Allowed Dependency Direction

```
Controller → Application → Domain
                      ↓
               Infrastructure
```

### 4.2 Cross-Module Calls

* Modules **never call another module's domain directly**
* All collaboration happens through:

```
OtherModule API → Application Service
```

This rule remains unchanged after microservice split.

---

## 5. Microservice Readiness Checklist

A module is **ready to become a microservice** when:

* [ ] Has a clear bounded context
* [ ] Exposes functionality only via API layer
* [ ] No database tables shared with other modules
* [ ] Cross-module calls already go through APIs
* [ ] Boot module can start it independently

If any item is missing → **do not split**.

---

## 6. Evolution Path

### Stage 1: Modular Monolith

* All modules deployed together
* Single database instance
* Multiple boot modules for different clients

> This is the default and recommended starting point.

---

### Stage 2: Deployment-Level Separation

* Selectively deploy some boot modules independently
* Same codebase, different runtime units
* Still shared database

Example:

* Mobile: 5 instances
* Admin: 1 instance

---

### Stage 3: Service Extraction

Steps:

1. Select one module (e.g. `order`)
2. Move its boot module under the business module
3. Replace in-process calls with HTTP / RPC
4. Keep API contracts unchanged

```
order-service
 ├─ order-boot
 ├─ order-api
 ├─ order-application
 ├─ order-domain
 └─ order-infrastructure
```

---

### Stage 4: Data Isolation

* Each microservice owns its database
* No cross-service joins
* Use events or APIs for consistency

This is the **true microservice boundary**.

---

## 7. Database Strategy During Evolution

### Early Stages

* Shared database instance is acceptable
* Separate schema per module
* Strict ownership rules

### Later Stages

* Database-per-service
* Event-driven synchronization
* Saga or eventual consistency patterns

---

## 8. API Layer in Microservices

### Responsibilities

* REST / RPC contracts
* DTOs
* Versioning
* Backward compatibility

### Rules

* API layer must remain stable during extraction
* Internal refactoring must not leak into APIs

---

## 9. Boot Module Evolution

### Before

* Centralized boot modules
* Boot decides which controllers to scan

### After

* Each microservice owns its own boot module
* Boot becomes the service boundary

Boot modules are **deployment units, not business logic containers**.

---

## 10. Common Anti-Patterns

❌ Splitting services before modularization
❌ Sharing domain models across services
❌ Letting infrastructure leak into APIs
❌ Designing APIs around UI instead of use cases

---

## 11. Summary

* Modular monolith is the foundation of microservices
* API-first design enables painless extraction
* Deployment flexibility comes before service splitting
* True microservices start with data ownership

> **If you can evolve gradually, you are doing it right.**
