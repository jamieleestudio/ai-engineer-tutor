# Monolithic Single Module Architecture

The simplest architecture pattern where all code resides in a single module with one deployable artifact.

## When to Use

- MVP/prototype development
- Small applications (1-3 developers)
- Learning projects
- Tight deadlines, simple business logic

## Project Structure

```
my-app/
├── src/main/java/com/example/
│   ├── MyAppApplication.java
│   ├── config/          # Configuration classes
│   ├── controller/      # REST controllers
│   ├── service/         # Business logic
│   ├── repository/      # Data access
│   ├── entity/          # JPA entities
│   ├── dto/             # Data transfer objects
│   └── exception/       # Exception handling
├── src/main/resources/
│   └── application.yml
└── pom.xml
```

## Key Skills

### 1. Layered Package Organization
- **Controller**: Thin layer, handles HTTP only
- **Service**: Business logic and transactions
- **Repository**: Data access abstraction
- **DTO**: API contracts (never expose entities)

### 2. Spring Boot Essentials
```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;
    
    @GetMapping("/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDTO createUser(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }
}
```

### 3. Service Layer Pattern
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserService {
    private final UserRepository userRepository;
    
    public UserDTO findById(Long id) {
        return userRepository.findById(id)
            .map(this::toDTO)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }
    
    @Transactional
    public UserDTO create(CreateUserRequest request) {
        User user = User.builder()
            .email(request.getEmail())
            .name(request.getName())
            .build();
        return toDTO(userRepository.save(user));
    }
}
```

### 4. Global Exception Handling
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setProperty("errors", ex.getFieldErrors());
        return problem;
    }
}
```

## Best Practices

- Use constructor injection (`@RequiredArgsConstructor`)
- Keep controllers thin
- `@Transactional` at service layer
- Validate DTOs with `@Valid`
- Use Lombok to reduce boilerplate
- Profile-based configuration (`application-{env}.yml`)

## Migration Path

When the app grows:
1. Extract modules → **Multi-Module**
2. Add formal layers → **Layered Architecture**
3. Extract services → **Microservices**
