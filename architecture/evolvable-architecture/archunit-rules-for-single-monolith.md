# ArchUnit Rules for Single Monolith Modules

## 1. Purpose

This document defines how to use **ArchUnit** to automatically verify the governance rules of a **single monolith with multiple modules**.

The goal is to turn architectural intent into executable tests so that:

- module boundaries do not silently erode
- layer responsibilities stay clear
- cross-module dependencies remain explicit
- the monolith stays evolvable as complexity grows

This document targets the **single-monolith stage** only. It does not cover microservice runtime governance.

---

## 2. Why ArchUnit Is Necessary

A modular monolith often starts clean and becomes coupled gradually through:

- direct calls to another module's internals
- controller bypass of application services
- domain leakage into infrastructure or web layers
- accidental component scanning and framework coupling
- test code depending on forbidden internals

Code review alone is not enough to prevent this drift.

ArchUnit gives the team:

- automated architecture verification
- fast feedback in CI
- explicit, versioned architecture rules
- safer refactoring across modules

---

## 3. Scope of Verification

For the single-monolith governance model, ArchUnit should verify at least:

1. layer dependency direction
2. domain purity
3. interfaces only calling application
4. cross-module access only through `api`
5. repositories and persistence staying inside the owning module
6. optional client adapter segregation such as `interfaces.web`, `interfaces.admin`, `interfaces.mobile`

ArchUnit is best used to enforce **structural rules**.

It does not replace:

- domain behavior tests
- integration tests
- transaction tests
- API contract tests

---

## 4. Recommended Package Convention

To make ArchUnit effective, package naming must be predictable.

Recommended package structure:

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
```

Optional client-specific adapters:

```text
..interfaces.web..
..interfaces.admin..
..interfaces.mobile..
```

ArchUnit rules become much simpler and more stable when package names are consistent.

---

## 5. Minimal Dependency Setup

Maven:

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

---

## 6. Base Test Class

A shared base class keeps rules consistent across the codebase.

```java
package com.example.architecture;

import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import com.tngtech.archunit.core.importer.ImportOption;

abstract class BaseArchUnitTest {

  protected static final String BASE_PACKAGE = "com.example";

  protected final JavaClasses classes = new ClassFileImporter()
      .withImportOption(ImportOption.Predefined.DO_NOT_INCLUDE_TESTS)
      .importPackages(BASE_PACKAGE);
}
```

If you also want to verify test code boundaries, create a second test class that imports test classes intentionally.

---

## 7. Core Rule Set

## 7.1 Layer Dependency Rules

These rules enforce the internal structure of each module.

```java
package com.example.architecture;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

import static com.tngtech.archunit.library.Architectures.layeredArchitecture;

@AnalyzeClasses(packages = "com.example")
class LayeredArchitectureTest {

  @ArchTest
  static final ArchRule layered_architecture_should_be_respected =
      layeredArchitecture()
          .consideringAllDependencies()
          .layer("Interfaces").definedBy("..interfaces..")
          .layer("Application").definedBy("..application..")
          .layer("Domain").definedBy("..domain..")
          .layer("Infrastructure").definedBy("..infrastructure..")
          .layer("Api").definedBy("..api..")

          .whereLayer("Interfaces").mayNotBeAccessedByAnyLayer()
          .whereLayer("Application").mayOnlyBeAccessedByLayers("Interfaces", "Infrastructure")
          .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Infrastructure")
          .whereLayer("Infrastructure").mayOnlyBeAccessedByLayers("Application")
          .whereLayer("Api").mayBeAccessedByAnyLayer();
}
```

Notes:

- `interfaces` is the outer entry layer
- `application` orchestrates use cases
- `domain` stays business-pure
- `api` is the public collaboration boundary
- `infrastructure` should not become a public integration surface

If your project treats `infrastructure` as implementing ports defined in `application` or `domain`, this rule is appropriate. If your implementation style differs, adjust the access rule but keep the dependency direction explicit.

---

## 7.2 Interfaces Must Not Bypass Application

Controllers, MQ consumers, and scheduled adapters must not call `domain` or `infrastructure` directly.

```java
package com.example.architecture;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

@AnalyzeClasses(packages = "com.example")
class InterfacesRulesTest {

  @ArchTest
  static final ArchRule interfaces_should_not_depend_on_domain_or_infrastructure =
      noClasses()
          .that().resideInAPackage("..interfaces..")
          .should().dependOnClassesThat()
          .resideInAnyPackage("..domain..", "..infrastructure..");
}
```

This is one of the most important rules in a modular monolith, because input adapters are a common place for architecture shortcuts.

---

## 7.3 Domain Must Stay Pure

The `domain` layer should not depend on framework or protocol details.

```java
package com.example.architecture;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

@AnalyzeClasses(packages = "com.example")
class DomainPurityTest {

  @ArchTest
  static final ArchRule domain_should_not_depend_on_frameworks =
      noClasses()
          .that().resideInAPackage("..domain..")
          .should().dependOnClassesThat()
          .resideInAnyPackage(
              "org.springframework..",
              "jakarta.persistence..",
              "jakarta.servlet..",
              "org.apache.kafka..");
}
```

You can tighten or relax this list depending on whether your domain allows validation annotations or selected shared abstractions.

---

## 7.4 Cross-Module Access Must Go Through API

This is the key rule for single-monolith module governance.

If `order` needs `payment`, it may depend on `payment.api`, but not on `payment.application`, `payment.domain`, `payment.infrastructure`, or `payment.interfaces`.

Example targeted rule:

```java
package com.example.architecture;

import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import org.junit.jupiter.api.Test;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

class CrossModuleIsolationTest {

  private final JavaClasses classes = new ClassFileImporter().importPackages("com.example");

