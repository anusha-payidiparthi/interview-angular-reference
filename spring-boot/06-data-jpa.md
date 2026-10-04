# 06 — Spring Data JPA

## 1. The layers: JDBC, JPA, Hibernate, Spring Data JPA

| Layer | What it is |
|---|---|
| **JDBC** | Low-level Java API for SQL: connections, statements, result sets |
| **JPA** (Jakarta Persistence) | A **specification** for ORM: `@Entity`, `EntityManager`, JPQL |
| **Hibernate** | The most common JPA **implementation** (Boot's default) |
| **Spring Data JPA** | Repository abstraction on top of JPA: interfaces → generated implementations |

Also available: `JdbcTemplate`/`JdbcClient` (plain SQL), Spring Data JDBC (simpler aggregate
persistence, no lazy loading or dirty checking), jOOQ, MyBatis.

## 2. Setup

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: shop
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10        # HikariCP is the default pool
  jpa:
    hibernate:
      ddl-auto: validate           # none | validate | update | create | create-drop
    open-in-view: false            # see section 10
    properties:
      hibernate:
        jdbc.batch_size: 50
        order_inserts: true
    show-sql: false                # use logging.level.org.hibernate.SQL=DEBUG instead
```
`ddl-auto`: use `create-drop` only for embedded DB tests, `validate` or `none` in production, and
manage schema with **Flyway** or **Liquibase** migrations.

## 3. Entities

```java
@Entity
@Table(name = "orders", indexes = @Index(columnList = "customer_id"))
@EntityListeners(AuditingEntityListener.class)
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)   // DB auto-increment
    private Long id;

    @Enumerated(EnumType.STRING)                           // never ORDINAL (reordering breaks data)
    @Column(nullable = false, length = 20)
    private OrderStatus status;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();

    @Version                                               // optimistic locking
    private Long version;

    @CreatedDate  private Instant createdAt;
    @LastModifiedDate private Instant updatedAt;

    protected Order() {}                                   // JPA needs a no-arg constructor

    public void addLine(OrderLine line) {                  // keep both sides in sync
        lines.add(line);
        line.setOrder(this);
    }
}
```
Enable auditing with `@EnableJpaAuditing` on a config class (and an `AuditorAware<String>` bean for
`@CreatedBy`).

### ID generation strategies
| Strategy | How | Notes |
|---|---|---|
| `IDENTITY` | DB auto-increment column | Simple; **disables JDBC insert batching** in Hibernate |
| `SEQUENCE` | DB sequence (`allocationSize` pre-fetches IDs) | Best for Postgres/Oracle; batch-friendly |
| `TABLE` | Simulated sequence table | Slow; avoid |
| `AUTO` | Provider chooses | |
| `UUID` (`@UuidGenerator` / `GenerationType.UUID`) | Generated in app | Good for distributed systems; prefer time-ordered UUIDv7 for index locality |

### Records as entities?
No: entities need a no-arg constructor, mutable state and non-final classes for proxies. Records
are great for **DTOs and projections**.

## 4. Relationships

| Annotation | Example | Default fetch |
|---|---|---|
| `@OneToOne` | User ↔ Profile | **EAGER** |
| `@ManyToOne` | OrderLine → Order | **EAGER** |
| `@OneToMany` | Order → OrderLines | LAZY |
| `@ManyToMany` | Student ↔ Course | LAZY |

**Make every association `LAZY`** (set `fetch = FetchType.LAZY` on `@ManyToOne`/`@OneToOne`) and
fetch what each use case needs explicitly.

**Owning side:** the side with the foreign key (`@JoinColumn`, usually `@ManyToOne`). The other side
uses `mappedBy` and is ignored when writing. Updating only the inverse side doesn't persist the
relation.

**Cascade types:** `PERSIST`, `MERGE`, `REMOVE`, `REFRESH`, `DETACH`, `ALL`. Cascade from parent to
children it owns (Order → OrderLines), never from child to parent and rarely on `@ManyToMany`.
`orphanRemoval = true` deletes a child removed from the collection.

**`@ManyToMany`:** prefer an explicit join entity (`Enrollment` with extra columns like `grade`)
over a raw `@ManyToMany`; use `Set`, not `List`, to avoid inefficient delete-and-reinsert.

## 5. Repositories

```java
public interface OrderRepository extends JpaRepository<Order, Long>,
                                         JpaSpecificationExecutor<Order> {
    // derived queries: parsed from the method name
    List<Order> findByStatus(OrderStatus status);
    List<Order> findByCustomerIdAndStatusOrderByCreatedAtDesc(Long customerId, OrderStatus status);
    Optional<Order> findFirstByCustomerIdOrderByCreatedAtDesc(Long customerId);
    Page<Order> findByCreatedAtBetween(Instant from, Instant to, Pageable pageable);
    long countByStatus(OrderStatus status);
    boolean existsByCustomerIdAndStatus(Long customerId, OrderStatus status);
    List<Order> findTop10ByTotalGreaterThan(BigDecimal amount);

    // JPQL
    @Query("select o from Order o where o.customer.email = :email and o.total > :min")
    List<Order> findBigOrders(@Param("email") String email, @Param("min") BigDecimal min);

    // native SQL
    @Query(value = "select * from orders where created_at > now() - interval '1 day'",
           nativeQuery = true)
    List<Order> findRecent();

    // bulk update/delete: bypasses the persistence context
    @Modifying(clearAutomatically = true)
    @Transactional
    @Query("update Order o set o.status = :status where o.createdAt < :before")
    int expireOld(@Param("status") OrderStatus status, @Param("before") Instant before);

    // fetch associations in one query (fixes N+1)
    @EntityGraph(attributePaths = {"customer", "lines"})
    Optional<Order> findWithDetailsById(Long id);
}
```

### Repository hierarchy
```
Repository<T, ID>                         marker
  └─ CrudRepository                       save, findById, findAll, deleteById, count, existsById
       └─ ListCrudRepository              returns List instead of Iterable
  └─ PagingAndSortingRepository           findAll(Pageable), findAll(Sort)
       └─ JpaRepository                   flush, saveAndFlush, saveAll, deleteAllInBatch, getReferenceById
