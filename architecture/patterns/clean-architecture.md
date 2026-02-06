# Clean Architecture

Organizes code into concentric layers with strict dependency rules. Inner layers contain business logic, outer layers contain implementation details. Dependencies only point inward.

## When to Use

- Enterprise applications with long lifespan
- High testability requirements
- Complex business rules
- May change frameworks or infrastructure

## Layer Structure

```
┌──────────────────────────────────────┐
│      Frameworks & Drivers            │
│  ┌────────────────────────────────┐  │
│  │     Interface Adapters         │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │   Application (Use Cases)│  │  │
│  │  │  ┌────────────────────┐  │  │  │
│  │  │  │  Enterprise (Entity)│  │  │  │
│  │  │  └────────────────────┘  │  │  │
│  │  └──────────────────────────┘  │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

Dependency Direction: Outside → Inside
```

## Project Structure

```
src/main/java/com/example/user/
├── domain/                      # Enterprise Business Rules
│   ├── entity/
│   │   └── User.java
│   ├── valueobject/
│   │   ├── UserId.java
│   │   └── Email.java
│   └── service/
│       └── PasswordHasher.java  # Interface
├── usecase/                     # Application Business Rules
│   ├── port/
│   │   ├── input/
│   │   │   └── CreateUserUseCase.java
│   │   └── output/
│   │       └── UserRepository.java
│   ├── interactor/
│   │   └── CreateUserInteractor.java
│   └── dto/
├── adapter/                     # Interface Adapters
│   ├── controller/
│   │   └── UserController.java
│   └── gateway/
│       └── UserDatabaseGateway.java
└── infrastructure/              # Frameworks & Drivers
    ├── persistence/
    ├── security/
    └── config/
```

## Key Skills

### 1. Domain Entity (No Dependencies)
```java
// domain/entity/ - Pure business rules
public class User {
    private final UserId id;
    private Email email;
    private String name;
    private UserStatus status;
    
    public static User create(UserId id, Email email, String name) {
        return new User(id, email, name);
    }
    
    public void changeEmail(Email newEmail) {
        if (this.email.equals(newEmail)) {
            throw new DomainException("Email unchanged");
        }
        this.email = newEmail;
    }
    
    public void deactivate() {
        if (status == UserStatus.INACTIVE) {
            throw new DomainException("Already inactive");
        }
        status = UserStatus.INACTIVE;
    }
}
```

### 2. Use Case Interface (Input Port)
```java
// usecase/port/input/
public interface CreateUserUseCase {
    UserResponse execute(CreateUserRequest request);
}

public record CreateUserRequest(String email, String password, String name) {}
public record UserResponse(String id, String email, String name) {}
```

### 3. Use Case Interactor (Implementation)
```java
// usecase/interactor/
@RequiredArgsConstructor
public class CreateUserInteractor implements CreateUserUseCase {
    private final UserRepository userRepository;
    private final PasswordHasher passwordHasher;
    
    @Override
    public UserResponse execute(CreateUserRequest request) {
        Email email = Email.of(request.email());
        
        if (userRepository.existsByEmail(email)) {
            throw new UseCaseException("Email already exists");
        }
        
        User user = User.create(
            UserId.generate(),
            email,
            request.name()
        );
        
        User saved = userRepository.save(user);
        return new UserResponse(saved.getId().getValue(), 
                               saved.getEmail().getValue(), 
                               saved.getName());
    }
}
```

### 4. Interface Adapter (Controller)
```java
// adapter/controller/
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    private final CreateUserUseCase createUser;
    
    @PostMapping
    public ResponseEntity<UserApiResponse> create(@Valid @RequestBody UserApiRequest request) {
        CreateUserRequest useCaseRequest = new CreateUserRequest(
            request.getEmail(),
            request.getPassword(),
            request.getName()
        );
        
        UserResponse response = createUser.execute(useCaseRequest);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(toApiResponse(response));
    }
}
```

### 5. Infrastructure (Gateway Implementation)
```java
// infrastructure/persistence/
@Component
@RequiredArgsConstructor
public class UserDatabaseGateway implements UserRepository {
    private final UserJpaRepository jpaRepository;
    private final UserMapper mapper;
    
    @Override
    public User save(User user) {
        UserJpaEntity entity = mapper.toJpaEntity(user);
        return mapper.toDomain(jpaRepository.save(entity));
    }
    
    @Override
    public boolean existsByEmail(Email email) {
        return jpaRepository.existsByEmail(email.getValue());
    }
}
```

## Dependency Rule Enforcement (ArchUnit)

```java
@AnalyzeClasses(packages = "com.example.user")
class CleanArchitectureTest {
    
    @ArchTest
    static final ArchRule domainHasNoDependencies = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..usecase..", "..adapter..", "..infrastructure..");
    
    @ArchTest
    static final ArchRule useCaseOnlyDependsOnDomain = noClasses()
        .that().resideInAPackage("..usecase..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..adapter..", "..infrastructure..");
}
```

## Layer Responsibilities

| Layer | Contains | Can Depend On |
|-------|----------|---------------|
| Domain | Entities, Value Objects | Nothing |
| Use Case | Interactors, Ports | Domain |
| Adapter | Controllers, Gateways | Use Case, Domain |
| Infrastructure | JPA, Security, Config | All above |

## Best Practices

- **Domain layer has NO dependencies**
- **Use Cases orchestrate domain logic**
- **DTOs at each boundary**
- **Interfaces in inner layers, implementations in outer**
- **No framework annotations in domain/usecase**