  @Test
  void order_should_only_access_payment_api() {
    noClasses()
        .that().resideInAnyPackage("..order..")
        .should().dependOnClassesThat()
        .resideInAnyPackage(
            "..payment.application..",
            "..payment.domain..",
            "..payment.infrastructure..",
            "..payment.interfaces..")
        .check(classes);
  }
}
```

You can add one such rule per important module pair, especially for core domains.

---

## 7.5 Generic Rule for All Modules

If your package naming is regular enough, you can encode a broader rule using slices.

```java
package com.example.architecture;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

import static com.tngtech.archunit.library.dependencies.SlicesRuleDefinition.slices;

@AnalyzeClasses(packages = "com.example")
class ModuleSlicesTest {

  @ArchTest
  static final ArchRule modules_should_be_free_of_cycles =
      slices().matching("com.example.(*)..")
          .should().beFreeOfCycles();
}
```

This rule does not replace explicit `api`-only access checks, but it is valuable because cyclic dependencies are one of the earliest signals of module erosion.

---

## 7.6 Repositories Must Stay Inside Their Module

Persistence types should not be reused across module boundaries.

```java
package com.example.architecture;

import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import org.junit.jupiter.api.Test;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

class RepositoryIsolationTest {

  private final JavaClasses classes = new ClassFileImporter().importPackages("com.example");

  @Test
  void order_repositories_should_not_be_used_outside_order() {
    noClasses()
        .that().resideOutsideOfPackage("..order..")
        .should().dependOnClassesThat()
        .haveSimpleNameEndingWith("OrderRepository")
        .check(classes);
  }
}
```

In real projects, prefer package-based targeting over class-name-only targeting when possible.

---

## 7.7 Controllers Should Only Talk to Application

If your team uses naming conventions such as `*Controller`, enforce them.

```java
package com.example.architecture;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes;

@AnalyzeClasses(packages = "com.example")
class ControllerRulesTest {

  @ArchTest
  static final ArchRule controllers_should_reside_in_interfaces =
      classes()
          .that().haveSimpleNameEndingWith("Controller")
          .should().resideInAPackage("..interfaces..");
}
```

You can combine this with a second rule that bans controllers from depending on repositories or infrastructure directly.

---

## 7.8 Optional Rule for Multi-Client Adapters

If your system supports `web`, `admin`, and `mobile` input adapters, keep them isolated from each other unless there is a deliberate shared adapter package.

```java
package com.example.architecture;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

@AnalyzeClasses(packages = "com.example")
class ClientAdapterIsolationTest {

  @ArchTest
  static final ArchRule web_should_not_depend_on_admin_or_mobile =
      noClasses()
          .that().resideInAPackage("..interfaces.web..")
          .should().dependOnClassesThat()
          .resideInAnyPackage("..interfaces.admin..", "..interfaces.mobile..");
}
```

This helps keep client-specific transport logic from blending together.

---

## 8. Suggested Test Layout

Recommended package:

```text
src/test/java/com/example/architecture
├─ BaseArchUnitTest.java
├─ LayeredArchitectureTest.java
├─ DomainPurityTest.java
├─ InterfacesRulesTest.java
├─ CrossModuleIsolationTest.java
├─ ModuleSlicesTest.java
└─ ControllerRulesTest.java
```

Keep the tests small and focused. Do not put every rule into one huge class.

---

## 9. Rule Strategy by Maturity

Start with a minimal rule set:

1. interfaces must not depend on domain/infrastructure
2. domain must stay framework-free
3. modules must be cycle-free
4. controllers must stay in interfaces

Then add stricter rules:

5. cross-module access only through `api`
6. repository isolation
7. client adapter isolation

This phased approach helps teams adopt architecture tests without creating too much initial friction.

---

## 10. Common Pitfalls

### 10.1 Overly Broad Base Package

If the imported package includes generated code, legacy modules, or external adapters with different rules, tests become noisy and lose credibility.

Prefer:

- targeted package import
- per-module exclusions when necessary
- separate rule classes for legacy areas

---

### 10.2 Overfitting Rules to Current Implementation

Do not encode temporary class names or implementation details into architecture rules unless they reflect stable conventions.

Prefer:

- package boundaries
- suffix conventions used across the project
- explicit ownership rules

---

### 10.3 Treating ArchUnit as Complete Coverage

ArchUnit can prove structural intent, but it cannot prove:

- correct transaction behavior
- business correctness
- runtime wiring under all environments
- SQL ownership discipline unless encoded through packages and naming

Use it as an architectural safety net, not a full testing strategy.

---

## 11. Recommended CI Policy

Run ArchUnit tests:

- on every pull request
- on the main branch build
- before merging large refactors

Recommended policy:

- architecture test failure blocks merge
- exceptions require explicit review and documented reason
- temporary waivers must be time-bounded

If architecture rules are optional, they will slowly stop being effective.

---

## 12. Mapping to Governance Rules

This document supports the governance model defined in [single-monolith-module-governance.md](file:///c:/Users/lixiaofeng/IdeaProjects/ai-engineer-tutor/architecture/evolvable-architecture/single-monolith-module-governance.md).

Suggested mapping:

- module boundary -> cross-module access only through `api`
- layer boundary -> layered architecture rules
- domain purity -> framework dependency ban
- input adapter discipline -> interfaces and controller rules
- module independence -> cycle detection
- client segregation -> adapter isolation rules

---

## 13. Summary

For a single monolith with multiple modules, ArchUnit should protect the architecture from silent erosion.

The most important automated guarantees are:

- layers depend in the correct direction
- domain stays clean
- controllers and adapters do not bypass application
- modules do not reach into each other's internals
- cycles are detected early

> **If the architecture matters, it must be executable.**
