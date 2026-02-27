# General Java Technology Stack Rules for AI

This document defines the **MANDATORY** technology stack for Java backend development in this project.
**INSTRUCTION**: AI Assistants MUST strictly adhere to these choices when generating code, configuration, or suggesting libraries. Do not introduce alternative libraries (e.g., do not use Gson, use Jackson; do not use ModelMapper, use MapStruct) unless explicitly requested.

## 1. Core Runtime & Framework
- **JDK Version**: **Java 17** (Minimum) or **Java 21** (Preferred). Use modern syntax (Records, Pattern Matching, Switch Expressions, Var).
- **Framework**: **Spring Boot 3.x**.
- **Dependency Injection**: Spring Framework (Core).

## 2. Build & CI/CD
- **Build Tool**: **Maven** (Default).
- **CI/CD**: GitHub Actions / GitLab CI.

## 3. Web Layer
- **Web Framework**: **Spring MVC** (Servlet Stack).
- **API Documentation**: **SpringDoc OpenAPI** (Swagger 3).
- **Validation**: **Jakarta Validation** (`@Valid`, `@NotNull`, etc.).
- **JSON Processing**: **Jackson**.
- **REST Client**: **Spring Cloud OpenFeign** (Declarative) or **RestClient** (Fluent).

## 4. Data Persistence
- **ORM**: **Spring Data JPA** (Hibernate).
- **Database**: **MySQL 8.0+** or PostgreSQL.
- **Connection Pool**: **HikariCP**.
- **Database Migration**: **Flyway** (Preferred) or Liquibase.
- **Cache**: **Redis** (Spring Data Redis).

## 5. Security
- **Framework**: **Spring Security 6.x**.
- **Authentication**: OAuth2 / OIDC / JWT (Stateless).

## 6. Utilities & Mapping
- **Boilerplate Reduction**: **Lombok** (`@Data`, `@RequiredArgsConstructor`, `@Slf4j`, `@Builder`).
- **Object Mapping**: **MapStruct** (Performance over reflection).
- **General Utils**: Apache Commons Lang3 (StringUtils), Guava (Immutable collections - usage sparing).

## 7. Observability
- **Logging**: **SLF4J** API + **Logback** implementation (JSON layout in prod).
- **Metrics**: **Micrometer** + **Prometheus**.
- **Tracing**: **Micrometer Tracing** + **OpenTelemetry**.

## 8. Testing
- **Unit Testing**: **JUnit 5** (Jupiter).
- **Mocking**: **Mockito**.
- **Assertions**: **AssertJ** (Fluent API).
- **Integration Testing**: **Testcontainers** (Dockerized DB/Redis).
- **Architecture Testing**: **ArchUnit**.

## 9. Code Quality
- **Style Guide**: **Google Java Style**.
- **Linter**: Spotless, SonarLint.

---

**Summary for AI**:
- **Always use**: Spring Boot 3, Java 17+, Lombok, Jackson, JUnit 5, AssertJ, Mockito.
- **Never use**: JUnit 4, Log4j 1.x, XML configuration (unless legacy), Fastjson (use Jackson).
