# 07 — Transactions

## 1. ACID refresher

| Property | Meaning |
|---|---|
| **Atomicity** | All statements succeed or none do |
| **Consistency** | Constraints hold before and after |
| **Isolation** | Concurrent transactions don't see each other's intermediate state (to a configurable degree) |
| **Durability** | Committed data survives crashes |

## 2. `@Transactional` basics

```java
@Service
@RequiredArgsConstructor
public class TransferService {
    private final AccountRepository accounts;

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accounts.findById(fromId).orElseThrow();
        Account to   = accounts.findById(toId).orElseThrow();
        from.withdraw(amount);              // throws InsufficientFundsException (unchecked)
        to.deposit(amount);
    }                                       // commit here (dirty checking flushes both UPDATEs)
}
```
Boot auto-configures a `PlatformTransactionManager` (`JpaTransactionManager` with JPA,
`DataSourceTransactionManager` with plain JDBC) and enables annotation-driven transactions; you
don't need `@EnableTransactionManagement`.

### How it works: proxies
```
caller ──▶ TransferService$$SpringCGLIB proxy ──▶ real TransferService.transfer()
             │ TransactionInterceptor:
             │  1. get/begin transaction (bind connection to thread)
             │  2. invoke method
             │  3. commit, or roll back on matching exception
```
The transaction is bound to the **current thread** (`TransactionSynchronizationManager` uses
`ThreadLocal`s). Code running in another thread (`@Async`, `CompletableFuture`, parallel streams)
is **not** part of it.

## 3. The self-invocation trap (most asked)

```java
@Service
public class OrderService {
    public void placeOrders(List<Order> orders) {
        for (Order o : orders) {
            saveOrder(o);                   // this.saveOrder() → bypasses the proxy → NO transaction
        }
    }

    @Transactional
    public void saveOrder(Order o) { ... }
}
```
Calls through `this` don't go through the proxy, so annotations (`@Transactional`, `@Async`,
`@Cacheable`, `@Retryable`) on the called method are ignored.

**Fixes:**
1. Move the method to a separate bean (cleanest).
2. Put `@Transactional` on the outer public method.
3. Self-inject the proxy (`@Lazy @Autowired OrderService self; self.saveOrder(o);`), a code smell.
4. Use `TransactionTemplate` programmatically.
5. AspectJ weaving mode (`mode = AdviceMode.ASPECTJ`), rarely used.

