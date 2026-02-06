## 目标
- 在仓库根目录创建以下顶层文件夹：foundations、interaction、memory、agents、evaluation、safety、performance、lifecycle、integration、architecture、practices、labs、templates、assessments、roadmaps、resources、case-studies、rules、skills。
- 将现有 skills 目录下的“架构模式/可演进架构”文档迁移到新的 architecture 目录下的合适子目录，并保持文件名不变。
- 保留 skills 目录作为索引入口，指向新的位置，避免重复内容。

## 当前仓库结构参考
- 已存在的目录：phases、practices、roles、skills
- skills 现有内容以架构模式为主：[skills/README.md](file:///Users/lixiaofeng/IdeaProjects/github/ai-engineer-tutor/skills/README.md)，以及子目录“Evolvable Architecture”与若干 *.md 架构文档。

## 目标目录结构
- architecture/
  - patterns/（通用架构模式文档）
  - evolvable-architecture/（可演进架构专题）
- 其他顶层目录均创建为空文件夹（为后续扩展预留）。

## 文档迁移映射
- 移动到 architecture/patterns/
  - clean-architecture.md
  - cqrs-architecture.md
  - ddd-architecture.md
  - event-driven-architecture.md
  - hexagonal-architecture.md
  - layered-architecture.md
  - microservices-architecture.md
  - monolithic-multi-module.md
  - monolithic-single-module.md
- 移动到 architecture/evolvable-architecture/
  - README.md
  - Avoiding Module Explosion in Evolvable Architecture.md
  - Microservice Deployment for Evolvable Architecture.md
  - Modular Deployment for Evolvable Architecture.md
  - Multi-Client Design for Web, Admin, and Mobile.md

## 索引与链接更新
- 在 architecture/ 下新增 README.md，包含模式选择指南与导航，合并并更新 [skills/README.md](file:///Users/lixiaofeng/IdeaProjects/github/ai-engineer-tutor/skills/README.md) 的目录与链接为新路径。
- 更新 roles/practices 等处若有指向 skills 的相对链接（目前仅发现 skills/README.md），统一指向 architecture/ 下的新位置。
- 将 skills/README.md 精简为索引页：说明文档已迁移至 architecture/，并提供跳转链接。

## 命名与规范
- 保留现有文件名与英文标题，避免破坏外部引用。
- 子目录命名使用 kebab-case（evolvable-architecture、patterns）。
- 不引入二进制或机密文件；仅移动 Markdown 与 README。

## 兼容与验证
- 验证所有相对链接是否 200 正常：本地渲染或 Markdown 链接检查脚本。
- 检查不存在的断链；如 phases/architecture/ 与新 architecture/ 的职责区分：前者保留“阶段性说明”，后者承载“模式与技能”。
- 如需回滚，保留一次性迁移脚本以实现反向移动。

## 后续扩展建议
- foundations/ 放置基础理论（计算复杂度、网络、存储等）。
- interaction/ 与 integration/ 分别用于交互设计与系统集成模式。
- safety/ security、performance/ 性能优化、lifecycle/ 构建与运维流程等，逐步填充内容。

## 执行步骤
1. 创建所有目标顶层目录；在 architecture/ 下创建 patterns/ 与 evolvable-architecture/。
2. 移动 skills 根下的架构模式到 architecture/patterns/；移动“Evolvable Architecture”子目录内容到 architecture/evolvable-architecture/。
3. 新建 architecture/README.md，合并并更新目录与链接；精简 skills/README.md 为索引与跳转。
4. 全仓库检索并修复指向旧路径的链接。
5. 运行本地链接检查，确认无断链后结束。