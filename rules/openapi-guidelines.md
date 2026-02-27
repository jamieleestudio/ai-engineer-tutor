---
rule_id: "openapi-api-guidelines"
title: "OpenAPI & REST 接口规范"
language: "java"
framework: "spring-boot"
tags:
  - "openapi"
  - "rest-api"
  - "swagger"
  - "frontend-backend-contract"
---

# OpenAPI & REST 接口规范

本文档旨在建立前后端统一的接口契约标准。基于 OpenAPI 3.0 (SpringDoc) 规范，确保后端输出高质量的接口文档，前端能够基于文档进行无缝对接与代码生成。

## 1. 核心原则 (Core Principles)

- **契约优先 (Contract First)**: 无论是先写代码还是先写 YAML，接口文档即契约。变更必须同步更新文档。
- **RESTful 风格**: 遵循资源导向设计，合理使用 HTTP 方法与状态码。
- **语义清晰**: URL 表达资源，Body 表达数据，Header 表达元数据。
- **统一包装**: 所有 API 响应（除文件下载外）应使用统一的信封结构。

## 2. 技术栈约定 (Tech Stack)

- **规范版本**: OpenAPI 3.0+
- **后端实现**: Spring Boot 3 + `springdoc-openapi-starter-webmvc-ui`
- **前端对接**: 推荐使用 OpenAPI Generator 或直接依据 Swagger UI 调试。

## 3. 接口设计规范 (API Design)

### 3.1 路径命名 (Path Naming)
- 使用 **kebab-case** (短横线隔开) 命名 URL 路径。
- 使用 **复数名词** 表示资源集合。
- 避免在 URL 中出现动词（特殊操作除外，如 `/users/{id}/activate`）。

**正例**:
- `GET /api/users` (获取用户列表)
- `GET /api/users/{userId}` (获取特定用户)
- `POST /api/orders` (创建订单)

**反例**:
- `GET /api/getUsers`
- `POST /api/order/create`

### 3.2 HTTP 方法 (HTTP Methods)
| 方法 | 语义 | 幂等性 | 备注 |
| :--- | :--- | :--- | :--- |
| `GET` | 查询资源 | 是 | 参数在 URL Query 或 Path 中 |
| `POST` | 创建资源 / 复杂指令 | 否 | 参数在 Body 中 |
| `PUT` | 全量替换资源 | 是 | 必须提供完整资源数据 |
| `PATCH`| 部分更新资源 | 否(理论上) | 仅提供修改的字段 |
| `DELETE`| 删除资源 | 是 | |

### 3.3 统一响应结构 (Standard Response)
后端应使用统一的泛型类包装响应数据。

```json
{
  "code": "00000",        // 业务状态码，"00000" 表示成功
  "message": "Success",   // 提示信息
  "data": { ... },        // 实际业务数据，失败时可为 null
  "traceId": "abc-123"    // 链路追踪 ID，便于排查问题
}
```

### 3.4 状态码使用 (Status Codes)
- **2xx (成功)**:
    - `200 OK`: 通用成功（同步请求）。
    - `201 Created`: 资源创建成功。
- **4xx (客户端错误)**:
    - `400 Bad Request`: 参数校验失败（BindingException）。
    - `401 Unauthorized`: 未登录/Token 过期。
    - `403 Forbidden`: 已登录但无权限。
    - `404 Not Found`: 资源不存在。
- **5xx (服务端错误)**:
    - `500 Internal Server Error`: 系统内部异常（Bug）。

## 4. OpenAPI 注解规范 (Annotations)

后端开发必须在 Controller 和 DTO 上添加完整的 Swagger/OpenAPI 注解，严禁裸奔。

### 4.1 控制器 (Controller)
使用 `@Tag` 对 Controller 进行分类。

```java
@RestController
@RequestMapping("/api/users")
@Tag(name = "用户管理", description = "用户的增删改查及状态控制")
public class UserController { ... }
```

### 4.2 接口方法 (Operation)
使用 `@Operation` 描述具体接口。必须包含 `summary` (简述) 和 `description` (详述)。

```java
@Operation(
    summary = "注册新用户",
    description = "创建用户账号，需要验证手机号验证码。成功后返回用户 ID。"
)
@PostMapping
public Result<Long> register(@RequestBody @Valid UserRegisterCmd cmd) { ... }
```

### 4.3 参数模型 (Schema & DTO)
所有 DTO (Request/Response) 必须使用 `@Schema` 描述字段含义、示例值和约束。

```java
@Schema(description = "用户注册请求参数")
public class UserRegisterCmd {

    @Schema(description = "用户名", example = "johndoe", requiredMode = Schema.RequiredMode.REQUIRED)
    @NotBlank
    private String username;

    @Schema(description = "年龄", example = "25", maximum = "150")
    private Integer age;
    
    @Schema(description = "用户状态", implementation = UserStatus.class)
    private UserStatus status;
}
```

### 4.4 枚举处理 (Enums)
枚举类应实现统一接口，并向前端暴露清晰的定义。建议在文档中列出所有枚举值。

```java
@Schema(enumAsRef = true, description = "用户状态：NORMAL(正常), LOCKED(锁定)")
public enum UserStatus {
    NORMAL, LOCKED;
}
```

## 5. 前后端协作流程 (Collaboration Workflow)

1.  **需求评审**: 确定业务流程。
2.  **接口定义**: 
    - 后端定义 Controller 结构和 DTO（可暂不实现逻辑）。
    - 生成 OpenAPI (Swagger) 文档地址 (e.g., `/v3/api-docs`, `/swagger-ui.html`)。
3.  **前端Mock/对接**:
    - 前端根据 Swagger 文档查看字段定义。
    - 后端可提供 Mock 数据或使用 Swagger 的 Try it out。
4.  **联调**: 验证实际调用。

## 6. 特殊类型规范

- **时间日期**:
    - 统一使用 `ISO-8601` 格式字符串: `yyyy-MM-dd HH:mm:ss` 或 `yyyy-MM-dd'T'HH:mm:ss.SSSX`。
    - 避免使用时间戳（long），可读性差且有精度问题（JS Number 精度）。
- **金额**:
    - 传输时建议使用 `String` 类型或者最小单位的 `Long` (如分)，避免 `Double` 精度丢失。
- **大整数**:
    - `Long` 类型在超过 16 位时，序列化为 JSON 需转为 `String`，否则 JS 会丢失精度。

## 7. 变更管理
- **兼容性原则**: 尽量新增接口或字段，避免修改现有字段含义或类型。
- **废弃标记**: 使用 `@Deprecated` 注解标记即将下线的接口，并在 `@Operation` 中说明替代方案。
