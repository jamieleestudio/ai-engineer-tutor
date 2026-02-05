# Monolithic Multi-Module Architecture

A modular approach with multiple Maven/Gradle modules in a single deployable application. Provides compile-time boundaries and better organization.

## When to Use

- Medium-sized applications (3-10 developers)
- Need clear module boundaries
- Preparation for potential microservices
- Teams working on different features

## Project Structure

```
my-app/
├── my-app-common/       # Shared DTOs, utils, exceptions
├── my-app-domain/       # Entities, repositories
├── my-app-service/      # Business logic
├── my-app-api/          # Controllers, bootstrap
└── pom.xml              # Parent POM
```

### Module Dependency Flow
```
api → service → domain → common
```

## Key Skills

### 1. Parent POM Configuration
```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <packaging>pom</packaging>
    
    <modules>
        <module>my-app-common</module>
        <module>my-app-domain</module>
        <module>my-app-service</module>
        <module>my-app-api</module>
    </modules>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.2.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!-- Internal modules -->
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>my-app-common</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

### 2. Module POM (Service Module)
```xml
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-app</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    
    <artifactId>my-app-service</artifactId>
    
    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>my-app-domain</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
    </dependencies>
</project>
```

### 3. API Module (Bootstrap)
```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>my-app-service</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

### 4. Shared API Response (Common Module)
```java
// In my-app-common
@Data
@Builder
public class ApiResponse<T> {
    private boolean success;
    private T data;
    private String message;
    
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder().success(true).data(data).build();
    }
    
    public static <T> ApiResponse<T> error(String message) {
        return ApiResponse.<T>builder().success(false).message(message).build();
    }
}
```

## Module Responsibilities

| Module | Contains | Dependencies |
|--------|----------|--------------|
| **common** | DTOs, Utils, Exceptions | External libs only |
| **domain** | Entities, Repositories | common |
| **service** | Business logic, Transactions | domain |
| **api** | Controllers, Config, Main | service |

## Best Practices

- **No circular dependencies** between modules
- **Version management** in parent POM
- Keep **common** truly shared (no business logic)
- **API module** only for bootstrap and web concerns
- Use **service interfaces** for cross-module communication

## Dependency Rules
```
✅ api → service → domain → common
❌ domain → service (upward)
❌ common → anything (circular)
```

## Migration Path

To Microservices:
1. Identify bounded contexts
2. Add API contracts between modules
3. Deploy modules as separate services
4. Add service discovery