```
**How do interfaces work without implementations?** At startup Spring Data creates a JDK dynamic
proxy per repository interface (backed by `SimpleJpaRepository`) and parses method names into JPQL.
Invalid method names fail at startup.

### `save()`: persist or merge?
`SimpleJpaRepository.save` calls `em.persist` if the entity is new (null ID or null `@Version`),
otherwise `em.merge`. Merge does a `SELECT` first. With assigned IDs (e.g. UUIDs set in the
constructor) every save looks "not new" → extra select; implement `Persistable<ID>.isNew()` to fix.

### `findById` vs `getReferenceById`
`findById` hits the DB and returns `Optional`. `getReferenceById` (formerly `getOne`) returns a
lazy **proxy** without a query; accessing a field triggers the load, and it throws
`EntityNotFoundException` if the row doesn't exist. Useful for setting a foreign key:
`line.setProduct(productRepo.getReferenceById(productId))`.

## 6. Projections (fetch only what you need)

```java
// Interface-based projection
public interface OrderSummary {
    Long getId();
    OrderStatus getStatus();
    @Value("#{target.customer.name}") String getCustomerName();  // open projection (less efficient)
}
List<OrderSummary> findByStatus(OrderStatus status);

// DTO / record projection (class-based)
public record OrderRow(Long id, BigDecimal total, String customerName) {}

@Query("select new com.example.OrderRow(o.id, o.total, c.name) from Order o join o.customer c")
List<OrderRow> findRows();

// Dynamic projection
<T> List<T> findByCustomerId(Long customerId, Class<T> type);
```
Projections select only needed columns, return read-only objects and avoid dirty checking:
ideal for read endpoints.

## 7. Dynamic queries

```java
// Specifications (Criteria API)
public final class OrderSpecs {
    static Specification<Order> hasStatus(OrderStatus s) {
        return (root, query, cb) -> s == null ? null : cb.equal(root.get("status"), s);
    }
    static Specification<Order> totalAtLeast(BigDecimal min) {
        return (root, query, cb) -> min == null ? null : cb.ge(root.get("total"), min);
    }
}
repo.findAll(where(hasStatus(status)).and(totalAtLeast(min)), pageable);

// Query by Example
Order probe = new Order(); probe.setStatus(OrderStatus.NEW);
repo.findAll(Example.of(probe));
```
Other options: Querydsl, jOOQ, or `JdbcClient` for complex reporting SQL.

## 8. The N+1 select problem

```java
List<Order> orders = orderRepo.findAll();           // 1 query
for (Order o : orders) {
    o.getCustomer().getName();                       // +1 query PER order (lazy load)
}
// 100 orders → 101 queries
```
**Detect:** enable SQL logging (`logging.level.org.hibernate.SQL=DEBUG`), Hibernate statistics,
or test-time query counting (datasource-proxy).

**Fix:**
```java
// 1. JOIN FETCH
@Query("select o from Order o join fetch o.customer where o.status = :status")
List<Order> findWithCustomer(OrderStatus status);

