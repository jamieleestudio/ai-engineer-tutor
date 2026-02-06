# Skill Name: Evolvable Architecture

## Description
This skill defines the architecture guidelines and module interaction rules for a multi-module, evolvable system. It provides clear rules for layer responsibilities, internal module calls, cross-module calls, microservices/RPC integration, and input adapters (Controllers, MQ listeners, RPC endpoints).  
The skill ensures **single-direction dependencies, modular independence, and future evolvable microservices**, supported by ArchUnit rules for enforcement.

---

## Layers and Responsibilities

| Layer | Responsibility | Allowed Dependencies | Forbidden Content |
|-------|----------------|--------------------|-----------------|
| **api** | Stable module contract: interfaces, DTOs, Commands, Queries | None | Controller, business implementation, technical infrastructure |
| **application** | Use case orchestration, transaction boundaries | domain, infrastructure, api (cross-module) | Controller, HTTP/MQ protocols |
| **domain** | Core domain model and rules | None | HTTP, MQ, Controller, cross-module dependency |
| **infrastructure** | Technical implementation: DB, cache, third-party services | domain, application | Business logic, Controller, cross-module dependency |
| **interfaces** | Input adapters: Controllers, MQ consumers, RPC implementation (Provider) | application, api | Core business logic, cross-module orchestration, direct domain/infrastructure access |

---

## Controller / Input Adapter Guidelines

- **Controllers are never in api or application layers.**  
- Place Controllers in **interfaces** layer or adapter module (`module-interfaces` / `module-adapter-web`).  
- Controllers call **application services**, never domain or repository directly.  
- `interfaces` layer may include:
  - HTTP Controllers (`@RestController`)
  - MQ Consumers (`@KafkaListener`, `@RabbitListener`)
  - RPC Implementation (`@DubboService`, `@GrpcService` - implements interfaces defined in `api` layer)
  - Protocol-specific DTOs and exception mapping  
- `interfaces` layer must not contain:
  - Core business logic  
  - Cross-module orchestration  
  - Repository or external client logic  

**Example: OrderController**

```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final OrderApplicationService orderApp;

    @PostMapping
    public OrderDTO createOrder(@RequestBody CreateOrderCommand cmd) {
        return orderApp.create(cmd);
    }
}
```
