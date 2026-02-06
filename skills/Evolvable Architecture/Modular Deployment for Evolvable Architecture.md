# Skill: Modular Deployment for Evolvable Architecture

## Purpose

This document defines how **modular deployment** works under the Evolvable Architecture model.

The goal is to support **three deployment modes with the same codebase**:

1. Monolithic (all modules in one JVM)
2. Single-module deployment (independent module runtime)
3. Microservice / RPC-based deployment

The architecture guarantees:
- Stable module boundaries
- Replaceable implementations
- No refactoring when evolving deployment topology

---

## Core Principles

### 1. Modules Are Deployable Units, Not Applications

A business module **is not** a Spring Boot application by itself.

- ❌ Each module owning `@SpringBootApplication`
- ❌ Multiple application entry points inside business modules

Instead:

- ✅ A module is a **Spring-loadable component set**
- ✅ Deployment is controlled by **boot applications**
- ✅ Modules expose **starters** for assembly

---

### 3. Logical Boundaries First, Physical Boundaries Later (Anti-Module Explosion)

Start with **logical modules** (packages) within a single Maven/Gradle module. Only split into **physical modules** (separate JARs) when:
- The module needs to be deployed independently.
- The module needs to be shared across multiple different projects.
- The build time becomes unmanageable.
- Strict boundary enforcement is required (though tools like ArchUnit can enforce package boundaries too).

**Evolution Path:**
1. `com.example.monolith.order` (Package)
2. `order-module.jar` (Single Jar containing all layers)
3. `order-api`, `order-core` (Split Interface/Implementation)
4. Full 5-module split (Only for complex, core domains)

---

## Project Structure (Deployment-Oriented)

```text
evolvable-architecture
├─ boot
│  ├─ boot-monolith        # All modules deployed together
│  ├─ boot-order           # Order module standalone
│  ├─ boot-process         # Process module standalone
│
├─ modules
│  ├─ order
│  │  ├─ order-api
│  │  ├─ order-application
│  │  ├─ order-domain
│  │  ├─ order-infrastructure
│  │  ├─ order-interfaces
│  │  └─ order-starter     # Module assembly entry
│  │
│  ├─ process
│  │  ├─ process-api
│  │  ├─ process-application
│  │  ├─ process-domain
│  │  ├─ process-infrastructure
│  │  ├─ process-interfaces
│  │  └─ process-starter
│
└─ pom.xml
