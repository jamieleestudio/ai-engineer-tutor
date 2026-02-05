# Microservices Architecture

Distributed architecture where applications are composed of loosely coupled, independently deployable services. Each service owns its data and communicates via APIs.

## When to Use

- Large applications with multiple teams (10+)
- Independent deployment requirements
- Different scaling needs per feature
- Clear bounded contexts

## Architecture Overview

```
┌─────────────────────────┐
│      API Gateway        │
└───────────┬─────────────┘
     ┌──────┼──────┐
     ▼      ▼      ▼
┌────────┐ ┌────────┐ ┌────────┐
│ User   │ │Product │ │ Order  │
│Service │ │Service │ │Service │
└───┬────┘ └───┬────┘ └───┬────┘
    ▼          ▼          ▼
 [UserDB]  [ProdDB]   [OrderDB]
```

## Key Skills

### 1. Service Discovery (Eureka)
```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```yaml
# application.yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

### 2. Service-to-Service Communication (Feign)
```java
@FeignClient(name = "order-service", fallback = OrderClientFallback.class)
public interface OrderServiceClient {
    
    @GetMapping("/api/orders/user/{userId}")
    List<OrderDTO> getOrdersByUser(@PathVariable Long userId);
}

@Component
class OrderClientFallback implements OrderServiceClient {
    @Override
    public List<OrderDTO> getOrdersByUser(Long userId) {
        return Collections.emptyList(); // Graceful degradation
    }
}
```

### 3. Circuit Breaker (Resilience4j)
```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final OrderServiceClient orderClient;
    
    @CircuitBreaker(name = "orderService", fallbackMethod = "getOrdersFallback")
    public List<OrderDTO> getUserOrders(Long userId) {
        return orderClient.getOrdersByUser(userId);
    }
    
    public List<OrderDTO> getOrdersFallback(Long userId, Exception ex) {
        log.warn("Circuit breaker triggered for user: {}", userId);
        return Collections.emptyList();
    }
}
```

### 4. Event Publishing (Kafka)
```java
@Component
@RequiredArgsConstructor
public class UserEventPublisher {
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishUserCreated(User user) {
        UserCreatedEvent event = UserCreatedEvent.builder()
            .userId(user.getId())
            .email(user.getEmail())
            .occurredAt(Instant.now())
            .build();
        
        kafkaTemplate.send("user-events", user.getId().toString(), event);
    }
}
```

### 5. Event Consuming
```java
@Component
@RequiredArgsConstructor
public class UserEventListener {
    
    @KafkaListener(topics = "user-events", groupId = "notification-service")
    public void handleUserCreated(UserCreatedEvent event) {
        log.info("User created: {}", event.getUserId());
        // Send welcome email, etc.
    }
}
```

### 6. API Gateway (Spring Cloud Gateway)
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
```

## Essential Patterns

| Pattern | Purpose | Technology |
|---------|---------|------------|
| Service Discovery | Find services | Eureka, Consul |
| API Gateway | Single entry point | Spring Cloud Gateway |
| Circuit Breaker | Fault tolerance | Resilience4j |
| Config Server | Centralized config | Spring Cloud Config |
| Distributed Tracing | Request tracking | Zipkin, Jaeger |

## Best Practices

- **Database per service**: No shared databases
- **Async communication**: Prefer events over sync calls
- **Idempotency**: Handle duplicate messages
- **Health checks**: Actuator endpoints
- **Centralized logging**: ELK Stack

## Anti-Patterns

- ❌ **Distributed Monolith**: Tightly coupled services
- ❌ **Shared Database**: Multiple services accessing same DB
- ❌ **Synchronous Chains**: Long chains of sync calls
- ❌ **No Circuit Breaker**: Cascade failures
