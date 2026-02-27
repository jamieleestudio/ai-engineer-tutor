---
name: "ddd-scaffolder"
description: "Generates Domain-Driven Design (DDD) compliant code structures for new business domains. Invoke when the user wants to create a new domain, feature, or entity."
---

# DDD Scaffolder

This skill generates a complete 4-layer DDD structure for a given business domain (e.g., "Order", "User"). It strictly follows the architectural rules defined in `rules/monolith-single-module.md`.

## Workflow

1.  **Analyze Request**: Identify the domain name (e.g., `Order`) and key attributes.
2.  **Verify Rules**: Check against `monolith-single-module.md` for:
    *   Package structure (`domain`, `application`, `interfaces`, `infrastructure`)
    *   Naming conventions (`*Command`, `*Query`, `*Request`, `*RepositoryImpl`)
    *   Dependency boundaries
3.  **Generate Code**: Produce the following files with boilerplate code:

### 1. Domain Layer (`domain/model`)
*   **Entity**: `Xxx.java` (Rich domain model with behaviors, no setters)
*   **Identifier**: `XxxId.java` (Strongly typed ID)
*   **Repository Interface**: `domain/repository/XxxRepository.java` (Return Domain Entities)

### 2. Application Layer (`application`)
*   **Command**: `application/command/CreateXxxCommand.java`
*   **UseCase/Service**: `application/service/XxxCommandService.java`
*   **Result**: `application/dto/XxxResult.java`

### 3. Interfaces Layer (`interfaces`)
*   **Web Request**: `interfaces/web/rest/dto/CreateXxxRequest.java`
*   **Controller**: `interfaces/web/rest/XxxController.java`
*   **Mapper**: `interfaces/web/rest/mapper/XxxWebMapper.java`

### 4. Infrastructure Layer (`infrastructure`)
*   **JPA Entity**: `infrastructure/persistence/jpa/entity/XxxEntity.java`
*   **JPA Repository**: `infrastructure/persistence/jpa/JpaXxxRepository.java`
*   **Repository Impl**: `infrastructure/persistence/jpa/XxxRepositoryImpl.java` (Implements Domain Repository)

## Template Examples

### Domain Entity
```java
package com.yourorg.yourapp.domain.model.order;

import com.yourorg.yourapp.domain.model.common.AggregateRoot;

public class Order extends AggregateRoot<OrderId> {
    private Money totalAmount;
    private OrderStatus status;

    // Factory method
    public static Order create(Money totalAmount) {
        return new Order(new OrderId(), totalAmount, OrderStatus.PENDING);
    }
    // Business behaviors...
}
```

### Application Service
```java
package com.yourorg.yourapp.application.service;

@Service
@Transactional
public class OrderCommandService {
    private final OrderRepository orderRepository;

    public OrderResult createOrder(CreateOrderCommand command) {
        Order order = Order.create(command.totalAmount());
        orderRepository.save(order);
        return OrderResult.from(order);
    }
}
```

### Infrastructure Implementation
```java
package com.yourorg.yourapp.infrastructure.persistence.jpa;

@Component
public class OrderRepositoryImpl implements OrderRepository {
    private final JpaOrderRepository jpaRepo;
    private final OrderPersistenceMapper mapper;

    @Override
    public void save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        jpaRepo.save(entity);
    }
}
```

## User Prompts to Handle
*   "Create a new Order domain"
*   "Scaffold a User feature with DDD"
*   "Add a Product module"