Other cases where `@Transactional` silently does nothing:
- The method is `private` or `final`, or the class is `final` (CGLIB can't override). Since Spring
  6, `protected` and package-private methods are supported with class-based proxies.
- The object wasn't created by Spring (`new OrderService()`).
- The exception is caught inside the method and not rethrown.
- Checked exception thrown without `rollbackFor`.

## 4. Rollback rules

By default, Spring rolls back on **unchecked exceptions** (`RuntimeException`, `Error`) and
**commits** on checked exceptions (EJB heritage: checked exceptions were seen as business outcomes).

```java
@Transactional(rollbackFor = Exception.class)                     // roll back on checked too
@Transactional(noRollbackFor = EmailFailedException.class)        // don't roll back for this one
```
Swallowed exceptions don't roll back:
```java
@Transactional
public void process() {
    try { repo.save(x); riskyCall(); }
    catch (Exception e) { log.error("failed", e); }   // commits anyway!
}
```
To roll back without throwing: `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.

**`UnexpectedRollbackException`:** an inner `REQUIRED` method throws, which marks the shared
transaction rollback-only; the outer method catches the exception and tries to commit → Spring
throws `UnexpectedRollbackException`. Use `REQUIRES_NEW` for the inner call if its failure should
be independent.

## 5. Propagation

What happens when a transactional method is called while a transaction may already exist?

| Propagation | Existing transaction | No transaction |
|---|---|---|
| `REQUIRED` (default) | Join it | Create a new one |
| `REQUIRES_NEW` | **Suspend** it, create a new independent one | Create a new one |
| `NESTED` | Savepoint inside the existing one (JDBC only, not JPA) | Create a new one |
| `SUPPORTS` | Join it | Run without a transaction |
| `NOT_SUPPORTED` | Suspend it, run without | Run without |
| `MANDATORY` | Join it | **Throw** `IllegalTransactionStateException` |
| `NEVER` | **Throw** | Run without |

```java
@Service
class OrderService {
    private final AuditService audit;

    @Transactional
    public void placeOrder(Order o) {
        repo.save(o);
        audit.log("order placed " + o.getId());   // separate bean, REQUIRES_NEW
        paymentClient.charge(o);                  // throws → order rolled back, audit row stays
    }
}

@Service
class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String msg) { auditRepo.save(new AuditEntry(msg)); }
}
```
Typical `REQUIRES_NEW` uses: audit logs, failure records, ID allocation, anything that must persist
even if the caller rolls back. Caveat: it needs a **second DB connection** while the first is held,
so heavy use can exhaust the pool (deadlock if pool size is 1).

`REQUIRES_NEW` vs `NESTED`: `REQUIRES_NEW` commits independently even if the outer rolls back
later. `NESTED` is a savepoint: inner rollback undoes only the inner work, but an outer rollback
undoes everything.

## 6. Isolation levels and read phenomena

| Phenomenon | Description |
|---|---|
| **Dirty read** | Reading another transaction's uncommitted changes |
| **Non-repeatable read** | Reading the same row twice returns different values (someone committed an update in between) |
| **Phantom read** | Re-running a query returns new rows (someone inserted matching rows) |
| **Lost update** | Two transactions read-modify-write the same row; one overwrites the other |

| Isolation | Dirty | Non-repeatable | Phantom |
|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible |
| `READ_COMMITTED` (Postgres, Oracle, SQL Server default) | Prevented | Possible | Possible |
| `REPEATABLE_READ` (MySQL InnoDB default) | Prevented | Prevented | Possible (InnoDB mostly prevents) |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```
`Isolation.DEFAULT` uses the database default. Higher isolation = more locking/aborts = lower
throughput. Prevent **lost updates** with optimistic locking (`@Version`) or
`SELECT ... FOR UPDATE`, not by cranking isolation.

## 7. Other attributes

```java
@Transactional(
    readOnly = true,          // hint: Hibernate skips dirty checking/flush; drivers may route to replicas
    timeout = 5,              // seconds; rolls back if exceeded
    transactionManager = "ordersTxManager")   // multiple data sources
```
A common pattern: `@Transactional(readOnly = true)` on the class, `@Transactional` on write methods
(method-level overrides class-level).

## 8. Programmatic transactions

```java
@Service
class ImportService {
    private final TransactionTemplate tx;
    ImportService(PlatformTransactionManager tm) {
        this.tx = new TransactionTemplate(tm);
        this.tx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    }

    void importAll(List<Row> rows) {
        for (List<Row> chunk : Lists.partition(rows, 500)) {
            tx.executeWithoutResult(status -> chunk.forEach(this::insert));   // one tx per chunk
        }
    }
}
```
Use it for fine-grained boundaries (batch chunks), or when you need a transaction inside a
self-invoked method.

## 9. Keep transactions short

Don't do remote HTTP calls, sleeps or long computations inside a transaction: it holds a DB
connection and row locks the whole time. Do I/O before or after, or use events:
```java
@Transactional
public void placeOrder(Order o) {
    repo.save(o);
    events.publishEvent(new OrderPlaced(o.getId()));
}

@Component
class OrderEmailer {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)   // only if commit succeeds
    void on(OrderPlaced e) { emailClient.sendConfirmation(e.orderId()); }
}
```

## 10. Distributed transactions

`@Transactional` covers **one resource** (one DB). Across DB + Kafka or multiple services:
- Avoid 2PC/XA (JTA with Atomikos/Narayana): slow, operationally complex, often unsupported.
- **Transactional outbox:** write the business row and an `outbox` row in the same DB transaction;
  a relay (poller or Debezium CDC) publishes outbox rows to the broker.
