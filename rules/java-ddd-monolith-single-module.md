---
rule_id: "java-ddd-clean-architecture-iadi"
title: "Java DDD Clean Architecture (Monolith, Single Module)"
language: "java"
architecture_style: "DDD-Clean-Architecture"
layering:
  - "domain"
  - "application"
  - "interfaces"
  - "infrastructure"
package_prefix: "com.yourorg.yourapp"
package_patterns:
  domain: "com.yourorg.yourapp.domain.."
  application: "com.yourorg.yourapp.application.."
  interfaces: "com.yourorg.yourapp.interfaces.."
  infrastructure: "com.yourorg.yourapp.infrastructure.."
dependency_rules:
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
  controllers_package: "com.yourorg.yourapp.interfaces.web.rest.."
  repositories_impl_package: "com.yourorg.yourapp.infrastructure.persistence.jpa.."
automation_examples:
  archunit_tests:
    - "DomainNoExternalDeps"
    - "ApplicationOnlyDependsOnDomain"
    - "InterfacesOnlyDependOnAppAndDomain"
    - "InfrastructureOnlyDependsOnAppAndDomain"
test_policy:
  domain: "unit"
  application: "unit-mock-ports"
  interfaces: "slice-or-integration"
  infrastructure: "integration"
pr_checklist:
  - "类位于正确包层"
  - "无反向依赖"
  - "无框架类型渗透到 domain/application"
  - "通过端口注入外部资源"
  - "补充匹配层级测试"
  - "通过 ArchUnit"
skill_prompt:
  input:
    - "changed_files"
    - "import_graph"
  output:
    - "violations"
    - "fix_suggestions"
---

# Java DDD 清洁架构规约（单体、单模块）

**适用范围**
- 单体、单模块 Java 项目，以包分层实现边界与依赖约束
- 目标：可维护、可测试、可演进，领域模型内聚与不变式清晰

**分层语义**
- domain：实体、值对象、聚合根、领域服务、领域事件、规格、工厂；不依赖框架与 I/O
- application：应用服务（用例）、输入/输出 DTO、命令/查询、事件编排；仅依赖 domain，通过端口访问外部
- interfaces：入站适配（REST/CLI/消息）、控制器、映射；调用 application，用以把 I/O 映射为用例输入
- infrastructure：出站实现（仓储/外部服务/消息）、配置、依赖注入、启动、ACL 防腐层；实现端口并提供技术细节

**依赖边界**
- 允许：application → domain；interfaces → application|domain；infrastructure → application|domain
- 禁止：domain → application|interfaces|infrastructure；application → interfaces|infrastructure；interfaces → infrastructure；任何循环依赖

**端口位置**
- 在 domain 或 application 定义端口（Repository、ExternalService、EventPublisher）
- 在 infrastructure 实现端口；interfaces 仅调用 application 用例

**包结构示例**
- com.yourorg.yourapp.domain.model、domain.value、domain.aggregate、domain.service、domain.event、domain.spec、domain.factory、domain.repository
- com.yourorg.yourapp.application.usecase、application.dto、application.command、application.query、application.event
- com.yourorg.yourapp.interfaces.web.rest、interfaces.cli、interfaces.messaging、interfaces.mapping
- com.yourorg.yourapp.infrastructure.persistence.jpa、infrastructure.external.http、infrastructure.messaging、infrastructure.config、infrastructure.di、infrastructure.boot、infrastructure.acl

**测试策略**
- domain：纯单元测试
- application：Mock 端口的用例测试
- interfaces：控制器层切片/集成测试
- infrastructure：仓储/外部服务集成测试
- 架构测试：ArchUnit 独立套件并纳入 CI

**PR 检查清单**
- 新增类位于正确包层，未出现反向依赖或框架类型渗透到 domain/application
- 端口在 domain/application 定义，infrastructure 中实现；interfaces 仅调用 usecase
- 补充匹配层级测试与 ArchUnit 约束

**Skills 审查提示模板**
- 输入：改动文件列表（changed_files）、导入依赖图（import_graph）
- 输出：违反分层/依赖的文件与原因（violations），修复建议（fix_suggestions）

**ArchUnit 最小示例**

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

