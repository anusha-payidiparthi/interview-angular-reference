# 14 — Messaging: Kafka & RabbitMQ

## 1. Why messaging?

- **Decoupling:** producers don't know consumers; services can be down temporarily.
- **Load leveling:** queue absorbs spikes; consumers process at their own pace.
- **Fan-out:** one event, many independent consumers.
- **Event-driven architecture:** state changes broadcast as events (event notification, event-carried
  state transfer, event sourcing).

## 2. Kafka vs RabbitMQ

| | Apache Kafka | RabbitMQ |
|---|---|---|
| Model | Distributed, partitioned, replicated **log** | **Message broker** with exchanges and queues |
| Consumption | Consumers pull and track **offsets**; messages stay after reading (retention) | Broker pushes; message **removed** after ack |
| Replay | Yes (reset offsets) | No (once consumed, gone) |
| Ordering | Per partition | Per queue (single consumer) |
| Throughput | Very high (millions/s) | High (tens of thousands/s per queue) |
| Routing | Topic + key → partition | Flexible: direct, topic (wildcards), fanout, headers exchanges |
| Best for | Event streaming, logs, analytics, event sourcing, CDC | Task queues, RPC-style work distribution, complex routing, per-message TTL/priority |

## 3. Kafka concepts

```
Topic "orders" (3 partitions, replication factor 3)
  P0: [m0][m1][m2][m3] ...  ← offsets
  P1: [m0][m1] ...
  P2: [m0][m1][m2] ...

Consumer group "billing": C1 ← P0, P1    C2 ← P2
Consumer group "shipping": C3 ← P0, P1, P2   (independent offsets: each group gets every message)
```
- **Partition**: unit of parallelism and ordering. Messages with the same **key** go to the same
  partition → ordered per key (e.g. key = orderId).
- **Consumer group**: each partition is consumed by exactly one consumer in the group. Max useful
  consumers = number of partitions.
- **Offset**: position of a consumer group in a partition; committed after processing.
- **Rebalance**: partitions reassigned when consumers join/leave.
- **Replication**: leader + followers; `acks=all` + `min.insync.replicas=2` for durability.
- KRaft mode replaced ZooKeeper (Kafka 4.0 removed ZooKeeper).

## 4. Spring for Apache Kafka

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      properties:
        enable.idempotence: true
    consumer:
      group-id: billing
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: com.example.events
    listener:
      ack-mode: record          # commit after each record is processed
      concurrency: 3            # 3 consumer threads
```

### Producer
```java
public record OrderPlaced(String orderId, String customerId, BigDecimal total) {}

@Service
class OrderEventsPublisher {
    private final KafkaTemplate<String, OrderPlaced> kafka;
    OrderEventsPublisher(KafkaTemplate<String, OrderPlaced> kafka) { this.kafka = kafka; }

    void publish(OrderPlaced e) {
        kafka.send("orders.placed", e.orderId(), e)          // key = orderId → ordering per order
             .whenComplete((result, ex) -> {
                 if (ex != null) log.error("Publish failed for {}", e.orderId(), ex);
                 else log.debug("Sent to partition {} offset {}",
                         result.getRecordMetadata().partition(), result.getRecordMetadata().offset());
             });
    }
}

@Bean
NewTopic ordersPlaced() {                                     // auto-create topic via KafkaAdmin
    return TopicBuilder.name("orders.placed").partitions(6).replicas(3).build();
}
```

### Consumer with retries and a dead-letter topic
```java
@Component
class BillingListener {

    @KafkaListener(topics = "orders.placed", groupId = "billing")
    void on(OrderPlaced event, @Header(KafkaHeaders.RECEIVED_KEY) String key) {
        billing.invoice(event);                 // throw → error handler retries
    }
}

