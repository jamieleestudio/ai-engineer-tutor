# Monolithic Single Module Architecture

The simplest architecture pattern where all code resides in a single module with one deployable artifact.

## When to Use

- MVP/prototype development
- Small applications (1-3 developers)
- Learning projects
- Tight deadlines, simple business logic

## Project Structure

### Option A: Package by Layer (Traditional)
Good for very small apps, but leads to \"spaghetti code\" as the app grows.

```
src/main/java/com/example/
├── config/
├── controller/      # All controllers mixed here
├── service/         # All services mixed here
├── repository/      # All repositories mixed here
└── entity/
```

### Option C: Domain-Driven Design (Highly Recommended)
Organize code by Domain (Bounded Context) first, then by Hexagonal/Clean Architecture layers.

```
my-app/
├── src/main/java/com/example/
│   ├── MyAppApplication.java
│   ├── shared/          # Kernel: Value Objects, Base Classes
│   │
│   ├── identity/        # Domain: Identity & Access Management
│   │   ├── api/         # Inbound Adapters (Controllers)
│   │   ├── domain/      # Core Business Logic (Entities, Aggregates)
│   │   │   ├── model/
│   │   │   └── service/ # Domain Services
│   │   ├── infrastructure/ # Outbound Adapters (Persistence, Clients)
│   │   └── application/    # Application Services (Use Cases)
│   │
│   └── catalog/         # Domain: Product Catalog
│       ├── api/
│       ├── domain/
│       ├── infrastructure/
│       └── application/
```

## Key Skills

### 1. DDD Layer Responsibilities
- **API (Interface Layer)**: Controllers, DTOs. Translates HTTP to App Commands.
- **Application**: Orchestration, transaction boundaries, security. **No business rules**.
- **Domain**: Pure business logic. Entities, Value Objects, Domain Services. **No Spring dependencies**.
- **Infrastructure**: Implementation of repositories, external clients, configs.

### 2. Domain Model (Pure Java)
```java
// domain/model/User.java
@Getter
// No JPA annotations here if using strict DDD (optional separation)
public class User {
    private UserId id;
    private Email email;
    
    public void changeEmail(Email newEmail) {
        // Business Invariant
        if (newEmail == null) throw new IllegalArgumentException(\"Email required\");
        this.email = newEmail;
    }
}
```

### 3. Application Service (Orchestration)
```java
// application/UserApplicationService.java
@Service
@RequiredArgsConstructor
public class UserApplicationService {
    private final UserRepository repo;
    private final DomainEventPublisher publisher;

    @Transactional
    public void registerUser(RegisterUserCommand cmd) {
        User user = User.create(cmd.email(), cmd.name());
        repo.save(user);
        publisher.publish(new UserRegisteredEvent(user.getId()));
    }
}
```

### 4. Infrastructure (Persistence Adapter)
```java
// infrastructure/persistence/JpaUserRepository.java
@Repository
@RequiredArgsConstructor
public class JpaUserRepository implements UserRepository {
    private final SpringDataUserRepo jpaRepo;
    private final UserMapper mapper;

    @Override
    public void save(User user) {
        UserEntity entity = mapper.toEntity(user);
        jpaRepo.save(entity);
    }
}
```

### 5. Testing Strategy (DDD)
- **Domain Tests**: Pure JUnit (No Spring). Fast execution.
- **Application Tests**: Mock repositories, test use case flows.
- **Contract Tests**: Ensure domain boundaries are respected.

### 6. Global Exception Handling
Use `ProblemDetail` (Spring Boot 3 standard).

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }
}
```

## Best Practices

- **Package by Feature**: Groups related code together.
- **Constructor Injection**: Always.
- **Validation**: JSR-380 on DTOs.
- **Lombok**: Reduce boilerplate (Data, Builder, Slf4j).
- **Flyway/Liquibase**: Version control your database schema from day 1.

## Migration Path

1. **Refactor**: Move from \"Package by Layer\" to \"Package by Feature\".
2. **Modularize**: Move feature packages into Maven/Gradle modules.
3. **Split**: Deploy modules as separate Microservices.
