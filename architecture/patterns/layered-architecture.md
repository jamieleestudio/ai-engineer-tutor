# Layered Architecture (N-Tier)

Traditional horizontal layer organization with strict unidirectional dependencies. Each layer has a specific responsibility.

## When to Use

- Traditional enterprise applications
- CRUD-heavy applications
- Teams familiar with layered patterns
- Clear separation of concerns needed

## Layer Structure

```
┌─────────────────────────────┐
│     Presentation Layer      │  ← Controllers, DTOs
├─────────────────────────────┤
│      Business Layer         │  ← Services, Business Rules
├─────────────────────────────┤
│     Persistence Layer       │  ← Repositories, Entities
├─────────────────────────────┤
│        Database             │
└─────────────────────────────┘
```

## Project Structure

```
src/main/java/com/example/
├── presentation/
│   ├── controller/
│   ├── dto/request/
│   ├── dto/response/
│   └── advice/
├── business/
│   ├── service/
│   ├── mapper/
│   └── validator/
└── persistence/
    ├── entity/
    ├── repository/
    └── specification/
```

## Key Skills

### 1. Presentation Layer (Thin)
```java
@RestController
@RequestMapping(\"/api/v1/users\")
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;
    
    @GetMapping(\"/{id}\")
    public UserResponse getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }
}
```

### 2. Business Layer (Core Logic)
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserService {
    private final UserRepository userRepository;
    private final UserMapper userMapper;
    
    public UserResponse findById(Long id) {
        return userRepository.findById(id)
            .map(userMapper::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException(\"User\", id));
    }
    
    @Transactional
    public UserResponse create(CreateUserRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException(\"Email already exists\");
        }
        UserEntity entity = userMapper.toEntity(request);
        return userMapper.toResponse(userRepository.save(entity));
    }
}
```

### 3. Persistence Layer
```java
@Entity
@Table(name = \"users\")
@Data
@Builder
public class UserEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(nullable = false)
    private String name;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

@Repository
public interface UserRepository extends JpaRepository<UserEntity, Long> {
    boolean existsByEmail(String email);
    Optional<UserEntity> findByEmail(String email);
}
```

### 4. DTO Mapping (MapStruct)
```java
@Mapper(componentModel = \"spring\")
public interface UserMapper {
    UserResponse toResponse(UserEntity entity);
    
    @Mapping(target = \"id\", ignore = true)
    @Mapping(target = \"createdAt\", ignore = true)
    UserEntity toEntity(CreateUserRequest request);
}
```

## Layer Responsibilities

| Layer | Should | Should NOT |
|-------|--------|------------|
| Presentation | HTTP handling, validation | Business logic, DB access |
| Business | Business rules, transactions | HTTP concerns, direct queries |
| Persistence | Data access, entity mapping | Business decisions |

## Best Practices

- **Dependency direction**: Upper → Lower only
- **DTOs at boundaries**: Never expose entities in controllers
- **Transactions**: Declare at service layer
- **Validation**: Request DTOs with Bean Validation
- **Cross-cutting**: Use AOP for logging, security

## Anti-Patterns

- ❌ Business logic in controllers
- ❌ Entities exposed in API
- ❌ Lower layer calling upper layer
- ❌ Bypassing layers (Controller → Repository)
