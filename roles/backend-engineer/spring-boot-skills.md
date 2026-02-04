# Spring Boot Skills & Patterns

This document outlines the essential skills, patterns, and best practices for Backend Engineers working with Spring Boot in this project.

## Core Architecture Patterns

### 1. Layered Architecture
Strict separation of concerns is mandatory:
- **Controller Layer** (`@RestController`): Handles HTTP requests, validation, and response formatting. **Thin layer**.
- **Service Layer** (`@Service`): Contains business logic, transaction management, and orchestration.
- **Repository Layer** (`@Repository`): Abstraction for data access (JPA, MyBatis, etc.).
- **DTOs**: Data Transfer Objects for API request/response. **Never expose Entities directly**.

### 2. Dependency Injection
- **Constructor Injection**: Always prefer constructor injection over `@Autowired` on fields.
  ```java
  @Service
  public class OrderService {
      private final OrderRepository orderRepository;
      
      public OrderService(OrderRepository orderRepository) {
          this.orderRepository = orderRepository;
      }
  }
  ```

### 3. Exception Handling
- Use `@ControllerAdvice` for global exception handling.
- Return standardized error responses (RFC 7807 Problem Details).
- Custom exception hierarchy (e.g., `BusinessException`, `ResourceNotFoundException`).

## Implementation Best Practices

### REST API Design
- **Versioning**: Use URI versioning (e.g., `/api/v1/users`).
- **HTTP Verbs**: Use correct verbs (GET, POST, PUT, DELETE, PATCH).
- **Status Codes**: Return appropriate codes (200, 201, 204, 400, 401, 403, 404, 500).
- **Pagination**: Use `Pageable` and `Page<T>` for list endpoints.

### Data Access (Spring Data JPA)
- **Repositories**: Extend `JpaRepository` or `CrudRepository`.
- **Transactions**: Use `@Transactional` at the service level.
- **Auditing**: Use `@EntityListeners(AuditingEntityListener.class)` for created/updated timestamps.

### Validation
- Use **Bean Validation (JSR 380)** annotations (`@NotNull`, `@Size`, `@Email`) on DTOs.
- Use `@Valid` in controllers to trigger validation.

### Security
- **Spring Security**: Configure filter chains for authentication/authorization.
- **JWT**: Stateless authentication using JSON Web Tokens.
- **Password Hashing**: Use `BCryptPasswordEncoder`.

### Performance & Scalability
- **Caching**: Use `@Cacheable` with Redis or Caffeine for read-heavy data.
- **Async**: Use `@Async` for non-blocking operations (email sending, file processing).
- **Scheduling**: Use `@Scheduled` for background tasks.

## Observability
- **Logging**: Use SLF4J with structured logging.
- **Metrics**: Expose metrics via Actuator + Micrometer (Prometheus).
- **Tracing**: Implement distributed tracing (Zipkin/Jaeger) for microservices.

## Testing Strategy
- **Unit Tests**: JUnit 5 + Mockito (Service layer).
- **Integration Tests**: `@SpringBootTest` + `TestContainers` (Database/Controller layer).
- **Contract Tests**: Spring Cloud Contract (Microservices).
