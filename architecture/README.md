# Java Backend Architecture Patterns

Essential architecture patterns for Java backend development. Each pattern includes when to use, structure, and key implementation skills.

## Pattern Selection Guide

| Pattern | Complexity | Team Size | Best For |
|---------|-----------|-----------|----------|
| [Monolithic Single Module](./patterns/monolithic-single-module.md) | Low | 1-3 | MVPs, small apps |
| [Monolithic Multi-Module](./patterns/monolithic-multi-module.md) | Medium | 3-10 | Medium apps, clear boundaries |
| [Layered Architecture](./patterns/layered-architecture.md) | Low-Medium | 2-8 | Traditional enterprise |
| [Microservices](./patterns/microservices-architecture.md) | High | 10+ | Large-scale, independent deploy |
| [DDD](./patterns/ddd-architecture.md) | High | 5+ | Complex business domains |
| [Hexagonal](./patterns/hexagonal-architecture.md) | Medium-High | 3-10 | Testable, infra-agnostic |
| [Clean Architecture](./patterns/clean-architecture.md) | High | 5+ | Long-term maintainability |
| [CQRS](./patterns/cqrs-architecture.md) | High | 5+ | Read/write optimization |
| [Event-Driven](./patterns/event-driven-architecture.md) | High | 5+ | Async, loose coupling |

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

## Evolvable Architecture

- 专题内容已迁移至 [evolvable-architecture](./evolvable-architecture/README.md)，涵盖模块爆炸规避、模块化部署、多端适配、微服务演进路径等。
