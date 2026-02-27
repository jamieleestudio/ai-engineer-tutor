# 技术栈约定

本文档定义了项目的标准技术栈约定。这些是默认推荐的选择，但根据具体需求可以进行调整。

## 核心运行环境
- **语言**: Java 17+
- **框架**: Spring Boot 3.x

## 构建与依赖管理
- **构建工具**: Maven (默认) / Gradle (可选)

## Web 层
- **框架**: Spring MVC
- **API 文档**: springdoc-openapi (OpenAPI 3)
- **参数校验**: Jakarta Validation (`@Valid`, `@NotNull`, etc.)

## 数据存储与访问
- **数据库访问**: JPA (Spring Data JPA)
- **数据库**: MySQL
- **缓存**: Redis (可选)

## 可观测性
- **日志**: SLF4J + Logback

## 测试
- **单元/集成测试**: JUnit 5 + Mockito
- **容器化测试**: Testcontainers (可选)
