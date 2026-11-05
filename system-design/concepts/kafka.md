# Kafka Reference: Concepts, Spring Boot, and AWS

Practical guide to Kafka fundamentals, consumer reliability, Spring Kafka, and an AWS DMS-to-MSK pipeline. Examples are illustrative; check behavior against the Kafka and Spring Kafka versions deployed by your service.

## 1. Mental model

Kafka is a durable, distributed event log.

```text
Producer → Topic → Partitions → Consumer group → Service
```

- A **producer** publishes records.
- A **topic** is a named stream of related records.
- A **partition** is an ordered log inside a topic.
- A **consumer** reads records.
- A **consumer group** is one logical application whose consumers share the work.
- An **offset** is a consumer's position within a partition.

Kafka records are retained by a configured time/size policy or compacted by key; reading a record does not delete it. With compaction, tombstone records represent deletes and are retained only according to the configured cleanup policy.

## 2. Core vocabulary

| Concept | Meaning |
|---|---|
| Broker | Kafka server that stores and serves records |
| Topic | Named stream, such as `orders.events` |
| Partition | Ordered, append-only log within a topic |
| Record | Producer sends key, value, timestamp, and headers; Kafka assigns a partition and offset when it appends the record |
| Key | Determines partition placement and often preserves entity ordering |
| Offset | Position of a record within one partition |
| Leader | Broker that handles reads and writes for a partition |
| Follower | Replica that copies a partition from its leader |
| Consumer group | Consumers that share partition assignments |
| Retention | Time/size policy for keeping records |
| Compaction | Policy that keeps the latest record for each key |
| Consumer lag | Per-partition offset distance between the log end and the group's committed position; on compacted topics, offset distance is not an exact count of remaining records |

## 3. How a record reaches a partition

The producer normally chooses the partition before sending the record to the partition leader.

```text
producer.send(topic, key, value)
        ↓
producer obtains topic metadata
        ↓
partition is selected
        ↓
record is sent to that partition's leader
        ↓
leader stores it and replicas copy it
```

Partition selection:

1. **Explicit partition**

   ```java
   new ProducerRecord<>("orders.events", 2, "order-123", event);
   ```

2. **Key provided**: the partitioner maps the key to a partition (the default partitioner normally hashes the serialized key).

   ```text
   key=order-123, value=OrderCreated → P2
   key=order-123, value=OrderPaid    → P2
   key=order-123, value=OrderShipped → P2
   ```

   This preserves ordering for that key while it continues to map to the same partition. Increasing the partition count can change the default key-to-partition mapping for future records; old and new events for one key may then live in different partitions. If strict long-term per-key ordering matters, plan partition counts and partitioner changes carefully.

3. **No key**: the producer uses its default partitioning strategy to distribute records, typically sticking to a partition while it fills a batch, then changing partitions.

Use a business key such as `order_id`, `customer_id`, or `product_id` when related events must remain ordered. Avoid hardcoding partition numbers unless you have a specialized routing or replay requirement.

## 4. Partitions and ordering

Kafka guarantees ordering **within one partition**, not across a whole topic.

```text
orders.events
├── P0: offset 0, 1, 2, 3...
├── P1: offset 0, 1, 2, 3...
└── P2: offset 0, 1, 2, 3...
```

If a topic has three partitions, at most three consumers in one classic consumer group can actively process records at the same time. Extra consumers remain idle.

### Choosing partition count

Use load testing to measure realistic per-partition throughput.

```text
partitions_by_throughput = ceil(peak topic throughput / tested throughput per partition)
partitions_by_consumers   = required consumer instances

partitions = max(partitions_by_throughput, partitions_by_consumers)
```

Example:

```text
Peak traffic:             40 MB/s
Tested per-partition rate: 5 MB/s
Consumer parallelism:       6

40 / 5 = 8
Choose at least 8 partitions, with reasonable headroom.
```

Watch for **hot partitions**: if one key produces most of the traffic, adding partitions may not help because the same key still maps to one partition.

More partitions improve parallelism but increase metadata, rebalancing, replication, and recovery overhead. Increasing partitions later can change key-to-partition mapping, so plan carefully when ordering matters.

### Replication and producer durability