// 2. Entity graph
@EntityGraph(attributePaths = "customer")
List<Order> findByStatus(OrderStatus status);

// 3. Batch fetching: loads lazy associations in IN (...) batches
spring.jpa.properties.hibernate.default_batch_fetch_size=50

// 4. DTO projection with a join: selects exactly the columns needed
```
**Pitfall:** `JOIN FETCH` of a collection + pagination → Hibernate paginates **in memory** (warning
`HHH90003004`) or fails. Instead page over IDs first, then fetch by IDs with join fetch, or use batch
fetching. Fetching two `List` collections at once → `MultipleBagFetchException` (use `Set` or
separate queries).

## 9. Persistence context, entity states and dirty checking

```
new Order()                       TRANSIENT   (not tracked)
  em.persist / repo.save   ───▶   MANAGED     (tracked in the persistence context)
  transaction ends / clear ───▶   DETACHED    (no longer tracked; changes ignored)
  em.remove                ───▶   REMOVED
```
**Dirty checking:** inside a transaction, Hibernate snapshots managed entities and at flush/commit
issues `UPDATE`s for changed fields. **You don't need to call `save()` to update a managed entity:**
```java
@Transactional
public void rename(Long id, String name) {
    Product p = repo.findById(id).orElseThrow();
    p.setName(name);                 // UPDATE happens at commit
}
```
The persistence context is also a **first-level cache**: `findById(1)` twice in the same
transaction runs one query. The **second-level cache** (Ehcache/Caffeine/Infinispan via JCache) is
shared across sessions and opt-in per entity with `@Cacheable`/`@Cache`.

## 10. `LazyInitializationException` and Open Session In View

```java
Order o = orderService.find(id);    // transaction closed here
o.getLines().size();                // LazyInitializationException: no Session
```
Fixes: fetch what you need inside the transaction (join fetch / entity graph / DTO projection), or
map to a DTO inside the `@Transactional` service method.

**Open Session In View (OSIV)** (`spring.jpa.open-in-view`, **true by default**, logs a warning)
keeps the session open until the view/JSON is rendered so lazy loads work in controllers. It hides
N+1 problems and holds a DB connection for the whole request. **Set it to `false`** for REST APIs.

## 11. Locking

```java
// Optimistic: @Version column; UPDATE ... WHERE id=? AND version=?
// 0 rows updated → ObjectOptimisticLockingFailureException → return 409 or retry
@Version private Long version;

// Pessimistic: SELECT ... FOR UPDATE
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select a from Account a where a.id = :id")
Optional<Account> findForUpdate(Long id);
```
Optimistic for low-contention edits (most web apps); pessimistic when conflicts are frequent and
retries are expensive (seat booking, balances). See [07](./07-transactions.md).

## 12. Batch inserts and performance

- `saveAll` + `hibernate.jdbc.batch_size` + `order_inserts` (doesn't work with `IDENTITY` IDs).
- For huge loads, flush and clear every N entities, or use `JdbcTemplate.batchUpdate` / `COPY`.
- `@Transactional(readOnly = true)` on reads: Hibernate skips dirty-check snapshots.
- Index foreign keys and columns used in `WHERE`/`ORDER BY`.
- Stream large results: `Stream<Order> streamAllBy()` inside a transaction (close the stream).

## 13. `JdbcClient` / `JdbcTemplate` (when JPA is overkill)

```java
@Repository
class ReportDao {
    private final JdbcClient jdbc;            // Spring 6.1+ fluent API
    ReportDao(JdbcClient jdbc) { this.jdbc = jdbc; }

    List<SalesRow> salesByDay(LocalDate from) {
        return jdbc.sql("""
                select date(created_at) as day, sum(total) as revenue
                from orders where created_at >= :from group by day order by day
                """)
            .param("from", from)
            .query(SalesRow.class)            // maps columns to record components
            .list();
    }
}
```

## 14. Database migrations (Flyway)

```
src/main/resources/db/migration/
  V1__create_orders.sql
  V2__add_status_index.sql
