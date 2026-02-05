# Domain-Driven Design (DDD) Architecture

A software design approach focused on modeling complex business domains. Uses strategic patterns (bounded contexts) and tactical patterns (aggregates, entities, value objects).

## When to Use

- Complex business domains with intricate rules
- Close collaboration with domain experts needed
- Long-term applications that will evolve
- Teams of 5+ developers

## Core Concepts

```
Strategic Design          Tactical Design
─────────────────         ─────────────────
Bounded Context           Entity
Ubiquitous Language       Value Object
Context Mapping           Aggregate
                          Domain Service
                          Domain Event
                          Repository
```

## Project Structure

```
src/main/java/com/example/order/
├── domain/                  # Core Domain
│   ├── model/
│   │   ├── Order.java       # Aggregate Root
│   │   ├── OrderLine.java   # Entity
│   │   ├── OrderId.java     # Value Object
│   │   └── Money.java       # Value Object
│   ├── event/
│   │   └── OrderCreatedEvent.java
│   ├── service/
│   │   └── PricingService.java
│   └── repository/
│       └── OrderRepository.java   # Interface
├── application/             # Use Cases
│   ├── service/
│   └── dto/
└── infrastructure/          # Implementation
    └── persistence/
```

## Key Skills

### 1. Value Object (Immutable)
```java
@Getter
@EqualsAndHashCode
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;
    
    private Money(BigDecimal amount, Currency currency) {
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }
    
    public static Money of(BigDecimal amount, String currency) {
        return new Money(amount, Currency.getInstance(currency));
    }
    
    public Money add(Money other) {
        validateSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }
    
    public Money multiply(int quantity) {
        return new Money(amount.multiply(BigDecimal.valueOf(quantity)), currency);
    }
}
```

### 2. Aggregate Root (Business Rules Inside)
```java
@Getter
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private OrderStatus status;
    private final List<OrderLine> lines = new ArrayList<>();
    private Money totalAmount;
    
    public static Order create(OrderId id, CustomerId customerId) {
        Order order = new Order(id, customerId);
        order.registerEvent(new OrderCreatedEvent(id));
        return order;
    }
    
    public void addLine(ProductId productId, int quantity, Money unitPrice) {
        validateModifiable();
        lines.add(new OrderLine(productId, quantity, unitPrice));
        recalculateTotal();
    }
    
    public void submit() {
        if (lines.isEmpty()) {
            throw new OrderDomainException("Cannot submit empty order");
        }
        status = OrderStatus.SUBMITTED;
    }
    
    private void validateModifiable() {
        if (status != OrderStatus.DRAFT) {
            throw new OrderDomainException("Order cannot be modified");
        }
    }
}
```

### 3. Domain Event
```java
@Getter
@RequiredArgsConstructor
public class OrderCreatedEvent {
    private final OrderId orderId;
    private final Instant occurredAt = Instant.now();
}
```

### 4. Repository Interface (Domain Layer)
```java
// In domain layer - interface only
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
}
```

### 5. Application Service (Orchestration)
```java
@Service
@RequiredArgsConstructor
@Transactional
public class OrderApplicationService {
    private final OrderRepository orderRepository;
    private final EventPublisher eventPublisher;
    
    public OrderDTO createOrder(CreateOrderCommand cmd) {
        Order order = Order.create(
            OrderId.generate(),
            new CustomerId(cmd.getCustomerId())
        );
        
        cmd.getItems().forEach(item -> 
            order.addLine(new ProductId(item.getProductId()), 
                         item.getQuantity(), 
                         Money.of(item.getPrice(), "USD"))
        );
        
        Order saved = orderRepository.save(order);
        saved.getDomainEvents().forEach(eventPublisher::publish);
        
        return toDTO(saved);
    }
}
```

## Rich vs Anemic Domain Model

```java
// ❌ Anemic (Anti-pattern)
public class Order {
    private OrderStatus status;
    public void setStatus(OrderStatus s) { this.status = s; }
}

// ✅ Rich (Correct)
public class Order {
    private OrderStatus status;
    
    public void ship(String trackingNumber) {
        if (status != OrderStatus.PAID) {
            throw new OrderDomainException("Cannot ship unpaid order");
        }
        status = OrderStatus.SHIPPED;
        registerEvent(new OrderShippedEvent(id, trackingNumber));
    }
}
```

## Aggregate Design Rules

1. **One aggregate per transaction**
2. **Reference other aggregates by ID**
3. **Keep aggregates small**
4. **Use eventual consistency between aggregates**

## Best Practices

- Use **Ubiquitous Language** in code
- **Protect invariants** inside aggregates
- **Domain Events** for cross-aggregate communication
- **Repository per Aggregate Root**
- Keep **domain layer pure** (no framework dependencies)