@Configuration
class KafkaErrorConfig {
    @Bean
    DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
        var recoverer = new DeadLetterPublishingRecoverer(template);       // → orders.placed.DLT
        var backoff = new ExponentialBackOffWithMaxRetries(3);
        backoff.setInitialInterval(500);
        backoff.setMultiplier(2);
        var handler = new DefaultErrorHandler(recoverer, backoff);
        handler.addNotRetryableExceptions(ValidationException.class);      // poison messages: no retry
        return handler;
    }
}
```
Non-blocking retries (separate retry topics so one bad message doesn't block the partition):
```java
@RetryableTopic(attempts = "4", backoff = @Backoff(delay = 1000, multiplier = 2))
@KafkaListener(topics = "orders.placed")
void on(OrderPlaced e) { ... }

@DltHandler
void dlt(OrderPlaced e, @Header(KafkaHeaders.EXCEPTION_MESSAGE) String error) { alert(e, error); }
```

### Delivery semantics
| Semantics | How | Risk |
|---|---|---|
| At most once | Commit offset **before** processing | Message loss on crash |
| **At least once** (default) | Commit **after** processing | Duplicates on crash/rebalance → consumers must be idempotent |
| Exactly once | Idempotent producer + Kafka transactions (`transactional.id`, `isolation.level=read_committed`) | Only within Kafka (read-process-write); external side effects still need idempotency |

## 5. Idempotent consumers

Because at-least-once delivery means duplicates *will* happen:
```java
@Transactional
@KafkaListener(topics = "payments.completed")
void on(PaymentCompleted e) {
    if (processedRepo.existsById(e.eventId())) return;       // already handled
    orderService.markPaid(e.orderId());
    processedRepo.save(new ProcessedEvent(e.eventId()));     // same DB transaction
}
```
Alternatives: natural idempotency (`UPDATE ... SET status='PAID' WHERE id=? AND status='NEW'`),
unique constraints, upserts.

## 6. Transactional outbox

Problem: "save order to DB **and** publish event" can't be atomic across two systems. If the app
crashes between them, you lose the event or publish an event for a rolled-back order.

```java
@Transactional
public Order place(CreateOrderRequest req) {
    Order order = orderRepo.save(new Order(req));
    outboxRepo.save(new OutboxEvent("Order", order.getId(), "OrderPlaced", toJson(order)));
    return order;                 // both rows commit or neither does
}

// Relay: poll and publish (or use Debezium CDC to stream the outbox table to Kafka)
@Scheduled(fixedDelay = 1000)
@SchedulerLock(name = "outboxRelay")
@Transactional
public void relay() {
    for (OutboxEvent e : outboxRepo.findTop100ByPublishedFalseOrderByIdAsc()) {
        kafka.send(e.topic(), e.aggregateId(), e.payload()).join();
        e.markPublished();
    }
}
```
Delivery is at least once, so consumers stay idempotent. Spring Modulith offers an
event-externalization feature built on its persisted event registry.

## 7. RabbitMQ with Spring AMQP

```
Producer → Exchange ──(binding: routing key / pattern)──▶ Queue → Consumer
             types: direct (exact key), topic (order.*.created), fanout (broadcast), headers
```
```java
@Configuration
class RabbitConfig {
    @Bean TopicExchange ordersExchange() { return new TopicExchange("orders"); }

    @Bean Queue emailQueue() {
        return QueueBuilder.durable("orders.email")
                .deadLetterExchange("orders.dlx")          // failed messages → DLX
                .build();
    }

    @Bean Binding emailBinding() {
        return BindingBuilder.bind(emailQueue()).to(ordersExchange()).with("order.*.placed");
    }

    @Bean Jackson2JsonMessageConverter converter() { return new Jackson2JsonMessageConverter(); }
}

@Service
class Publisher {
    private final RabbitTemplate rabbit;
    Publisher(RabbitTemplate rabbit) { this.rabbit = rabbit; }
    void publish(OrderPlaced e) { rabbit.convertAndSend("orders", "order.eu.placed", e); }
}

