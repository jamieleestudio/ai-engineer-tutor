---
rule_id: "java-ddd-clean-architecture-monolith-multi-module"
title: "Java DDD Clean Architecture (Monolith, Multi-Module)"
language: "java"
architecture_style: "DDD-Clean-Architecture"
layering:
  - "domain"
  - "application"
  - "interfaces"
  - "infrastructure"
modules:
  domain: "domain"
  application: "application"
  interfaces: "interfaces"
  infrastructure: "infrastructure"
package_prefix: "com.yourorg.yourapp"
package_patterns:
  domain: "com.yourorg.yourapp.domain.."
  application: "com.yourorg.yourapp.application.."
  interfaces: "com.yourorg.yourapp.interfaces.."
  infrastructure: "com.yourorg.yourapp.infrastructure.."
module_dependency_rules:
  allow:
    - { from: "application", to: "domain" }
    - { from: "interfaces", to: "application" }
    - { from: "interfaces", to: "domain" }
    - { from: "infrastructure", to: "application" }
    - { from: "infrastructure", to: "domain" }
  deny:
    - { from: "domain", to: "application|interfaces|infrastructure" }
    - { from: "application", to: "interfaces|infrastructure" }
    - { from: "interfaces", to: "infrastructure" }
    - { from: "any", to: "any", type: "cycle" }
purity_constraints:
  domain_disallow_types:
    - "javax.persistence.*"
    - "jakarta.persistence.*"
    - "org.springframework.*"
    - "java.sql.*"
  application_disallow_types:
    - "javax.persistence.*"
    - "jakarta.persistence.*"
    - "org.springframework.*"
    - "java.sql.*"
    - "http.client.*"
roles:
  controllers_module: "interfaces"
  repositories_impl_module: "infrastructure"
automation_examples:
  archunit_tests:
    - "DomainNoExternalDeps"
    - "ApplicationOnlyDependsOnDomain"
    - "InterfacesOnlyDependOnAppAndDomain"
    - "InfrastructureOnlyDependsOnAppAndDomain"
build_examples:
  maven:
    parent:
      artifactId: "yourapp-parent"
      modules: ["domain","application","interfaces","infrastructure"]
    dependencies:
      application: ["domain"]
      interfaces: ["application","domain"]
      infrastructure: ["application","domain"]
  gradle:
    settings_include: [":domain",":application",":interfaces",":infrastructure"]
    project_deps:
      application: [":domain"]
      interfaces: [":application",":domain"]
      infrastructure: [":application",":domain"]
test_policy:
  domain: "unit"
  application: "unit-mock-ports"
  interfaces: "slice-or-integration"
  infrastructure: "integration"
pr_checklist:
  - "模块间依赖满足允许/禁止规则"
  - "类位于正确包层"
  - "无反向依赖"
  - "无框架类型渗透到 domain/application"
  - "通过端口注入外部资源"
  - "补充匹配层级测试与架构测试"
  - "通过 ArchUnit 与构建依赖校验"
skill_prompt:
  input:
    - "changed_files"
    - "import_graph"
    - "module_graph"
  output:
    - "violations"
    - "fix_suggestions"
---

# Java DDD 清洁架构规约（单体、多模块）

**适用范围**
- 单体但拆分为多个模块（Maven/Gradle 子模块），以模块与包共同实现边界与依赖约束
- 目标：可维护、可测试、可演进，领域模型内聚与不变式清晰

**模块与分层**
- domain 模块：实体、值对象、聚合根、领域服务、领域事件、规格、工厂、仓储端口接口
- application 模块：应用服务（用例）、输入/输出 DTO、命令/查询、事件编排；依赖 domain
- interfaces 模块：入站适配（REST/CLI/消息）、控制器、映射；依赖 application 与 domain
- infrastructure 模块：出站实现（仓储/JPA/外部服务/消息）、配置、DI、启动、ACL；依赖 application 与 domain

**依赖边界（模块级与包级）**
- 允许：application → domain；interfaces → application|domain；infrastructure → application|domain
- 禁止：domain → 其他模块；application → interfaces|infrastructure；interfaces → infrastructure；任何循环依赖
- 包模式与单模块规约一致，确保内层纯净与外层实现分离

**端口位置与实现**
- 在 domain 或 application 定义端口（Repository、ExternalService、EventPublisher）
- 在 infrastructure 实现端口；interfaces 仅调用 application 用例

