# Java Backend Architecture Patterns

Essential architecture patterns for Java backend development. Each pattern includes when to use, structure, and key implementation skills.

## Pattern Selection Guide

| Pattern | Complexity | Team Size | Best For |
|---------|-----------|-----------|----------|
| [Monolithic Single Module](./monolithic-single-module.md) | Low | 1-3 | MVPs, small apps |
| [Monolithic Multi-Module](./monolithic-multi-module.md) | Medium | 3-10 | Medium apps, clear boundaries |
| [Layered Architecture](./layered-architecture.md) | Low-Medium | 2-8 | Traditional enterprise |
| [Microservices](./microservices-architecture.md) | High | 10+ | Large-scale, independent deploy |
| [DDD](./ddd-architecture.md) | High | 5+ | Complex business domains |
| [Hexagonal](./hexagonal-architecture.md) | Medium-High | 3-10 | Testable, infra-agnostic |
| [Clean Architecture](./clean-architecture.md) | High | 5+ | Long-term maintainability |
| [CQRS](./cqrs-architecture.md) | High | 5+ | Read/write optimization |
| [Event-Driven](./event-driven-architecture.md) | High | 5+ | Async, loose coupling |

## Quick Decision Flow

```
Simple/Small Project? → Monolithic Single Module
Need Module Boundaries? → Monolithic Multi-Module / Layered
Complex Business Domain? → DDD
High Testability Needed? → Hexagonal / Clean
Heavy Read/Write Separation? → CQRS
Async & Loose Coupling? → Event-Driven
Multiple Teams & Scale? → Microservices
```

## Common Pattern Combinations

- **Microservices + DDD**: Each service owns a bounded context
- **DDD + Hexagonal**: Domain isolated from infrastructure
- **CQRS + Event-Driven**: Events sync read/write models
- **Clean + DDD**: Domain entities with Clean Architecture layers
