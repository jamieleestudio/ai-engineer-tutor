---
rule_id: "spring-data-jpa-guidelines"
title: "Spring Data JPA 最佳实践与规范"
language: "java"
framework: "spring-boot"
tags:
  - "jpa"
  - "database"
  - "orm"
  - "performance"
---

# Spring Data JPA 最佳实践与规范

本文档旨在指导 AI 及开发者在使用 Spring Data JPA 时遵循统一的规范，以确保代码的性能、可维护性和一致性。

## 1. 实体定义 (Entity Definition)

### 1.1 基本注解
- 所有实体类必须使用 `@Entity` 注解。
- 建议显式使用 `@Table` 指定表名，避免依赖默认命名策略导致的不确定性。
- 实现 `Serializable` 接口（视项目规范而定，通常推荐）。

### 1.2 主键策略
- 推荐使用代理主键（Surrogate Key），如 `Long` 或 `UUID`。
- 使用 `@Id` 和 `@GeneratedValue`。
- 对于 MySQL，通常使用 `GenerationType.IDENTITY`。
- 避免使用业务主键作为数据库主键。

### 1.3 字段映射
- 使用包装类型（如 `Integer`, `Long`, `Boolean`）而非基本类型，以支持 `null` 值（表示“未知”或“未设置”）。
- 显式使用 `@Column` 定义列属性，特别是 `nullable`, `length`, `unique` 等约束，保持代码与数据库定义一致。
- 对于枚举类型，必须使用 `@Enumerated(EnumType.STRING)`，避免使用索引值导致后续维护困难。

### 1.4 关联关系
- **默认抓取策略**：所有 `ToOne` 关系（`@ManyToOne`, `@OneToOne`）默认是 `EAGER`，建议显式设置为 `FetchType.LAZY` 以避免不必要的查询。
  ```java
  @ManyToOne(fetch = FetchType.LAZY)
  private User user;
  ```
- **集合关联**：`@OneToMany` 和 `@ManyToMany` 默认是 `LAZY`，保持默认即可。
- **级联操作**：谨慎使用 `CascadeType.ALL`，建议根据业务需求精确指定 `PERSIST`, `MERGE` 等。
- **双向关联**：必须在 `mappedBy` 一侧维护关系，并提供辅助方法（helper methods）来确保内存中对象状态的一致性。

### 1.5 `equals` 和 `hashCode`
- **不要**使用 Lombok 的 `@Data` 或 `@EqualsAndHashCode`（默认包含所有字段），这会导致循环引用和性能问题。
- 推荐基于 **业务键（Business Key）** 实现，如果没有业务键，可以使用 ID（需注意 ID 在持久化前可能为 null 的问题）或使用 `getClass()` 比较。
- 推荐使用 `@Getter` 和 `@Setter`（或 `@Data` 但手动重写 equals/hashcode）。

### 1.6 审计与乐观锁
- 核心业务实体应包含创建时间、更新时间。使用 `@EntityListeners(AuditingEntityListener.class)` 配合 `@CreatedDate` 和 `@LastModifiedDate`。
- 并发修改较高的实体应添加 `@Version` 字段实现乐观锁。

## 2. 仓储层 (Repository Layer)

### 2.1 接口定义
- 继承 `JpaRepository<Entity, ID>` 或 `ListCrudRepository` / `PagingAndSortingRepository`。
- 避免直接使用 `EntityManager`，除非有复杂的动态查询需求。

### 2.2 查询方法命名
- 简单查询优先使用 **Derived Query Methods**（方法名查询），如 `findByUsername`。
- 保持方法名简洁，如果过长，应转为 `@Query`。

### 2.3 自定义查询 (`@Query`)
- 复杂查询、多表关联必须使用 `@Query`。
- 优先使用 JPQL，必要时使用 Native SQL (`nativeQuery = true`)。
- **性能关键**：为了避免 N+1 问题，在查询关联对象时，必须使用 `JOIN FETCH`。
  ```java
  @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
  Optional<Order> findByIdWithItems(@Param("id") Long id);
  ```

### 2.4 投影 (Projections)
- 只读操作且不需要完整实体生命周期管理时，优先使用 **接口投影 (Interface-based Projections)** 或 **DTO 投影 (Class-based Projections)**。
- 这可以显著减少数据库查询字段和内存消耗。

## 3. 性能优化 (Performance)

### 3.1 解决 N+1 问题
- 使用 `@EntityGraph` 或 JPQL `JOIN FETCH` 预加载关联数据。
- 避免在循环中调用 Repository 方法。

### 3.2 分页与排序
- 查询列表时，必须考虑分页。使用 `Pageable` 接口作为参数，返回 `Page<T>` 或 `Slice<T>`。
- 避免返回大列表（`List<T>`），除非确定数据量极小。
- count 查询开销大，如果不需要总记录数，使用 `Slice<T>` 替代 `Page<T>`。

### 3.3 批量操作
- JPA 的 `saveAll` 在某些主键策略下可能不会通过 JDBC Batch 执行。
- 对于大批量插入/更新，考虑使用 `JdbcTemplate` 或专门的 Batch Insert 扩展库。
- 更新操作尽量使用 `@Modifying` + `@Query` 直接在数据库层面执行，而非先查后改（针对简单字段更新）。

## 4. 事务管理 (Transaction Management)

### 4.1 事务边界
- 事务注解 `@Transactional` 应加在 **Service 层（应用层）** 方法上，而非 Repository 层或 Controller 层。
- Repository 方法默认已有事务（`SimpleJpaRepository`），但业务逻辑通常需要跨多个 Repository 调用。

### 4.2 只读事务
- 对于纯查询业务，务必添加 `@Transactional(readOnly = true)`。这可以带来性能优化（如 Hibernate 不会执行脏检查）。

### 4.3 事务传播
- 默认使用 `Propagation.REQUIRED`。
- 谨慎使用 `REQUIRES_NEW`，以免造成死锁或连接池耗尽。

## 5. 规范示例

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY, cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();

    // 辅助方法
    public void addOrder(Order order) {
        orders.add(order);
        order.setUser(this);
    }
    
    // equals & hashCode implementation...
}

public interface UserRepository extends JpaRepository<User, Long> {
    
    // 方法名查询
    Optional<User> findByUsername(String username);

    // 解决 N+1 问题
    @Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> findByIdWithOrders(@Param("id") Long id);
    
    // DTO 投影
    @Query("SELECT new com.example.dto.UserSummary(u.id, u.username) FROM User u")
    List<UserSummary> findAllSummaries();
}
```

## 6. 常见反模式 (Anti-Patterns)
- **Open Session In View (OSIV)**: 建议在 `application.properties` 中关闭 `spring.jpa.open-in-view=false`。这强迫开发者在 Service 层处理完所有的数据加载（初始化 Lazy 集合），避免在 Controller 层触发数据库查询。
- **过度关联**: 不要为了方便而在实体间建立过多的双向关联，这会增加复杂度和死锁风险。
- **逻辑删除**: 如果使用逻辑删除，可以使用 `@SQLDelete` 和 `@Where`，但要注意这可能会影响原生查询和关联查询。

