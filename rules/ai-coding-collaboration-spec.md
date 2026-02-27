# AI Coding 协作规范

本文档旨在建立一套基于契约（Contract-First）的高效 AI 辅助编码协作流程，确保前后端开发的一致性、代码质量与交付效率。

## 1. 项目结构规范

### 1.1 API 契约目录
在项目根目录下 **强制** 创建 `api-contract` 文件夹，用于集中管理所有 API 契约文件。

```text
/
├── api-contract/              # [强制] 存放所有 OpenAPI/Swagger 契约文件
│   ├── v1/                    # 版本目录
│   │   ├── user-service.yaml  # 用户服务契约
│   │   └── order-service.yaml # 订单服务契约
│   └── README.md              # 契约维护说明
├── {project}-backend/         # 后端模块（示例）
├── {project}-ui/              # [强制] 前端模块统一命名
└── ...
```

### 1.2 多模块命名与组织
- **前端模块**：统一命名为 `{project}-ui`。
  - 目录结构需遵循主流框架（如 Vue/React）最佳实践。
  - 必须包含 `src/api` 或 `src/services` 目录，用于存放基于契约生成的 API 客户端代码。
- **后端模块**：建议采用 `{project}-backend` 或按业务领域拆分（如 `{project}-user`）。
  - 模块间依赖必须单向：`UI -> Contract -> Backend (Impl)`（逻辑上）。

### 1.3 接口边界
- **物理边界**：`api-contract` 是唯一的事实来源（Single Source of Truth）。
- **逻辑边界**：
  - 前端：仅依赖契约生成的 SDK/类型定义，严禁硬编码 API 路径。
  - 后端：Controller/Resource 层必须实现契约定义的接口，严禁随意变更字段。

## 2. 契约管理规范

### 2.1 契约格式
- **标准**：必须符合 **OpenAPI 3.0+** 规范。
- **格式**：推荐使用 YAML 格式（`*.yaml` 或 `*.yml`），便于阅读和 Diff。

### 2.2 契约内容要求
每个 API 接口定义必须包含：
- **Summary & Description**：清晰的接口用途描述。
- **Operation ID**：唯一的某些操作标识（用于生成方法名）。
- **Tags**：合理的业务分组。
- **Request**：
  - Path/Query/Header 参数及校验规则（必填、长度、正则）。
  - Request Body 的完整 Schema 定义。
- **Response**：
  - 成功响应（200）的完整 Schema。
  - 常见错误响应（400, 401, 403, 500）及对应的错误码定义。

### 2.3 版本控制
- 契约文件纳入 Git 版本控制。
- **变更流程**：
  1. 变更契约文件（`api-contract/*.yaml`）。
  2. 提交 Pull Request (PR)。
  3. 审核通过后合并。
  4. 触发 CI 流程生成新的客户端代码/文档。

## 3. 协作流程规范

### 3.1 完整协作流
1.  **需求分析**：确定业务需求。
2.  **契约设计**：由后端（或架构师）在 `api-contract` 中编写/更新 OpenAPI 文档。
3.  **契约评审**：前后端开发人员共同评审契约（字段、类型、URL）。
4.  **并行开发**：
    - **前端**：使用 Mock Server（基于契约自动生成）进行 UI 开发与逻辑联调。
    - **后端**：基于契约生成 Interface/DTO，实现业务逻辑。
5.  **联调验证**：后端接口完成后，替换 Mock 服务进行真实联调。

### 3.2 变更机制
- **审批**：任何契约变更必须经由相关的前端和后端开发人员共同 Approve。
- **通知**：变更合并后，自动通知相关人员（邮件/IM），并触发 SDK 版本更新。

### 3.3 测试策略
- **Mock 服务**：开发阶段强制使用基于契约的 Mock 服务（如 Prism, Swagger Codegen Mock）。
- **契约测试**：后端 CI 流程中需运行契约测试，验证实现与文档的一致性。

## 4. 代码质量规范

### 4.1 契约合规性
- **后端**：Controller 方法签名、DTO 字段必须与 OpenAPI 定义完全一致。
  - *建议*：使用工具（如 `openapi-generator-maven-plugin`）生成接口定义，后端仅实现逻辑。
- **前端**：禁止手动拼接 URL，必须使用生成的 API Client。

### 4.2 代码审查 Checklist
- [ ] `api-contract` 是否已更新且版本号递增？
- [ ] 后端实现是否通过了契约兼容性检查？
- [ ] 前端是否使用了生成的类型定义而非 `any`？
- [ ] 错误码是否在契约中有定义？

### 4.3 自动化测试
- **API 契约测试覆盖率**：100% 接口需有对应的集成测试。
- **Schema 校验**：测试响应体必须通过 JSON Schema 校验。

## 5. 交付物要求

### 5.1 文档与工具
- **规范文档**：即本文档 (`rules/ai-coding-collaboration-spec.md`)。
- **脚手架**：项目根目录需提供 `scripts/` 或 `tools/`，包含：
  - `generate-api.sh`: 基于契约生成前后端代码的脚本。
  - `validate-contract.sh`: 校验 OpenAPI 文件格式的脚本。
  - `start-mock.sh`: 启动本地 Mock 服务的脚本。

### 5.2 持续集成 (CI)
- 每次提交必须通过 `validate-contract.sh`。
- 契约变更时自动触发 `generate-api.sh` 验证生成无误。

## 6. 验收标准

1.  **契约一致性**：所有 API 调用通过契约校验工具验证，无 Schema 错误。
2.  **独立部署**：前端可连接 Mock 环境独立部署与演示；后端可独立通过接口测试。
3.  **覆盖率**：`api-contract` 对 API 接口的覆盖率达到 100%。
4.  **可追溯**：Git Log 中可追溯每次接口变更对应的业务需求和契约修改记录。
