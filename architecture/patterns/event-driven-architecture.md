# Event-Driven Architecture

A design paradigm where services communicate through events. Components produce, detect, consume, and react to events, enabling loose coupling and high scalability.

## When to Use

- Systems requiring loose coupling
- High-throughput async processing
- Microservices communication
- Audit trail requirements
- Real-time data processing

## Architecture Patterns

```
Event Notification          Event-Carried State Transfer
─────────────────          ─────────────────────────────
Producer ──event──► Consumer    Producer ──event+data──► Consumer
  │                               │
  └─ \"Order placed\"               └─ \"User updated\" + {name, email}

Event Sourcing
──────────────
Commands ──► Event Store ──► Multiple Consumers
                         ├──► Read Model
                         ├──► Analytics
                         └──► Audit Log
```

## Key Skills

### 1. Domain Event
```java
@Getter
@Builder
public class OrderCreatedEvent {
    private final String eventId = UUID.randomUUID().toString();
    private final String orderId;
    private final String customerId;
    private final BigDecimal totalAmount;
    private final Instant occurredAt = Instant.now();
    
    // Metadata for tracing
    private String correlationId;
    private String causationId;
}
```

### 2. Event Publisher (Kafka)
```java
@Component
@RequiredArgsConstructor
public class EventPublisher {
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ObjectMapper objectMapper;
    
    public void publish(String topic, Object event) {
        try {
            String payload = objectMapper.writeValueAsString(event);
            kafkaTemplate.send(topic, getAggregateId(event), payload)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error(\"Failed to publish: {}\", event, ex);
                    }
                });
        } catch (JsonProcessingException e) {
            throw new EventPublishException(\"Serialization failed\", e);
        }
    }
}
```

### 3. Event Consumer with Retry
```java
@Component
@RequiredArgsConstructor
public class OrderEventListener {
    private final PaymentService paymentService;
    
    @RetryableTopic(
        attempts = \"3\",
        backoff = @Backoff(delay = 1000, multiplier = 2),
        dltTopicSuffix = \"-dlt\"
    )
    @KafkaListener(topics = \"order-events\", groupId = \"payment-service\")
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info(\"Processing order: {}\", event.getOrderId());
        
        // Idempotency check
        if (paymentService.isAlreadyProcessed(event.getOrderId())) {
            log.info(\"Already processed: {}\", event.getOrderId());
            return;
        }
        
        paymentService.processPayment(event);
    }
    
    // Dead Letter Topic handler
    @KafkaListener(topics = \"order-events-dlt\", groupId = \"payment-service-dlt\")
    public void handleDlt(ConsumerRecord<String, String> record) {
        log.error(\"Event failed permanently: {}\", record.value());
        // Alert or manual intervention
    }
}
```

### 4. Transactional Outbox Pattern
```java
// Ensures events are published reliably with database transaction
@Entity
@Table(name = \"outbox_events\")
public class OutboxEvent {
    @Id
    private String id;
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    private String payload;
    private Instant createdAt;
    private boolean published;
}

@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = // create order
        orderRepository.save(order);
        
        // Save event in same transaction
        OutboxEvent event = OutboxEvent.builder()
            .id(UUID.randomUUID().toString())
            .aggregateType(\"Order\")
            .aggregateId(order.getId())
            .eventType(\"OrderCreated\")
            .payload(toJson(order))
            .createdAt(Instant.now())
            .published(false)
            .build();
        outboxRepository.save(event);
        
        return order;
    }
}

// Separate job publishes events
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishOutboxEvents() {
    List<OutboxEvent> events = outboxRepository.findUnpublished();
    for (OutboxEvent event : events) {
        kafkaTemplate.send(event.getAggregateType() + \"-events\", event.getPayload());
        event.setPublished(true);
    }
}
```

### 5. Saga Pattern (Choreography)
```java
// Order Service - starts saga
@Transactional
public Order createOrder(CreateOrderRequest request) {
    Order order = // create and save order
    eventPublisher.publish(\"order-events\", new OrderCreatedEvent(order));
    return order;
}

// Payment Service - continues saga
@KafkaListener(topics = \"order-events\")
public void onOrderCreated(OrderCreatedEvent event) {
    try {
        Payment payment = processPayment(event.getOrderId());
        eventPublisher.publish(\"payment-events\", 
            new PaymentCompletedEvent(event.getOrderId(), payment.getId()));
    } catch (PaymentException e) {
        // Compensation event
        eventPublisher.publish(\"payment-events\",
            new PaymentFailedEvent(event.getOrderId(), e.getMessage()));
    }
}

// Inventory Service - continues saga
@KafkaListener(topics = \"payment-events\")
public void onPaymentCompleted(PaymentCompletedEvent event) {
    try {
        reserveInventory(event.getOrderId());
        eventPublisher.publish(\"inventory-events\",
            new InventoryReservedEvent(event.getOrderId()));
    } catch (InsufficientStockException e) {
        // Trigger refund compensation
        eventPublisher.publish(\"inventory-events\",
            new InventoryFailedEvent(event.getOrderId()));
    }
}
```

## Kafka Configuration
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      acks: all
      retries: 3
      properties:
        enable.idempotence: true
    consumer:
      group-id: ${spring.application.name}
      auto-offset-reset: earliest
      enable-auto-commit: false
    listener:
      ack-mode: manual
```

## Best Practices

| Practice | Description |
|----------|-------------|
| Idempotency | Handle duplicate events |
| Ordering | Same partition key for related events |
| Dead Letter | Handle failed events |
| Schema Evolution | Use Avro/Protobuf with schema registry |
| Correlation ID | Track requests across services |

## Event Design

```java
// Good: Self-contained event
OrderCreatedEvent {
    orderId: \"123\"
    customerId: \"456\"
    customerName: \"John\"   // Denormalized
    items: [...]           // Full data
    totalAmount: 100.00
    occurredAt: \"2024-01-01T10:00:00Z\"
}

// Bad: Requires additional lookup
OrderCreatedEvent {
    orderId: \"123\"
    // Consumer needs to call Order Service
}
```

## Anti-Patterns

- ❌ Large event payloads (use references for big data)
- ❌ Tight coupling through events
- ❌ Ignoring event ordering
- ❌ Missing idempotency
- ❌ No dead letter handling