- **Saga:** a sequence of local transactions with compensating actions on failure, orchestrated
  (central coordinator) or choreographed (events).
- Make consumers **idempotent** (see [14](./14-messaging-kafka-rabbitmq.md)).

## 11. Gotchas

1. Self-invocation bypasses the proxy.
2. Checked exceptions commit by default.
3. Catching the exception inside the method → commit.
4. `@Transactional` on private methods or on a class created with `new`.
5. `@Async` + `@Transactional` on the same call chain: the async thread has no transaction from the caller.
6. `REQUIRES_NEW` in a loop can exhaust the connection pool.
7. `@Transactional` on a controller: keeps the transaction open during serialization; put it on services.
8. Mixing `jakarta.transaction.Transactional` and Spring's `@Transactional`: both work, but only
   Spring's supports `propagation`, `isolation`, `readOnly`, `timeout`.

## 12. Interview questions

1. **How does `@Transactional` work?**
   Spring wraps the bean in an AOP proxy. The `TransactionInterceptor` begins (or joins) a transaction
   via the `PlatformTransactionManager`, binds it to the thread, invokes the method, then commits or
   rolls back based on the outcome.

2. **Why doesn't `@Transactional` work when calling a method from the same class?**
   Self-invocation goes through `this`, not the proxy, so the interceptor never runs. Move the method
   to another bean or annotate the caller.

3. **Does `@Transactional` work on private methods?**
   No. Proxies can't intercept private (or final) methods. Spring 6 supports protected and
   package-private methods with CGLIB proxies.

4. **When does Spring roll back?**
   On unchecked exceptions and `Error`s by default; not on checked exceptions unless `rollbackFor`
   says so; not when the exception is caught.

5. **Explain propagation levels.**
   `REQUIRED` joins or creates; `REQUIRES_NEW` suspends and creates an independent one; `NESTED` uses
   a savepoint; `SUPPORTS` joins if present; `NOT_SUPPORTED` suspends; `MANDATORY` requires one;
   `NEVER` forbids one.

6. **`REQUIRED` vs `REQUIRES_NEW`?**
   `REQUIRED` shares one transaction: any failure rolls back everything. `REQUIRES_NEW` commits or
   rolls back independently of the caller (e.g. audit logs).

7. **What are isolation levels and which problems do they solve?**
   Read uncommitted, read committed, repeatable read, serializable; progressively preventing dirty
   reads, non-repeatable reads and phantom reads.

8. **What does `readOnly = true` do?**
   Hints that no writes happen: Hibernate sets flush mode to manual and skips dirty-check snapshots;
   some drivers/routers use it to send queries to a read replica. It doesn't block writes by itself
   on every database.

9. **What is `UnexpectedRollbackException`?**
   Thrown when an inner participating transaction marked the shared transaction rollback-only and the
   outer code tries to commit.

10. **Declarative vs programmatic transactions?**
    `@Transactional` vs `TransactionTemplate`/`PlatformTransactionManager` in code. Programmatic gives
    finer control (per-chunk commits).

11. **How do you handle transactions across microservices?**
    Avoid distributed 2PC; use sagas with compensating actions, the transactional outbox pattern and
    idempotent consumers for eventual consistency.

12. **How do you prevent lost updates?**
    Optimistic locking with `@Version` (retry or 409 on conflict), or pessimistic `SELECT FOR UPDATE`
    when contention is high.

13. **Where should `@Transactional` go: controller, service or repository?**
    Service layer, around a business use case. Spring Data repository methods are already
    transactional individually; a service method groups several calls into one unit.

14. **How do you run code only after a transaction commits?**
    `@TransactionalEventListener(phase = AFTER_COMMIT)` or `TransactionSynchronization.afterCommit`.

15. **Does `@Transactional` work with `@Async`?**
    Each async method runs on a different thread, so it doesn't join the caller's transaction. It can
    start its own if annotated with `@Transactional` itself.
