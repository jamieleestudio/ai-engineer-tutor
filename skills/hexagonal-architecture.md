# Hexagonal Architecture (Ports & Adapters)

Isolates application core from external concerns through ports (interfaces) and adapters (implementations). Business logic is independent of frameworks, databases, and UI.

## When to Use

- High testability requirements
- May switch infrastructure (DB, messaging)
- Long-lived applications
- Clean separation from frameworks

## Architecture Overview

```
           ┌─────────────┐
           │ REST Adapter│
           └──────┬──────┘
                  ▼
         ┌───────────────┐
         │  Input Port   │
         │  (Use Case)   │
         ├───────────────┤
         │  APPLICATION  │
         │     CORE      │
         │   (Domain)    │
         ├───────────────┤
         │ Output Port   │
         │ (Repository)  │
         └───────┬───────┘
                 ▼
         ┌───────────────┐
         │  DB Adapter   │
         └───────────────┘
```

## Project Structure

```
src/main/java/com/example/payment/
├── core/
│   ├── domain/
│   │   ├── model/
│   │   │   ├── Payment.java
│   │   │   └── Money.java
│   │   └── service/
│   └── application/
│       ├── port/
│       │   ├── input/
│       │   │   └── ProcessPaymentUseCase.java
│       │   └── output/
│       │       ├── PaymentRepository.java
│       │       └── PaymentGateway.java
│       └── service/
│           └── PaymentService.java
└── adapter/
    ├── input/
    │   └── rest/
    │       └── PaymentController.java
    └── output/
        ├── persistence/
        │   └── PaymentRepositoryAdapter.java
        └── gateway/
            └── StripePaymentAdapter.java
```

## Key Skills

### 1. Input Port (Use Case Interface)
```java
// Core - port/input/
public interface ProcessPaymentUseCase {
    PaymentResult process(ProcessPaymentCommand command);
}

@Builder
public record ProcessPaymentCommand(
    UUID orderId,
    BigDecimal amount,
    String cardToken
) {}
```

### 2. Output Port (Repository Interface)
```java
// Core - port/output/
public interface PaymentRepository {
    Payment save(Payment payment);
    Optional<Payment> findById(PaymentId id);
}

public interface PaymentGateway {
    GatewayResponse charge(String cardToken, Money amount);
}
```

### 3. Application Service (Implements Input Port)
```java
// Core - application/service/
@RequiredArgsConstructor
public class PaymentService implements ProcessPaymentUseCase {
    private final PaymentRepository repository;
    private final PaymentGateway gateway;
    
    @Override
    public PaymentResult process(ProcessPaymentCommand cmd) {
        Payment payment = Payment.create(cmd.orderId(), Money.of(cmd.amount()));
        payment.markProcessing();
        
        GatewayResponse response = gateway.charge(cmd.cardToken(), payment.getAmount());
        
        if (response.isSuccess()) {
            payment.complete(response.getTransactionId());
        } else {
            payment.fail(response.getErrorMessage());
        }
        
        repository.save(payment);
        return new PaymentResult(payment.getId(), payment.getStatus());
    }
}
```

### 4. Input Adapter (REST Controller)
```java
// Adapter - input/rest/
@RestController
@RequestMapping("/api/payments")
@RequiredArgsConstructor
public class PaymentController {
    private final ProcessPaymentUseCase processPayment;
    
    @PostMapping
    public ResponseEntity<PaymentResponse> process(@Valid @RequestBody PaymentRequest request) {
        ProcessPaymentCommand cmd = ProcessPaymentCommand.builder()
            .orderId(request.getOrderId())
            .amount(request.getAmount())
            .cardToken(request.getCardToken())
            .build();
        
        PaymentResult result = processPayment.process(cmd);
        return ResponseEntity.ok(toResponse(result));
    }
}
```

### 5. Output Adapter (Repository Implementation)
```java
// Adapter - output/persistence/
@Component
@RequiredArgsConstructor
public class PaymentRepositoryAdapter implements PaymentRepository {
    private final PaymentJpaRepository jpaRepository;
    private final PaymentMapper mapper;
    
    @Override
    public Payment save(Payment payment) {
        PaymentEntity entity = mapper.toEntity(payment);
        return mapper.toDomain(jpaRepository.save(entity));
    }
    
    @Override
    public Optional<Payment> findById(PaymentId id) {
        return jpaRepository.findById(id.getValue())
            .map(mapper::toDomain);
    }
}
```

## Dependency Rules

```
Adapters → Core (allowed)
Core → Adapters (NOT allowed)

Input Adapters  → Input Ports
Output Adapters → Output Ports (implements)
```

## Bean Configuration
```java
@Configuration
public class PaymentConfig {
    @Bean
    public ProcessPaymentUseCase processPaymentUseCase(
            PaymentRepository repository,
            PaymentGateway gateway) {
        return new PaymentService(repository, gateway);
    }
}
```

## Testing Strategy

```java
// Unit test core without Spring
@Test
void processPayment_Success() {
    PaymentRepository repo = mock(PaymentRepository.class);
    PaymentGateway gateway = mock(PaymentGateway.class);
    
    when(gateway.charge(any(), any()))
        .thenReturn(GatewayResponse.success("txn_123"));
    when(repo.save(any())).thenAnswer(i -> i.getArgument(0));
    
    PaymentService service = new PaymentService(repo, gateway);
    PaymentResult result = service.process(command);
    
    assertThat(result.status()).isEqualTo("COMPLETED");
}
```

## Benefits

| Benefit | Description |
|---------|-------------|
| Testability | Core tested without infrastructure |
| Flexibility | Easy adapter swapping |
| Independence | No framework in core |