Partitions are replicated across brokers for availability. The leader handles writes; followers replicate the log. The producer's `acks` setting controls when a send is considered successful:

| `acks` | Producer considers the write successful when… |
|---|---|
| `0` | It does not wait for a broker acknowledgment; fastest, with highest loss risk |
| `1` | The leader acknowledges; a leader failure before replication can lose the record |
| `all` (`-1`) | All current in-sync replicas acknowledge; strongest normal durability setting |

`acks=all` means all **in-sync** replicas, not every configured replica. `min.insync.replicas` is a broker/topic setting that defines how many in-sync replicas must be available for an `acks=all` write to succeed. A common illustrative setup is replication factor 3 and minimum ISR 2, but choose values based on failure and availability goals.

For modern Kafka clients, idempotence is generally enabled by default unless conflicting settings disable it. It deduplicates producer retries within Kafka's producer protocol; it does not deduplicate the same business event submitted twice as separate sends. Check the deployed client version and effective config. See the [Kafka producer configuration](https://kafka.apache.org/40/configuration/producer-configs/) for `acks` and idempotence constraints.

Example Spring Boot producer settings:

~~~yaml
spring:
  kafka:
    producer:
      acks: all
      properties:
        enable.idempotence: true
~~~

## 5. Consumer groups and scaling

Consumers with the same group ID share work:

```text
fulfillment-service group
├── Consumer A → P0
├── Consumer B → P1
└── Consumer C → P2
```

Different group IDs each receive their own copy of the topic:

```text
fulfillment-service → processes orders
analytics-service    → processes orders
notifications-service → processes orders
```

In Spring Boot:

```yaml
spring:
  kafka:
    consumer:
      group-id: fulfillment-service
```

The number of consumers can come from application replicas and listener concurrency:

```text
total consumer threads = application instances × concurrency
```

Example: three application replicas with `concurrency = 2` create up to six consumers in the same group.

Kafka assigns each partition to at most one consumer in a classic consumer group at a time. Adding/removing consumers or partitions can trigger a rebalance and briefly pause or move work. If there are more consumer threads than partitions, some threads stay idle. Keep processing time, polling, and offset commits within the consumer's configured limits so a slow handler does not repeatedly lose its assignment.

## 6. Offsets and delivery semantics

The consumer's current position and its committed position are separate. Configuration determines when offsets are committed. For reliable processing, applications commonly disable auto-commit and commit only after the record or batch succeeds. On restart, the consumer resumes from its committed position; if the business side effect succeeded but the offset did not commit, the record can be delivered again.

