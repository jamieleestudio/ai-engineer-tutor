# Evolvable Modular Monolith Architecture Rules

此文档定义了本项目的架构设计原则、分包结构与演进策略。AI 助手在进行代码生成、架构分析与重构时，**必须严格遵守**以下规则。

## 1. 核心设计理念 (Core Philosophy)

*   **物理隔离契约**：模块间通信必须通过独立的 API 模块（Maven Module）进行，严禁直接依赖实现模块。
*   **单向依赖**：依赖关系只能由上层指向下层，由实现指向契约。
*   **可演进性**：默认作为单体部署，但必须保持随时可拆分为微服务的结构就绪度（Microservices-Ready）。
*   **实事求是**：API 模块按需创建。只有当模块需要被其他后端模块依赖时，才提取 API 模块；否则保持单模块结构。

## 2. 物理工程结构 (Maven Modules)

对于一个典型的业务域（如 `user`），物理结构如下：

### 2.1 API 模块 (`xxx-api`)
*   **定位**：外交契约 (External Contract)。
*   **职责**：存放提供给**其他业务模块**调用的接口定义。
*   **包含内容**：
    *   `Service Interfaces` (RPC 风格，如 `UserReadService`)
    *   `DTOs` (Request/Response POJOs, 实现 `Serializable`)
    *   `Enums` (跨模块共享的状态枚举)
    *   `Events` (跨模块集成事件)
*   **禁止内容**：
    *   **严禁**包含任何业务逻辑实现。
    *   **严禁**包含 Spring Context / Persistence 等重量级依赖。
    *   **严禁**包含 Web 层相关对象 (如 `HttpServletRequest`)。

### 2.2 实现模块 (`xxx`)
*   **定位**：业务实现 (Implementation)。
*   **职责**：包含所有业务逻辑、数据持久化及对外暴露的适配器。
*   **依赖规则**：
    *   若需要被外部调用：`implementation project(':xxx-api')`
    *   若需要调用其他模块：`implementation project(':other-module-api')`
    *   **严禁**依赖其他模块的实现模块 (e.g., `user` 依赖 `order` 是绝对禁止的)。

## 3. 逻辑分层架构 (Logical Layers)

在实现模块 (`xxx`) 内部，严格遵循以下 **4 层架构**：

```text
com.example.module
├── interfaces              <-- [适配层/入口层]
│   ├── web                 <-- HTTP Rest Controller (给前端/App)
│   ├── facade              <-- RPC Service Implementation (给其他模块，实现 xxx-api)
│   └── listener            <-- MQ Consumer (给消息中间件)
├── application             <-- [应用层/编排层]
│   └── service             <-- 业务流程编排 (Transaction script, DTO <-> Entity 转换)
├── domain                  <-- [领域层/核心层]
│   ├── model               <-- Entity, Value Object, Aggregate Root
│   ├── service             <-- Domain Service (核心业务规则)
│   └── repository          <-- Repository Interface (仓储契约)
└── infrastructure          <-- [基础设施层]
    ├── persistence         <-- Repository Implementation (MyBatis/JPA)
    ├── client              <-- Feign Client / RPC Client (外部服务调用)
    └── config              <-- Configuration Classes
```

### 3.1 层级依赖规则
1.  **Interfaces** -> 依赖 -> **Application**
2.  **Application** -> 依赖 -> **Domain**
3.  **Infrastructure** -> 依赖 -> **Domain** (DIP: 依赖倒置)
    *   *注：在工程实践中，Infrastructure 也可以被 Application 直接使用（如工具类），但 Domain 层严禁依赖 Infrastructure。*

## 4. API 设计规范 (API Design Guidelines)

### 4.1 模块间接口 (RPC Contract)
*   **位置**：`xxx-api` 模块。
*   **命名**：`XxxService` (e.g., `UserReadService`, `OrderCreateService`)。
*   **方法签名**：
    *   入参：必须封装为 `XxxRequest` 对象 (支持向后兼容)。
    *   出参：必须封装为 `XxxDTO` 对象 (贫血模型，不暴露 Entity)。
    *   异常：不抛出受检异常，使用 Result 包装或运行时异常。
*   **原则**：
    *   **最小权限**：只暴露对方需要的方法。
    *   **读写分离**：建议将查询接口 (`ReadService`) 和命令接口 (`WriteService`) 分离。

### 4.2 前端接口 (Web API)
*   **位置**：`xxx` 模块 -> `interfaces.web` 包。
*   **形式**：`@RestController`。
*   **原则**：BFF (Backend for Frontend) 模式，专注于视图适配，不应直接暴露 RPC 接口。

### 4.3 单模块命名对齐 (Single-Module Alignment)
*   当业务域尚未拆分 `xxx-api` 模块时，仍按“边界用途”选择对象后缀，以保证未来抽取契约模块时命名稳定：
    *   Web I/O：`XxxRequest` / `XxxResponse`（或 `XxxView`）
    *   模块间契约（未来进入 `xxx-api`）：`XxxRequest` / `XxxDTO`
    *   用例输入/输出（应用层内部）：`XxxCommand` / `XxxQuery` / `XxxResult`
*   约束：`*Request/*Response/*DTO` 不进入 Domain；Domain 只保留业务语言（Entity/VO/DomainEvent）。

## 5. 演进与部署策略

### 5.1 单体模式 (当前状态)
*   **部署**：所有 `xxx` 模块打包在一个 Spring Boot Application (Server) 中。
*   **调用**：`OrderService` 注入 `UserReadService` 时，Spring 容器直接注入 `UserReadServiceFacade` 的本地实例。
*   **性能**：本地方法调用，零网络开销。

### 5.2 微服务模式 (未来演进)
*   **部署**：`user` 模块独立打包部署。
*   **调用**：
    1.  `order` 模块保持依赖 `user-api` 不变。
    2.  `order` 模块配置 RPC 框架 (Dubbo/Feign) 代理 `UserReadService`。
    3.  `user` 模块配置 RPC 框架暴露 `UserReadServiceFacade`。
*   **代码变更**：业务逻辑层 (Application/Domain) **零修改**。

## 6. ArchUnit 守护规则 (Architecture Guard)

所有 Java 代码必须通过以下架构测试：

1.  **物理边界**：`xxx` 模块不能访问其他 `yyy` 模块的包 (除了 `yyy.api`)。
2.  **分层边界**：
    *   `Domain` 层不能依赖 `Application`, `Interfaces`, `Infrastructure`。
    *   `Interfaces` 层只能被外部框架调用，不能被内部层调用。
3.  **API 实现约束**：
    *   实现 `xxx-api` 接口的类，必须位于 `interfaces.facade` 包下。

---
**Summary for AI**: When generating code or refactoring, ALWAYS check if a separate `api` module is needed. If yes, place interfaces/DTOs there. If no, keep it simple but strictly follow the 4-layer structure inside the module.
