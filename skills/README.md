# Technical Skills Matrix

This directory contains deep-dives into specific technical skills, tools, and frameworks. These resources are referenced by the Role guides but are organized here by technology to avoid duplication.

## Organization Strategy

### 1. [languages](./languages)
Core programming languages and their ecosystems.
- **Java**: Core syntax, streams, concurrency, JVM internals.
- **Python**: Scripting, data science libraries, async/await.
- **TypeScript/JavaScript**: Modern ES features, types.

### 2. [frameworks](./frameworks)
Libraries and frameworks built on top of the languages.
- **Spring Boot**: Dependency injection, data, security (moved from Roles if applicable).
- **React/Vue**: Component lifecycle, state management.
- **PyTorch/TensorFlow**: Model building, training loops.
- **[Java Architecture Patterns](./frameworks/java-architecture-patterns)**: Backend architecture patterns including:
  - Monolithic (Single/Multi-Module)
  - Layered Architecture
  - Microservices Architecture
  - DDD (Domain-Driven Design)
  - Hexagonal (Ports & Adapters)
  - Clean Architecture
  - CQRS
  - Event-Driven Architecture

### 3. [infrastructure](./infrastructure)
Tools for deploying and running applications.
- **Docker/Kubernetes**: Containerization and orchestration.
- **Cloud Providers**: AWS, Azure, GCP specific services.
- **CI/CD**: GitHub Actions, Jenkins, GitLab CI.

### 4. [ai-engineering](./ai-engineering)
Specialized skills for the AI era.
- **Prompt Engineering**: Zero-shot, few-shot, CoT patterns.
- **RAG (Retrieval Augmented Generation)**: Vector DBs, embeddings, chunking strategies.
- **Agents**: Tool use, planning, memory management.

### 5. [tools](./tools)
Developer productivity tools.
- **IDEs**: IntelliJ IDEA, VS Code extensions.
- **CLI**: Shell scripting, git advanced usage.

## How to Contribute
When adding a new skill:
1. Determine the category (Language, Framework, Infra, etc.).
2. Create a markdown file (e.g., `frameworks/spring-boot.md`).
3. Include:
   - **Concepts**: Key theoretical concepts.
   - **Code Snippets**: Practical examples.
   - **Best Practices**: Dos and Don'ts.
   - **AI Prompts**: How to use AI to generate/refactor code for this skill.