Common delivery semantics (from a consumer's perspective):

| Semantic | Meaning |
|---|---|
| At-most-once | May lose records, but does not intentionally redeliver |
| At-least-once | Does not intentionally lose records, but duplicates can occur |
| Exactly-once | Requires transactions and cooperation from the systems involved |

At-least-once is a common practical choice. Design consumers to tolerate duplicates. Producer idempotence prevents certain retry duplicates at the broker; it does not make a consumer's database update or external API call exactly-once.

Kafka transactions can provide exactly-once processing when consuming from Kafka and producing to Kafka, if output records and consumed offsets are committed in the same transaction and downstream consumers use `read_committed`. They do not automatically make an arbitrary database write, payment, or email exactly-once. For external systems, use an atomic integration where available or idempotency/inbox/outbox patterns. See [Kafka delivery semantics](https://kafka.apache.org/43/design/design/).

### Spring Kafka acknowledgment modes

| `ackMode` | Commit behavior |
|---|---|
| `RECORD` | After each record returns successfully |
| `BATCH` | After all records from one poll are processed |
| `TIME` | After processing and the configured time threshold |
| `COUNT` | After the configured record count |
| `COUNT_TIME` | When count or time threshold is reached |
| `MANUAL` | Listener calls `acknowledge()`; generally batch-style |
| `MANUAL_IMMEDIATE` | Listener acknowledgment commits immediately when called on the consumer thread |

`ackMode` controls **when** an offset is ready to commit. `syncCommits` controls whether the commit call is synchronous or asynchronous.

Keep Kafka auto-commit disabled when the Spring listener container manages acknowledgments. Spring Kafka 4.1 added the `ackMode` attribute directly to `@KafkaListener`; with earlier versions, set the mode on the listener container factory's `ContainerProperties`. `MANUAL_IMMEDIATE` only commits immediately when acknowledgment is called on the consumer thread. Check the [Spring Kafka listener documentation](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/listener-annotation.html) for the deployed release.

For database-backed processing, a common pattern is:

```text
read record
→ commit business transaction
→ acknowledge Kafka record
```

## 7. Idempotent consumers

Use a stable `event_id` in every event and store processed IDs with a database unique constraint.

```sql
CREATE TABLE processed_events (
    consumer_name VARCHAR(100),
    event_id      VARCHAR(100),
    processed_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (consumer_name, event_id)
);
```

Processing pattern:

```text
BEGIN database transaction
  insert (consumer_name, event_id)
  if duplicate: skip business operation
  otherwise: apply business update
COMMIT database transaction
acknowledge Kafka record
```

If the service crashes after the database commit but before the Kafka offset commit, the message is retried. The unique event ID prevents the business update from running twice.

For external APIs such as payments or email, pass the same idempotency key downstream or persist an outbox/inbox record before performing the side effect.

## 8. Spring Boot consumer template

### Dependency

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### Configuration

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: fulfillment-service
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.value.default.type: com.example.kafka.OrderCreated
        spring.json.trusted.packages: com.example.kafka
        spring.json.use.type.headers: false
```

`auto-offset-reset: earliest` applies only when the group has no valid committed offset (or its offset is no longer available); it does not rewind a group that already has committed offsets. For production, use a schema strategy such as a schema registry or explicitly versioned JSON/Avro/Protobuf contracts, and configure deserialization/error handling deliberately.

### Event and listener

```java
public record OrderCreated(
        String eventId,
        String orderId,
        BigDecimal total
) {}
```

```java
@Component
public class OrderConsumer {

    private final OrderService orderService;

    public OrderConsumer(OrderService orderService) {
        this.orderService = orderService;
    }

    @KafkaListener(
            id = "orders-listener",
            topics = "orders.events",
            groupId = "fulfillment-service",
            concurrency = "3",
            ackMode = "MANUAL_IMMEDIATE"
    )
    public void consume(
            OrderCreated event,
            Acknowledgment acknowledgment) {

        orderService.processIdempotently(event);
        acknowledgment.acknowledge();
    }
}
```

### Common `@KafkaListener` attributes

| Attribute | Use |
|---|---|
| `id` | Unique Spring listener-container ID; useful for start/stop/control |
| `topics` | Fixed topic names |
| `topicPattern` | Regex-based topic subscription |
| `topicPartitions` | Explicit partition assignment; special-case usage |
| `groupId` | Kafka consumer group for this listener |
| `containerFactory` | Select a custom listener-container factory |
| `concurrency` | Consumer threads in this application instance |
| `ackMode` | Offset acknowledgment mode |
| `autoStartup` | Whether the listener starts with the application |
| `clientIdPrefix` | Prefix for Kafka client IDs and metrics |
| `properties` | Listener-specific Kafka consumer property overrides |
| `errorHandler` | Listener-level error handler |
| `batch` | Receive a batch/list instead of one record |
| `filter` | Apply a record filter strategy |

`id` is not the same as `groupId`. If no `groupId` is specified, Spring Kafka can use `id` as the group ID by default; explicitly set both to avoid ambiguity.

The `ackMode` annotation attribute in this example requires Spring Kafka 4.1 or later. With earlier releases, configure `ContainerProperties.AckMode.MANUAL_IMMEDIATE` on the listener-container factory and omit the annotation attribute.

## 9. Retry and dead-letter handling

Separate temporary failures from permanent failures.

```text
temporary failure → bounded retry with backoff → still failing → DLT
permanent failure ────────────────────────────────────────────→ DLT
```

Typical permanent failures include invalid schema, missing required data, and unrecoverable business validation errors.

Choose retry behavior with ordering in mind. Blocking retries hold up later records from the same partition while a record is retried. Retry-topic patterns let the main partition continue, but later events for the same key can be processed before the delayed retry. Dead-letter records should retain the original topic, partition, offset, key, and exception context; replay them through an idempotent handler after fixing the cause.

Use a dead-letter topic for investigation and controlled replay. Do not retry a poison message forever.

## 10. Event design

Prefer events that describe facts:

```json
{
  "event_id": "evt-123",
  "event_type": "OrderCreated",
  "event_version": 1,
  "occurred_at": "2026-10-28T10:00:00Z",
  "correlation_id": "request-789",
  "order_id": "order-456",
  "customer_id": "customer-999"
}
```

Include:

- Stable `event_id`
- `event_type`
- Schema/event version
- Event timestamp
- Aggregate/entity ID
- Correlation or trace ID
- Enough data for consumers to act without immediately calling the producer

Use backward-compatible schema evolution. Do not remove or rename fields that existing consumers require.

## 11. Database-to-Kafka patterns

### Transactional outbox

Use when your service writes a database and publishes an event:

```text
one database transaction:
  save business data
  save event to outbox table

outbox publisher:
  read outbox rows
  publish to Kafka
  mark rows published
```

This avoids losing the event intent between committing the business change and publishing to Kafka. The publisher can crash after publishing but before marking the row, so it can publish the event again; use a stable `event_id` and idempotent consumers. Outbox delivery is normally at-least-once, not an atomic database-plus-Kafka transaction.

### CDC in AWS

For RDS/Aurora PostgreSQL:

```text
RDS/Aurora PostgreSQL
        ↓ WAL/logical replication
AWS DMS
        ↓
Amazon MSK topic
        ↓
Spring Boot consumer groups
```

AWS DMS reads changes through logical replication and can publish them to an MSK Kafka topic. For domain events, consider writing an explicit outbox row in the same business transaction and capturing that table, instead of treating every raw table change as a stable event contract. Configure DMS object mapping and a partition key deliberately; target settings and output shape matter. Monitor replication-slot/WAL growth, task health, and CDC lag. See [DMS Kafka target guidance](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Target.Kafka.html) and [PostgreSQL source/CDC guidance](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html).

## 12. `OffsetStrategy.ASYNC` warning

`OffsetStrategy.ASYNC` is associated with Micronaut Kafka, not the normal Spring Kafka `ackMode` API.

In Micronaut, it means offsets are committed asynchronously after a successful poll loop. Duplicates can still occur after failures, so idempotent processing remains necessary.

In Spring Kafka, the comparable concepts are:

```java
factory.getContainerProperties().setAckMode(
        ContainerProperties.AckMode.RECORD
);

factory.getContainerProperties().setSyncCommits(false);
```

Use `ackMode` to control **when** the offset is ready and `syncCommits` to control **how** the commit is sent.

## 13. Monitoring checklist

Monitor at least:

- Consumer lag
- Processing latency
- Failed records and DLT volume
- Rebalance frequency
- Producer error/retry rate
- Broker disk usage
- Partition distribution and hot partitions
- Replication health
- Database/CDC WAL or replication-slot growth

For Amazon MSK, use CloudWatch and/or Prometheus for broker and consumer-lag metrics.

## 14. Design checklist

Before production, answer:

- What is the topic name and event contract?
- What key preserves the required ordering?
- How many partitions are needed for peak throughput and consumer parallelism?
- What is the consumer group ID?
- When is the offset acknowledged?
- How are duplicate events prevented?
- What happens for temporary versus permanent failures?
- Is there a retry topic and dead-letter topic?
- How will schemas evolve?
- Does the database-to-event flow need an outbox or CDC?
- Which metrics and alerts indicate trouble?
- How are Kafka and MSK credentials, TLS, and topic permissions managed?

## References

- [Apache Kafka introduction](https://kafka.apache.org/intro/)
- [Apache Kafka design and delivery semantics](https://kafka.apache.org/43/design/design/)
- [Apache Kafka producer configuration](https://kafka.apache.org/40/configuration/producer-configs/)
- [Spring Boot Kafka support](https://docs.spring.io/spring-boot/reference/messaging/kafka.html)
- [Spring Kafka `@KafkaListener`](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/listener-annotation.html)
- [Spring Kafka offset commits and acknowledgment modes](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/message-listener-container.html)
- [Amazon MSK documentation](https://docs.aws.amazon.com/msk/latest/developerguide/what-is-msk.html)
- [AWS DMS Kafka target](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Target.Kafka.html)
- [AWS DMS PostgreSQL CDC](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html)
