---
date: 2026-08-07 13:24:14
layout: post
title: "Inside a Kafka Order Workflow: Outbox, Retries, DLQs and Idempotency"
subtitle: A hands-on workshop on Transactional Outbox, retry topics, Dead Letter
  Queues, idempotent consumers, and reliable event processing with Spring Boot
  and Kafka.
description: Build an event-driven order workflow using Java, Spring Boot, and
  Kafka. Learn Outbox Pattern, retry topics, DLQs, idempotency, consumer groups,
  and Exactly Once Semantics through practical examples.
image: /assets/img/uploads/chatgpt-image-aug-7-2026-03_42_47-pm.png
optimized_image: /assets/img/uploads/chatgpt-image-aug-7-2026-03_42_47-pm.png
author: malkomich
permalink: /2026-08-07-inside-a-kafka-order-workflow-outbox-retries-dlqs-and-idempotency/
category: software-arquitecture
tags:
  - kafka
  - spring-boot
  - event-driven
  - microservices
  - reliability
  - idempotency
  - dlq
  - architexture
paginate: false
---
## 1. The Naive Workflow: First Failure Strikes

![A sequence diagram showing the dual-write problem: (1) Database transaction commits successfully, (2) Network fails before Kafka send, (3) Order exists in DB but event never reaches Kafka. Show both success and failure paths with timeline indicators.](https://images.ctfassets.net/8vofjvai1hpv/5947ZdOee28dmg5zNmyRcr/28c3d22701a39ad892ee0b22d7aa5584/4.png)

When you're building your first event-driven service with Kafka, the initial implementation feels natural and clean. You have a REST endpoint that saves an order to the database and publishes an event to Kafka. The code is straightforward, and in development, everything works perfectly.

```java
@PostMapping("/orders")
@Transactional
public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest req) {
    Order order = repository.save(new Order(req)); // persists in DB
    OrderEvent event = buildOrderCreatedEvent(order);
    kafkaTemplate.send("order.created", order.getId(), event);
    return ResponseEntity.ok(order);
}
```

The problem appears because the PostgreSQL transaction and the Kafka publication are two separate operations. Kafka may acknowledge the event while the database transaction later rolls back. The opposite failure is also possible, the database transaction may commit while Kafka publication ultimately fails. There is no atomic boundary covering both systems.

The result can be an order that exists in PostgreSQL but never reaches payment or inventory, or an event in Kafka that refers to an order that was never committed.

The fundamental issue is that you're performing two writes, one to PostgreSQL and one to Kafka, and treating them as if they were atomic when they are not. A regular database transaction cannot make both systems commit together. This is the dual-write problem, and it is the first reliability problem we need to solve.

## 2. Transactional Outbox: Escaping Dual-Write Hell

![A before/after architecture diagram comparing naive approach vs outbox pattern. Left side shows direct dual writes to DB and Kafka (with X marks for failure points). Right side shows atomic single transaction to DB+outbox table, then separate async publisher polling outbox and sending to Kafka.](https://chairnerd.seatgeek.com/images/transactional-outbox-pattern/outbox-flow.svg)

The transactional outbox pattern emerged from a simple insight: if you can't make two separate systems transactional, make one system the source of truth and derive the other from it. Instead of writing to both the database and Kafka in the same code path, you write both the business data and the *intent to publish* into the same database transaction. A separate process reads that intent and publishes to Kafka.

Your schema needs to capture this intent, so you add an outbox table alongside your business tables:

```sql
CREATE TABLE order_outbox (
    event_id UUID PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(64) NOT NULL,
    payload JSONB NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    published_at TIMESTAMPTZ,
    publication_status VARCHAR(32) NOT NULL
);
```

Now when an order is created, both records are written in a single transaction. If the transaction commits, both the order and the outbox entry are durable. If it rolls back, neither exists. You've achieved atomicity.

```java
@Transactional
public void saveOrderAndOutbox(Order order) {
    orderRepository.save(order);
    OutboxEvent outboxEvent = new OutboxEvent(order);
    outboxRepository.save(outboxEvent);
}
```

A background publisher polls the outbox table and pushes events to Kafka. This decouples the critical write path from Kafka's availability. If Kafka is down, orders can still be stored and their publication intent remains in the outbox until Kafka recovers.

```java
@Scheduled(fixedDelay = 500)
public void publishOutboxEvents() {
    List<OutboxEvent> events = outboxRepository.findUnpublished();
    for (OutboxEvent event : events) {
        try {
            kafkaTemplate
                .send("order.created", event.getAggregateId(), event.getPayload())
                .get();
            event.markPublished();
            outboxRepository.save(event);
        } catch (Exception e) {
            // Remain unpublished, retry on next poll
        }
    }
}
```

The outbox pattern solves the dual-write problem, but it introduces a new reality: your system now guarantees *at-least-once* delivery, not *exactly-once*. If the publisher successfully sends to Kafka but crashes before marking the event as published, it will send the same event again on the next poll. That is a deliberate trade-off. If an order exists, the intent to publish will eventually succeed. The cost is that downstream consumers must handle duplicates gracefully.

Some teams use Change Data Capture (CDC) tools such as Debezium to stream outbox changes directly to Kafka, avoiding the polling overhead. Polling approach works fine for most workloads and keeps the architecture simpler. Add CDC when you have clear evidence that polling is a bottleneck.

[Read about the Outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html)

## 3. Idempotent Consumers: Payment's Reality Check

Since events can arrive more than once, the next challenge is making sure your business logic doesn't execute more than once. Kafka's delivery guarantees are about messages, not side effects. If the payment service receives the same `OrderCreated` event twice, it shouldn't charge the customer twice.

We can track which events have already been processed. Before doing any business logic, check if you've seen this specific event before. If you have, skip it. If you haven't, record it and proceed.

```java
@Transactional
@KafkaListener(topics = "order.created")
public void consumeOrderCreated(OrderEvent event) {
    if (eventRepository.existsByEventId(event.getEventId())) {
        return; // Already processed
    }
    eventRepository.save(new ProcessedEvent(event.getEventId()));
    paymentRepository.authorizePayment(event.getOrderId());
}
```

The key is that checking and recording happen in the same transaction as the business logic. Either all three happen (check, record, authorize) or none of them do. This table structure is deliberately simple:

```sql
CREATE TABLE processed_events (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMPTZ DEFAULT now()
);
```

The event ID must be stable across publication attempts. Generate it once when the domain event is created. If the outbox publisher sends that event three times due to crashes, all three messages must carry the same event ID.

It's worth distinguishing between Kafka's producer idempotence and consumer idempotence. When you enable idempotent producers in Kafka, you're preventing duplicate messages caused by producer retries and network issues at the protocol level. That's valuable, but it doesn't prevent your business logic from executing twice if the same conceptual event arrives from two different sources or publishing attempts. Consumer idempotence is about protecting your business invariants, not just message delivery semantics.

## 4. Retry Topics: Handling Temporary Glitches Without Partition Stalls

![A message flow diagram showing: original topic → consumer processes message → failure occurs → message routes to retry topic with backoff delay → retry consumer processes it → success OR exhausts retries → routes to DLT. Include timing annotations showing delay between retry attempts.](https://miro.medium.com/v2/resize:fit:1400/1*46dJiQgbH_FS8c6Ed4EyGA.png)

Real distributed systems are filled with transient failures. The payment provider times out but succeeds on retry. The inventory database is under load and rejects connections for thirty seconds. An external API returns 503 but recovers moments later. Traditional retry approaches handle these by blocking: catch the exception, sleep, try again. The problem is that while you're sleeping, your Kafka consumer is stuck. No other messages in that partition get processed until the retry succeeds or gives up.

Spring Kafka's retry topics solve this by moving failed messages to separate topics with delayed consumption. When a message fails, instead of blocking the consumer, you route it to a retry topic and immediately move on to the next message in the original partition.

```java
@RetryableTopic(
    attempts = 3,
    backoff = @Backoff(2000),
    dltTopicSuffix = ".dlt"
)
@KafkaListener(topics = "order.created")
public void handleOrderCreated(OrderEvent event) {
    paymentProvider.authorize(event);
}
```

Behind the scenes, this creates topics like `order.created-retry-1`. Failed messages get republished to these topics with headers indicating retry count and timing. Separate consumers with appropriate delays process the retry topics. After exhausting all retries or facing non-retryable errors like malformed JSON, the message lands in the Dead Letter Topic.

![](/assets/img/uploads/untitled-diagram-2026-08-07-170933.png "Dead Letter Queue (DLQ) Topic flow")

What matters is that the original partition can keep processing later records while the failed record waits in a retry topic.

There is an important trade-off here. Once a failed record leaves the original topic, strict ordering relative to later records is no longer guaranteed. Event A can be waiting in a retry topic while event B, with the same business key, continues through the main topic. If the domain requires strict per-key ordering across failures, nonblocking retries may not be the right choice.

Retries should also be bounded. If a message has failed ten times over several hours, it's not going to magically succeed on attempt eleven. Something is fundamentally broken: bad data, a changed API contract, or a systemic issue. Continuing to retry just creates noise and burns resources. Set reasonable limits and route persistent failures to the DLT for human investigation.

[Spring Kafka Retry Topics Reference](https://docs.spring.io/spring-kafka/reference/retrytopic/how-the-pattern-works.html)

## 5. Dead Letter Topic (DLT): The Final Rest Stop

Some messages simply can't be processed. The event may violate a business rule, contain invalid data, or keep failing after the configured retry attempts. For these cases, you need a place where failed messages go to die, or more accurately, go to wait for manual intervention.

The Dead Letter Topic is that place. When Spring Kafka exhausts all retries, or when you explicitly classify an error as non-retryable, the message gets routed to a DLT, typically with a suffix like `.dlt`.

```java
@KafkaListener(topics = "order.created.dlt")
public void handleDlt(OrderEvent event, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
    log.error("DLT - manual intervention required: {} from {}", event, topic);
    // Optionally trigger notification, alerting, triage, or manual fix
}
```

The DLT handler's job isn't to fix the problem automatically. Instead, it's about making the failure visible and manageable.

A useful operational process is simple: inspect the message, classify the failure, fix the root cause, decide whether replay is safe, then replay through a controlled tool or endpoint. The original topic, partition, offset, exception details, event ID and correlation ID are valuable context during that investigation.

Never set up automatic replay from the DLT without understanding why messages landed there. A permanently invalid message that moves continuously between the main topic and the DLT only creates noise and consumes resources.

Using a Kafka UI tool like Conduktor or Kafka UI makes DLT management dramatically easier. You can inspect message payloads, see headers with retry history and error details, and selectively replay messages after fixes are deployed.

[DLT and retry strategies](https://docs.spring.io/spring-kafka/reference/retrytopic/dlt-strategies.html)

## 6. Consumer Groups, Partitions, Ordering: Concurrency Without Chaos

![A horizontal architecture diagram showing: Topic with 4 partitions, 3 consumer instances in a consumer group with partition assignment distribution (e.g., Instance 1 → Partition 0,1; Instance 2 → Partition 2; Instance 3 → Partition 3). Include message flow showing same order ID always routes to same partition.](https://blog.nashtechglobal.com/wp-content/uploads/2025/10/image-72.png)

Kafka achieves horizontal scalability through partitioning. A topic splits into multiple partitions, and consumers inside the same consumer group divide those partitions between them. When you run multiple instances of a service, Kafka assigns partitions to different instances, allowing parallel processing. Your maximum concurrency is bounded by the number of partitions. If a topic has four partitions, running eight consumer instances doesn't increase throughput.

Partitioning creates a subtle but critical ordering constraint: Kafka guarantees message order within a partition, but not across the whole topic. If messages for the same order land in different partitions, they might be processed out of sequence. The solution is to use the order ID as the message key when producing events.

```java
kafkaTemplate.send("order.created", orderId, event)
```

With a stable partition count and partitioning strategy, events with the same key are routed to the same partition. All events for order 12345 will be processed sequentially on one partition, while events for order 67890 process in parallel on a different partition.

Kafka UI can show which consumer owns each partition, the current offsets and consumer lag. During scaling or deployment, group rebalances redistribute partition ownership between active consumers.

The partition count itself is an architectural decision. Too few partitions and you can't scale beyond a handful of concurrent consumers. Too many and you face operational overhead: more files on disk, more metadata to manage, longer rebalance times.

One trap is assuming global ordering. If you partition by order ID but expect all events across all orders to maintain creation timestamp order, you'll be disappointed. Partition ordering is strict and reliable; cross-partition ordering requires additional application-level logic like vector clocks or sequence numbers.

## 7. Exactly Once Semantics: Understanding the Boundaries

![A layered diagram showing three concentric boundaries: innermost = Kafka transactions scope (producer/offsets only), middle = Producer idempotence scope (delivery-level duplicates), outermost = Consumer idempotency scope (business logic execution). Label what each layer protects and what it doesn't (external systems like databases, APIs).](https://activewizards.com/images/blog/kafka-eos3.svg)

The term "exactly once" in Kafka generates more confusion than almost any other concept. Developers hear it and assume it means their business logic executes exactly once per event, no matter what failures occur. That's not what Kafka's exactly-once semantics guarantee, and understanding the actual scope is crucial for building reliable systems.

Kafka offers two related features: idempotent producers and transactions. Idempotent producers prevent duplicate messages caused by producer retries. When you enable `enable.idempotence=true` and set `acks=all`, Kafka assigns sequence numbers to messages and deduplicates at the broker level. This prevents the scenario where a network timeout causes the producer to retry, resulting in the same message appearing twice in the topic.

Kafka transactions go further, allowing you to atomically commit both consumed offsets and produced messages. This is powerful for consume-transform-produce workflows common in stream processing.

```java
@Bean
public KafkaTemplate<String, InventoryEvent> kafkaTemplate(ProducerFactory pf) {
    pf.setTransactionIdPrefix("inv-tx-");
    return new KafkaTemplate<>(pf);
}

@KafkaListener(topics = "payment.outcome")
public void processPayment(PaymentEvent payment, Acknowledgment ack) {
    kafkaTemplate.executeInTransaction(t -> {
        InventoryEvent inv = new InventoryEvent(payment);
        t.send("inventory.reserve", payment.getOrderId(), inv);
        ack.acknowledge();
        return true;
    });
}
```

Within this transaction, either both the inventory event is produced and the payment event's offset is committed, or neither happens. If the process crashes mid-transaction, recovery will reprocess the payment event, but because the offset wasn't committed, no duplicate inventory event exists in Kafka.

Here's the crucial boundary: this atomicity applies only to Kafka records and offsets. A database update or a provider call is not automatically part of that Kafka transaction. If your processing touches those systems, Kafka's transaction alone cannot make the entire business operation exactly once.

This is why the patterns in this article are complementary:

1. **Idempotent producer**, reduces duplicate Kafka records caused by producer retries.
2. **Kafka transactions**, atomically coordinate Kafka records and consumed offsets.
3. **Idempotent business consumer**, prevents repeated domain side effects.
4. **Transactional Outbox**, coordinates a database change with the durable intent to publish an event.

Kafka transactions make sense for stream processing topologies where you're continuously transforming Kafka events into other Kafka events. For services that interact with databases or external APIs, the outbox pattern combined with consumer idempotency is often simpler and more appropriate.

[Kafka Delivery Semantics](https://docs.confluent.io/kafka/design/delivery-semantics.html)

## 8. Putting It Together: Complete Scenario Walkthrough

Run the workflow and force the failures these patterns are meant to handle:

1. Create an order and verify the order and outbox rows.
2. Observe `OrderCreated` in Kafka UI.
3. Force a temporary payment failure and follow the retry topic.
4. Publish the same event ID twice and verify the local payment transition happens once.
5. Send an invalid event and inspect it in `order.created.dlt`.
6. Complete the flow and confirm the final notification.

```bash
docker compose up -d
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"productId":"SKU-42","quantity":1}'
```

![](/assets/img/uploads/order-service-payment-2026-08-07-175420.png "Workflow diagram")

> **Diagram context:** This view summarizes the order workflow and its main failure paths, including the Outbox Pattern, retries, DLT routing, and duplicate delivery. Kafka transaction boundaries and partition details are omitted to keep the flow readable.

Each failure now has an explicit path: outbox, idempotency guard, retry topic or DLT.

## Workflow Reliability Checklist

* Atomic business data and outbox intent
* Stable event IDs
* Idempotent consumers
* Bounded retries
* Observable DLT replay
* Correct partition keys
* Explicit Exactly Once boundaries

## Further Reading

* [Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
* [Spring Kafka Retry Topics](https://docs.spring.io/spring-kafka/reference/retrytopic/how-the-pattern-works.html)
* [Spring Kafka DLT Strategies](https://docs.spring.io/spring-kafka/reference/retrytopic/dlt-strategies.html)
* [Kafka Consumer Design](https://docs.confluent.io/kafka/design/consumer-design.html)
* [Kafka Delivery Semantics](https://docs.confluent.io/kafka/design/delivery-semantics.html)