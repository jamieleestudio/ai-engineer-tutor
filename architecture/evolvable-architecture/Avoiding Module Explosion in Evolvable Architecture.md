# Skill: Avoiding Module Explosion in Evolvable Architecture

## 1. Purpose

Prevent "module explosion" by starting with package-level logical modules and evolving to physical modules only when needed. Enforce boundaries with architecture tests (ArchUnit) so the team can move fast without drifting.

---

## 2. Problem Statement

Splitting every domain into five physical modules (api, application, domain, infrastructure, interfaces) up front causes:
- Excessive build and dependency management
- Slower refactors and higher cognitive load
- Premature physical boundaries before logical ones are stable

The solution is to adopt **package-first logical boundaries** and evolve to physical modules **on demand**.

---

## 3. Package-First Strategy

Keep a single Maven/Gradle module, structure logical modules by packages:

```text
com.example
├─ order
│  ├─ api
│  ├─ application
│  ├─ domain
│  ├─ infrastructure
│  └─ interfaces
├─ payment
│  ├─ api
│  ├─ application
│  ├─ domain
│  ├─ infrastructure
│  └─ interfaces
└─ shared
   └─ ...
```

Principles:
- Controllers, MQ listeners, RPC providers stay under `..interfaces..`
- Use cases and orchestration under `..application..`
- Core business rules in `..domain..`
- DB/cache/clients in `..infrastructure..`
- Contracts (DTOs, commands, queries, facades) in `..api..`

Boot applications decide which client-specific interfaces to scan (web/admin/mobile), while business layers remain shared.

---

## 4. When to Extract Physical Modules

Evolve a package into separate JARs only when at least one is true:
- Needs independent deployment/scaling (runtime boundary)
- Needs reuse across different projects (artifact boundary)
- Build time and dependency graph become a bottleneck (operational boundary)
- Team ownership or compliance requires strict isolation (organizational boundary)

Evolution path example:
1. `com.example.order` (package)
2. `order-module.jar` (single JAR containing all layers)
3. `order-api` + `order-core` (interface/implementation split)
4. Full 5-module split for complex, core domains only

---

## 5. ArchUnit Architecture Tests

Use ArchUnit to enforce boundaries at the package level. This keeps the architecture intentional even before physical module split.

### 5.1 Dependency Direction Rules

```java
package com.example.architecture;

import com.tngtech.archunit.core.importer.ClassFileImporter;
import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.lang.ArchRule;
import static com.tngtech.archunit.library.Architectures.layeredArchitecture;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

class ArchitectureRulesTest {
  private final JavaClasses classes = new ClassFileImporter()
      .importPackages("com.example");

  ArchRule layered = layeredArchitecture()
      .layer("Interfaces").definedBy("..interfaces..")
      .layer("Application").definedBy("..application..")
      .layer("Domain").definedBy("..domain..")
      .layer("Infrastructure").definedBy("..infrastructure..")
      // Access rules (who may access a layer)
      .whereLayer("Application").mayOnlyBeAccessedByLayers("Interfaces")
      .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application")
      .whereLayer("Infrastructure").mayOnlyBeAccessedByLayers("Application");

  @org.junit.jupiter.api.Test
  void layered_dependencies_are_respected() {
    layered.check(classes);
  }

  @org.junit.jupiter.api.Test
  void interfaces_do_not_depend_on_domain_or_infrastructure_directly() {
    noClasses()
        .that().resideInAPackage("..interfaces..")
        .should().dependOnClassesThat().resideInAnyPackage("..domain..", "..infrastructure..")
        .check(classes);
  }

  @org.junit.jupiter.api.Test
  void domain_is_pure_business_without_framework_dependencies() {
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAnyPackage("org.springframework..", "jakarta..")
        .check(classes);
  }
}
```

### 5.2 Cross-Module Isolation (Package-Based)

Block direct domain-to-domain dependencies across logical modules:

```java
@org.junit.jupiter.api.Test
void order_domain_should_not_depend_on_payment_domain() {
  JavaClasses classes = new ClassFileImporter().importPackages("com.example");
  noClasses()
      .that().resideInAPackage("..order.domain..")
      .should().dependOnClassesThat().resideInAPackage("..payment.domain..")
      .check(classes);
}
```

These rules preserve boundaries even when everything lives in one physical module.

---

## 6. Boot and Client Scanning

Client-specific boot apps include only the corresponding interfaces:

```java
@SpringBootApplication
@ComponentScan(basePackages = {
  "com.example.order.interfaces.web",
  "com.example.payment.interfaces.web",
  // shared configuration packages
  "com.example.shared.config"
})
class WebBoot {}
```

Application/domain/infrastructure are on the classpath and discovered by dependency graph, but input adapters are selected per client (web/admin/mobile).

---

## 7. Benefits

- Avoids premature fragmentation and dependency sprawl
- Keeps boundaries enforceable via tests
- Enables gradual, low-risk evolution to true modules/microservices
- Aligns with API-first and clean layering principles

---

## 8. Minimal Setup

Maven (test scope):

```xml
<dependency>
  <groupId>com.tngtech.archunit</groupId>
  <artifactId>archunit-junit5</artifactId>
  <version>1.2.1</version>
  <scope>test</scope>
</dependency>
```

Gradle:

```groovy
testImplementation "com.tngtech.archunit:archunit-junit5:1.2.1"
```

Run `mvn test` or `gradle test` to validate architectural rules.

---

## 9. Summary

Start with package-level logical modules. Extract physical modules only when runtime, reuse, or organizational needs demand it. Use ArchUnit to codify the architecture so evolution remains intentional and safe.
