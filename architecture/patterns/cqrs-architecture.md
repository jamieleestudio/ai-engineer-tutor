# CQRS Architecture (Command Query Responsibility Segregation)

Separates read and write operations into different models. Commands modify state, queries retrieve data. Enables independent optimization of read/write paths.

## When to Use

- Different read/write load patterns
- Complex queries needing denormalized views
- Independent read/write scaling
- Event Sourcing implementation
- Reporting-heavy applications

## Architecture Overview

```
         ┌──────────────┐          ┌──────────────┐
         │ Command API  │          │  Query API   │
         └──────┬───────┘          └──────┬───────┘
                ▼                         ▼
         ┌──────────────┐          ┌──────────────┐
         │   Command    │          │    Query     │
         │   Handler    │          │   Handler    │
         └──────┬───────┘          └──────┬───────┘
                ▼                         ▼
         ┌──────────────┐   Event  ┌──────────────┐
         │ Write Model  │─────────►│  Read Model  │
         │ (PostgreSQL) │          │(Elasticsearch)│
         └──────────────┘          └──────────────┘
```

## Project Structure

```
src/main/java/com/example/order/
├── command/
│   ├── api/
│   │   └── OrderCommandController.java
│   ├── dto/
│   │   ├── CreateOrderCommand.java
│   │   └── CancelOrderCommand.java
│   ├── handler/
│   │   └── CreateOrderHandler.java
│   ├── domain/
│   │   ├── Order.java
│   │   └── OrderRepository.java
│   └── event/
│       └── OrderCreatedEvent.java
├── query/
│   ├── api/
│   │   └── OrderQueryController.java
│   ├── dto/
│   │   └── OrderDetailDTO.java
│   ├── handler/
│   │   └── GetOrderHandler.java
│   └── projection/
│       └── OrderProjection.java
└── sync/
    └── OrderEventListener.java
```

## Key Skills

### 1. Command DTO
```java
@Data
@Builder
public class CreateOrderCommand {
    @NotNull
    private UUID customerId;
    
    @NotEmpty
    private List<OrderItemCommand> items;
    
    @NotNull
    private AddressCommand shippingAddress;
}
```

### 2. Command Handler
```java
@Component
@RequiredArgsConstructor
public class CreateOrderHandler {
    private final OrderRepository repository;
    private final EventPublisher eventPublisher;
    
    @Transactional
    public UUID handle(CreateOrderCommand cmd) {
        Order order = Order.create(
            OrderId.generate(),
            new CustomerId(cmd.getCustomerId())
        );
        
        cmd.getItems().forEach(item -> 
            order.addItem(item.getProductId(), item.getQuantity(), item.getPrice())
        );
        
        Order saved = repository.save(order);
        
        // Publish event to sync read model
        eventPublisher.publish(new OrderCreatedEvent(
            saved.getId(),
            saved.getCustomerId(),
            saved.getTotalAmount()
        ));
        
        return saved.getId().getValue();
    }
}
```

### 3. Command Controller
```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderCommandController {
    private final CreateOrderHandler createHandler;
    
    @PostMapping
    @ResponseStatus(HttpStatus.ACCEPTED)
    public CommandResponse create(@Valid @RequestBody CreateOrderCommand cmd) {
        UUID orderId = createHandler.handle(cmd);
        return new CommandResponse(orderId, "Order creation initiated");
    }
}
```

### 4. Query Handler
```java
@Component
@RequiredArgsConstructor
public class GetOrderHandler {
    private final OrderElasticsearchRepository esRepository;
    
    public OrderDetailDTO handle(UUID orderId) {
        return esRepository.findById(orderId.toString())
            .map(this::toDTO)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
    
    public Page<OrderSummaryDTO> search(OrderSearchCriteria criteria) {
        return esRepository.search(criteria);
    }
}
```

### 5. Query Controller
```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderQueryController {
    private final GetOrderHandler handler;
    
    @GetMapping("/{id}")
    public OrderDetailDTO getOrder(@PathVariable UUID id) {
        return handler.handle(id);
    }
    
    @GetMapping
    public Page<OrderSummaryDTO> search(OrderSearchCriteria criteria) {
        return handler.search(criteria);
    }
}
```

### 6. Event Sync (Projector)
```java
@Component
@RequiredArgsConstructor
public class OrderEventListener {
    private final OrderElasticsearchRepository esRepository;
    private final CustomerClient customerClient;
    
    @KafkaListener(topics = "order-events", groupId = "query-service")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Denormalize: fetch additional data
        CustomerDTO customer = customerClient.getCustomer(event.getCustomerId());
        
        OrderDocument doc = OrderDocument.builder()
            .orderId(event.getOrderId().toString())
            .customerId(event.getCustomerId().toString())
            .customerName(customer.getName())  // Denormalized
            .totalAmount(event.getTotalAmount())
            .status(\"CREATED\")
            .createdAt(LocalDateTime.now())
            .build();
        
        esRepository.save(doc);
    }
}
```

## Read Model (Elasticsearch Document)
```java
@Document(indexName = \"orders\")
@Data
@Builder
public class OrderDocument {
    @Id
    private String orderId;
    
    @Field(type = FieldType.Keyword)
    private String customerId;
    
    @Field(type = FieldType.Text)
    private String customerName;  // Denormalized for search
    
    @Field(type = FieldType.Keyword)
    private String status;
    
    @Field(type = FieldType.Double)
    private BigDecimal totalAmount;
    
    @Field(type = FieldType.Date)
    private LocalDateTime createdAt;
}
```

## Key Concepts

| Concept | Command Side | Query Side |
|---------|--------------|------------|
| Model | Normalized domain | Denormalized views |
| Storage | PostgreSQL | Elasticsearch/Redis |
| Response | Acknowledgment | Rich data |
| Consistency | Strong | Eventual |

## Best Practices

- **Accept eventual consistency** for read model
- **Idempotent handlers** for event replay
- **Track read model lag** with monitoring
- **Separate APIs** for commands and queries
- **Events should be immutable** and self-contained

## Anti-Patterns

- ❌ Synchronous read model updates
- ❌ Query side depending on command side
- ❌ Ignoring eventual consistency in UI