@Component
class EmailConsumer {
    @RabbitListener(queues = "orders.email", concurrency = "3-10")
    void handle(OrderPlaced e) { email.send(e); }    // exception → retry/requeue or DLQ per config
}
```
```yaml
spring.rabbitmq.listener.simple:
  acknowledge-mode: auto
  retry:
    enabled: true
    max-attempts: 3
    initial-interval: 1s
  default-requeue-rejected: false   # after retries, reject → dead-letter instead of infinite requeue
```
Reliability: durable exchanges/queues, persistent messages, **publisher confirms**, consumer acks,
DLQs.

## 8. Spring Cloud Stream (abstraction)
Write `java.util.function` beans; a **binder** (Kafka, Rabbit) connects them to destinations.
```java
@Bean
Function<OrderPlaced, Invoice> invoice() { return order -> billing.createInvoice(order); }
// spring.cloud.stream.bindings.invoice-in-0.destination=orders.placed
// spring.cloud.stream.bindings.invoice-out-0.destination=invoices.created
```
Pros: broker-agnostic, less boilerplate. Cons: another abstraction layer; broker-specific features
need binder config.

## 9. Gotchas

1. Ordering assumed across partitions (only guaranteed **within** a partition).
2. More consumers than partitions → idle consumers.
3. Non-idempotent consumers + at-least-once → double charging.
4. A poison message retried forever blocks the partition → configure DLT and non-retryable exceptions.
5. Long processing exceeds `max.poll.interval.ms` → consumer kicked out → rebalance → reprocessing.
6. `JsonDeserializer` trusted packages not set → deserialization errors.
7. Publishing to Kafka inside a DB transaction and assuming atomicity → use outbox.
8. Changing event schemas incompatibly → use a schema registry (Avro/Protobuf) or additive changes.

## 10. Interview questions

1. **Kafka vs RabbitMQ?**
   Kafka is a durable partitioned log with consumer-tracked offsets, replay and very high
   throughput; good for event streaming. RabbitMQ is a broker with flexible routing and per-message
   acks; good for task queues and complex routing.

2. **What are partitions and consumer groups?**
   Partitions split a topic for parallelism and ordering; a consumer group shares partitions so each
   message is processed once per group, and different groups each get all messages.

3. **How is ordering guaranteed in Kafka?**
   Only within a partition; use a message key (e.g. order ID) so related messages go to the same
   partition.

4. **At-least-once vs exactly-once?**
   At-least-once commits offsets after processing (duplicates possible). Exactly-once uses idempotent
   producers and Kafka transactions for read-process-write within Kafka; external side effects still
   need idempotent handling.

5. **How do you handle failed messages?**
   Retry with backoff (blocking `DefaultErrorHandler` or non-blocking `@RetryableTopic`), then send
   to a dead-letter topic/queue for inspection and replay; don't retry non-transient errors.

6. **What is an idempotent consumer?**
   A consumer that produces the same result when processing the same message more than once (dedupe
   by event ID, conditional updates, upserts).

7. **What is the transactional outbox pattern?**
   Write the event to an outbox table in the same DB transaction as the business change, and publish
   it asynchronously with a relay or CDC (Debezium).

8. **How do you consume Kafka messages in Spring?**
   `@KafkaListener` on a bean method with consumer config in `spring.kafka.consumer.*`; concurrency
   via `listener.concurrency`; error handling via `DefaultErrorHandler`.

9. **How do you send messages in Spring?**
   `KafkaTemplate.send(topic, key, value)` (returns `CompletableFuture<SendResult>`) or
   `RabbitTemplate.convertAndSend(exchange, routingKey, payload)`.

10. **What is a rebalance and why does it matter?**
    Reassigning partitions when group membership changes; processing pauses and uncommitted messages
    are redelivered, so keep processing fast and consumers idempotent.

11. **RabbitMQ exchange types?**
    Direct (exact routing key), topic (pattern with `*`/`#`), fanout (broadcast to all bound queues),
    headers (match on headers).

12. **How do you evolve message schemas?**
    Backward-compatible changes (add optional fields), schema registry with compatibility rules
    (Avro/Protobuf/JSON Schema), versioned event types.