```
Add `flyway-core` (+ `flyway-database-postgresql`); Boot runs pending migrations at startup and
records them in `flyway_schema_history`. Never edit an applied migration; add a new one. Liquibase
is the alternative (XML/YAML/SQL changelogs, rollbacks).

## 15. Gotchas

1. `EAGER` on `@ManyToOne`/`@OneToOne` by default → hidden extra queries everywhere.
2. `equals`/`hashCode` on entities using all fields or a generated ID that is null before persist →
   broken `Set`s. Use a business key, or ID-based equality that treats null IDs as unequal and a
   constant `hashCode`.
3. Lombok `@Data`/`@ToString` on entities → lazy loading and infinite recursion in `toString`.
4. `@Transactional` missing on a `@Modifying` query → `TransactionRequiredException`.
5. Bulk `@Modifying` updates bypass the persistence context → stale entities unless cleared.
6. `ddl-auto=update` in production → unpredictable schema drift.
7. `CascadeType.REMOVE` on `@ManyToMany` deletes shared entities.
8. Calling `findAll()` on large tables without paging.

## 16. Interview questions

1. **JPA vs Hibernate vs Spring Data JPA?**
   JPA is the specification, Hibernate an implementation, Spring Data JPA a repository abstraction
   that generates data-access code on top of JPA.

2. **`CrudRepository` vs `JpaRepository`?**
   `JpaRepository` extends paging/sorting and CRUD repositories and adds JPA-specific methods
   (`flush`, `saveAndFlush`, batch deletes, `getReferenceById`), returning `List`s.

3. **How do derived query methods work?**
   Spring Data parses the method name (`findBy` + properties + operators like `And`, `Between`,
   `OrderBy`) into a JPQL query at startup; invalid names fail startup.

4. **How do you write custom queries?**
   `@Query` with JPQL or `nativeQuery = true`, `@Modifying` for updates/deletes, Specifications or
   Querydsl for dynamic ones, or a custom repository fragment implementation.

5. **What is the N+1 problem and how do you fix it?**
   One query for the parent list plus one per row for a lazy association. Fix with `JOIN FETCH`,
   `@EntityGraph`, batch fetching, or DTO projections.

6. **Lazy vs eager loading?**
   Lazy loads associations on first access (proxy); eager loads them immediately. Default: `ToOne` is
   eager, `ToMany` is lazy. Best practice: everything lazy, fetch explicitly per use case.

7. **What is `LazyInitializationException`?**
   Accessing an uninitialized lazy association after the persistence context is closed. Fetch it in
   the transaction or use a DTO; don't rely on OSIV or `EAGER`.

8. **What is Open Session In View? Should you disable it?**
   Keeps the Hibernate session open for the whole web request. Disable it for APIs: it hides N+1
   problems and holds DB connections longer.

9. **What are entity states?**
   Transient, managed (persistent), detached, removed.

10. **What is dirty checking?**
    Hibernate tracks managed entities and automatically writes changes at flush/commit; no explicit
    `save` needed inside a transaction.

11. **`save()` vs `saveAndFlush()`?**
    `save` makes the entity managed; SQL runs at flush/commit. `saveAndFlush` forces SQL immediately
    (e.g. to catch constraint violations right away).

12. **`persist` vs `merge`?**
    `persist` makes a new instance managed. `merge` copies a detached instance's state onto a managed
    copy (loading it first) and returns that copy.

13. **`findById` vs `getReferenceById`?**
    Immediate query returning `Optional` vs a lazy proxy without a query.

14. **What is a projection?**
    A query result shaped as an interface or DTO containing only selected columns, used for
    efficient read-only queries.

15. **First-level vs second-level cache?**
    First-level: per persistence context (transaction), always on. Second-level: shared across
    sessions, opt-in, needs a cache provider.

16. **Optimistic vs pessimistic locking?**
    Optimistic uses a `@Version` column and fails on conflicting updates (no DB locks). Pessimistic
    takes DB row locks (`SELECT ... FOR UPDATE`) to block concurrent writers.

17. **What does `@Modifying` do?**
    Marks a `@Query` as an `UPDATE`/`DELETE`; needs a transaction; `clearAutomatically` clears stale
    entities from the persistence context.

18. **Which ID generation strategy do you use?**
    `SEQUENCE` with allocation size for performance and batching on Postgres/Oracle; `IDENTITY` for
    MySQL simplicity; UUIDs (v7) when IDs must be generated outside the DB.

19. **What are `cascade` and `orphanRemoval`?**
    Cascade propagates operations (persist, remove…) from parent to children; `orphanRemoval` deletes
    children removed from the parent's collection.

20. **How do you manage schema changes?**
    Versioned migrations with Flyway or Liquibase run at startup or in the pipeline; `ddl-auto` set to
    `validate`/`none`.

21. **How do you speed up bulk inserts?**
    JDBC batching (`batch_size`, `order_inserts`, sequence IDs), flush/clear periodically, or drop
    to `JdbcTemplate.batchUpdate`/`COPY`.