**Maven 最小示例**

```xml
<!-- parent/pom.xml -->
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.yourorg</groupId>
  <artifactId>yourapp-parent</artifactId>
  <packaging>pom</packaging>
  <modules>
    <module>domain</module>
    <module>application</module>
    <module>interfaces</module>
    <module>infrastructure</module>
  </modules>
</project>
```

```xml
<!-- application/pom.xml 只依赖 domain -->
<dependencies>
  <dependency>
    <groupId>com.yourorg</groupId>
    <artifactId>domain</artifactId>
    <version>${project.version}</version>
  </dependency>
</dependencies>
```

```xml
<!-- interfaces/pom.xml 依赖 application 与 domain -->
<dependencies>
  <dependency>
    <groupId>com.yourorg</groupId>
    <artifactId>application</artifactId>
    <version>${project.version}</version>
  </dependency>
  <dependency>
    <groupId>com.yourorg</groupId>
    <artifactId>domain</artifactId>
    <version>${project.version}</version>
  </dependency>
</dependencies>
```

```xml
<!-- infrastructure/pom.xml 依赖 application 与 domain -->
<dependencies>
  <dependency>
    <groupId>com.yourorg</groupId>
    <artifactId>application</artifactId>
    <version>${project.version}</version>
  </dependency>
  <dependency>
    <groupId>com.yourorg</groupId>
    <artifactId>domain</artifactId>
    <version>${project.version}</version>
  </dependency>
</dependencies>
```

**Gradle 最小示例**

```groovy
// settings.gradle
include ':domain', ':application', ':interfaces', ':infrastructure'
```

```groovy
// application/build.gradle
dependencies {
  implementation project(':domain')
}
```

```groovy
// interfaces/build.gradle
dependencies {
  implementation project(':application')
  implementation project(':domain')
}
```

```groovy
// infrastructure/build.gradle
dependencies {
  implementation project(':application')
  implementation project(':domain')
}
```

**测试策略**
- domain：纯单元测试
- application：Mock 端口的用例测试
- interfaces：控制器层切片/集成测试
- infrastructure：仓储/外部服务集成测试
- 架构测试：ArchUnit 独立套件并纳入 CI；模块依赖由构建脚本控制

**PR 检查清单**
- 模块依赖符合规约；无反向依赖或循环依赖
- 包层归属正确；无框架类型渗透到 domain/application
- 端口位置正确且在 infrastructure 实现；interfaces 仅调用 usecase
- 覆盖匹配层级的测试与架构测试

**Skills 审查提示模板**
- 输入：改动文件（changed_files）、导入依赖图（import_graph）、模块依赖图（module_graph）
- 输出：违反分层/依赖的项（violations），修复建议（fix_suggestions）

**ArchUnit 示例（包层约束，与单模块一致）**

```java
package com.yourorg.yourapp.arch;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes;

@AnalyzeClasses(packages = "com.yourorg.yourapp")
public class LayeredArchitectureRules {

    @ArchTest
    static final ArchRule domain_no_external_deps =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage("..application..", "..interfaces..", "..infrastructure..");

    @ArchTest
    static final ArchRule application_only_depends_on_domain =
        noClasses().that().resideInAPackage("..application..")
            .should().dependOnClassesThat().resideInAnyPackage("..interfaces..", "..infrastructure..");

    @ArchTest
    static final ArchRule interfaces_only_depends_on_app_and_domain =
        noClasses().that().resideInAPackage("..interfaces..")
            .should().dependOnClassesThat().resideInAnyPackage("..infrastructure..");

    @ArchTest
    static final ArchRule infrastructure_only_depends_on_app_and_domain =
        noClasses().that().resideInAPackage("..infrastructure..")
            .should().dependOnClassesThat().resideInAnyPackage("..interfaces..");

    @ArchTest
    static final ArchRule controllers_location =
        classes().that().haveSimpleNameEndingWith("Controller")
            .should().resideInAPackage("..interfaces.web.rest..");

    @ArchTest
    static final ArchRule repositories_impl_location =
        classes().that().haveSimpleNameEndingWith("RepositoryImpl")
            .should().resideInAPackage("..infrastructure.persistence.jpa..");
}
```

**参考与交叉链接**
- 模式综述参见 [patterns](../architecture/patterns/README.md) 与 [clean-architecture](../architecture/patterns/clean-architecture.md)

