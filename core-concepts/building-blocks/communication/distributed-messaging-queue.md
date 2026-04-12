# 📬 Distributed Messaging Queue

> *"A queue doesn't make your system faster. It makes your system honest about how much work it can actually do."*

**⏱ Reading time:** ~13 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Communication

---

## Table of Contents

1. [What Problem a Messaging Queue Solves](#1-what-problem-a-messaging-queue-solves)
2. [How a Queue Works](#2-how-a-queue-works)
3. [Delivery Guarantees](#3-delivery-guarantees)
4. [Message Ordering](#4-message-ordering)
5. [Queue vs Direct Service Call — When to Use Each](#5-queue-vs-direct-service-call--when-to-use-each)
6. [Dead Letter Queues — Handling Poison Messages](#6-dead-letter-queues--handling-poison-messages)
7. [Backpressure — The Queue as a Buffer](#7-backpressure--the-queue-as-a-buffer)
8. [Queue Internals — How Kafka and SQS Work](#8-queue-internals--how-kafka-and-sqs-work)
9. [Queue Failure Modes](#9-queue-failure-modes)
10. [How Queues Connect to Other Building Blocks](#10-how-queues-connect-to-other-building-blocks)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What Problem a Messaging Queue Solves

When Service A needs to send work to Service B, the direct approach is a synchronous call: A calls B, waits for B to respond, and continues. This works. But it creates four problems at scale:

**Temporal coupling** — A and B must both be alive simultaneously. If B is down, A fails. If B is slow, A is slow.

**Load coupling** — if A sends 10,000 requests/second but B can only process 1,000/second, B is overwhelmed. A has no way to slow down without failing.

**Scaling coupling** — to handle more load, you must scale A and B together, even if only one is the bottleneck.

**Failure coupling** — if B fails mid-processing, A has no record of what was sent and can't retry intelligently.

A messaging queue solves all four by introducing an **intermediary**:

```
Without queue:           With queue:
A ──────────────► B      A ──────────► Queue ──────────► B

A waits for B.           A drops message and moves on.
B must be alive.         B processes when it's ready.
B's speed = A's speed.   B processes at its own pace.
```

A puts a message in the queue and immediately moves on to its next task. B reads from the queue whenever it has capacity. They're decoupled in time, in speed, and in failure mode.

---

## 2. How a Queue Works

The fundamental model is simple:

**Producer** — the service that sends messages. Writes to the queue and moves on.

**Queue** — the intermediary. Stores messages durably until they're processed. Handles delivery guarantees.

**Consumer** — the service that processes messages. Reads from the queue at its own pace. Acknowledges successful processing.

```
Producer                Queue                 Consumer
   │                      │                      │
   │  send(message)        │                      │
   ├─────────────────────► │                      │
   │  ack: message stored  │                      │
   │ ◄──────────────────── │                      │
   │                       │   receive(message)   │
   │                       │ ◄──────────────────── │
   │                       │   message delivered  │
   │                       │ ─────────────────────►│
   │                       │   [consumer processes]│
   │                       │                       │
   │                       │   ack: done           │
   │                       │ ◄─────────────────────│
   │                       │ [message deleted      │
   │                       │  from queue]          │
```

The critical step is the **acknowledgment**. The queue doesn't delete a message when it delivers it — it marks it as "in flight." Only when the consumer explicitly acknowledges completion does the queue permanently delete it. If the consumer crashes before acknowledging, the queue re-delivers the message to another consumer.

This is what makes queues reliable: messages are never lost because a consumer crashed mid-processing.

---

## 3. Delivery Guarantees

Different queues offer different guarantees about how messages are delivered. This is one of the most important design decisions when choosing a queue.

### At-Most-Once
The message is delivered zero or one times. If delivery fails, the message is lost. No retries.

```
Producer sends message
Queue delivers to consumer
Consumer crashes before processing
Message is gone — not retried

Acceptable for: metrics, analytics events, logs where occasional loss is fine
Not acceptable for: payments, orders, any operation where loss causes harm
```

### At-Least-Once
The message is delivered one or more times. If delivery or acknowledgment fails, the message is re-delivered. Duplicates are possible.

```
Producer sends message
Queue delivers to consumer
Consumer processes message
Consumer crashes before sending ack
Queue re-delivers to another consumer
Message processed TWICE

This is the most common guarantee.
Applications must be idempotent to handle duplicates safely.
```

This connects directly to [idempotency from RPC](../../preliminary-system-design-concepts/networkAbstraction.md). At-least-once delivery means your consumer will occasionally process the same message twice. If your processing is idempotent (charging with an idempotency key, using upsert instead of insert), duplicates are harmless.

### Exactly-Once
The message is delivered and processed exactly one time, guaranteed. No loss, no duplicates.

```
Sounds ideal. The reality: true exactly-once delivery is extremely
expensive to achieve in a distributed system.

It requires coordination between producer, queue, and consumer —
distributed transactions that add significant latency and complexity.
```

Kafka supports exactly-once semantics with transactions. Most systems achieve the *effect* of exactly-once by combining at-least-once delivery with idempotent consumers — practically equivalent, without the cost.

**The practical recommendation:** Design for at-least-once delivery with idempotent consumers. You get near-exactly-once behavior at the cost of only occasional duplicate processing, which your idempotency logic handles transparently.

---

## 4. Message Ordering

Does the order messages are processed matter for your use case? This is a critical question with significant architectural implications.

### FIFO (First In, First Out)
Messages are processed in the exact order they were sent.

```
Producer sends: [A, B, C, D]
Consumer receives: [A, B, C, D]  (guaranteed order)
```

**When it matters:** User action sequences (a user creates an account, then updates their profile — these must happen in order). Financial transactions where order affects the final balance.

**The cost:** Strict FIFO requires serialization — you can't process B until A is done. This limits parallelism and throughput.

### Best-Effort Ordering
Messages are generally delivered in order but with no strict guarantee.

**When it's fine:** Most background jobs where order doesn't affect correctness (sending emails, resizing images, generating reports).

### Partition-Based Ordering
A middle ground used by Kafka. Messages are partitioned by a key (e.g., user ID). Within a partition, ordering is guaranteed. Across partitions, it isn't.

```
user_id=123 messages → always go to Partition 2 → processed in order
user_id=456 messages → always go to Partition 5 → processed in order
But Partition 2 and Partition 5 process independently and concurrently
```

This gives you ordered processing per entity (all of user 123's events are in order) while still allowing high parallel throughput across many users. It's the standard approach for event streaming systems.

---

## 5. Queue vs Direct Service Call — When to Use Each

This is the decision you'll make constantly in system design. Neither is universally better — they solve different problems.

```
Use a DIRECT call when:
  ✓ The caller needs the result immediately to continue
  ✓ The operation must complete before returning a response to the user
  ✓ Failure of the downstream service should fail the caller
  ✓ You need the return value for next steps

  Examples:
    - User logs in → auth service validates credentials
    - Payment endpoint → payment processor charges card
    - Search endpoint → search service returns results

Use a QUEUE when:
  ✓ The work can happen after the response is returned to the user
  ✓ The downstream service is slower than the upstream
  ✓ You want to decouple failure modes
  ✓ Multiple consumers need to process the same event
  ✓ You need to smooth out traffic spikes

  Examples:
    - User uploads video → queue transcoding job
    - Order placed → queue email confirmation
    - Photo uploaded → queue resize to multiple resolutions
    - Event occurs → queue analytics recording
```

The key question: **does the user need to wait for this to complete?**

If yes → direct call.
If no → queue it.

---

## 6. Dead Letter Queues — Handling Poison Messages

Some messages can never be processed successfully — malformed data, a bug in the consumer, a dependency that's permanently unavailable. If you keep retrying these forever, they block the queue and waste resources.

A **Dead Letter Queue (DLQ)** is a separate queue where messages go after exceeding a maximum retry count. They're quarantined for later inspection and manual handling.

```
Normal queue:
  Message → Consumer fails → Retry 1 → Retry 2 → Retry 3 → Max retries exceeded
                                                                     │
                                                                     ▼
                                                              Dead Letter Queue
                                                              (for inspection)

Engineers can then:
  - Inspect the failed message to understand the bug
  - Fix the consumer code
  - Replay messages from the DLQ once fixed
  - Discard messages that were invalid input
```

Without a DLQ, a single malformed message could cause infinite retries, blocking the queue indefinitely. With a DLQ, problematic messages are isolated, normal messages continue processing, and the team gets visibility into what went wrong.

**DLQs are not optional** in any production queue system that processes critical work. They're your safety net.

---

## 7. Backpressure — The Queue as a Buffer

One of the most powerful properties of a queue is its ability to absorb traffic spikes — a concept called **backpressure**.

```
Without queue (direct calls):

Traffic spike: 10,000 requests/second
Service capacity: 1,000 requests/second
Result: 9,000 requests/second fail or time out
        Service may crash under load
        Failures cascade to callers

With queue:

Traffic spike: 10,000 messages/second enter queue
Service capacity: 1,000 messages/second consumed
Queue depth grows: 9,000 messages waiting
Result: all messages eventually processed
        Service operates at comfortable capacity
        No failures, just latency
```

The queue acts as a buffer. Instead of requests failing during spikes, they wait. The service processes them at its own pace. The queue absorbs the burst.

**The tradeoff:** messages are processed later, not immediately. If your system needs real-time responses, a queue introduces unacceptable latency. If your system processes work asynchronously, the queue's buffering is a feature.

**Queue depth as an operational metric:** if your queue depth is consistently growing, your consumers can't keep up with producers. You need more consumer instances. Queue depth is one of the most important metrics to monitor.

---

## 8. Queue Internals — How Kafka and SQS Work

Two very different queue designs with different tradeoffs.

### Amazon SQS — Simple Queue Service

SQS is a managed, traditional message queue. Messages are stored once, delivered once (or a few times if the consumer fails), and deleted after successful processing.

```
Key properties:
  - Messages deleted after consumer acks
  - Multiple consumers compete for messages (each message processed once)
  - Visibility timeout: message is hidden from others while one consumer processes it
  - Automatic scaling — fully managed
  - Max message retention: 14 days
  - FIFO queue variant available for ordered processing
```

**Good for:** Task queues where each job should be done exactly once. Background jobs. Worker queues.

### Apache Kafka — Distributed Event Log

Kafka is fundamentally different from a traditional queue. Messages are stored in an **append-only log** and are *not deleted* after consumption. Consumers track their own position (called an **offset**) in the log.

```
Kafka log for "order-events" topic:

Offset: [0]  [1]   [2]   [3]   [4]   [5]   [6]
        order order order order order order order
        placed placed paid  paid  shipped closed  closed

Consumer A (billing service): reading at offset 4
Consumer B (notification service): reading at offset 6
Consumer C (analytics service): reading at offset 2

Each consumer reads independently at its own pace.
Kafka doesn't care — it just keeps the log.
Old messages remain available (configurable retention period).
```

**Key difference from SQS:** In Kafka, multiple consumer groups can all read the same messages independently. Adding a new consumer doesn't affect existing ones — it just starts reading from wherever in the log it needs to.

**Good for:** Event streaming, audit logs, multiple consumers needing the same events, replaying historical events, real-time analytics pipelines.

### The Choice

```
Use SQS when:
  - Each message should be processed by exactly one consumer
  - You want a managed service with minimal ops overhead
  - Task queues, job queues, work queues

Use Kafka when:
  - Multiple independent consumers need the same events
  - You need to replay events (re-process from the beginning)
  - You're building an event-driven architecture
  - High throughput (millions of messages/second)
  - Event sourcing or audit logging
```

---

## 9. Queue Failure Modes

### Queue Unavailability
If the queue is down, producers can't send and consumers can't receive. This is why queue systems are deployed in clusters with replication.

**Design implication:** your system should handle queue unavailability gracefully. Can the producer fall back to a direct call? Can it buffer messages locally and retry? Or should it fail with a clear error?

### Consumer Lag
Producers publish faster than consumers process. Queue depth grows. Messages age. If this isn't caught, the queue can grow unbounded.

**Detection:** Monitor queue depth and consumer lag as operational metrics. Alert when lag exceeds acceptable thresholds.
**Fix:** Add more consumer instances. Scale out the consumer horizontally.

### Message Expiry
Messages that sit in the queue too long may expire before being processed (depending on your retention settings). This can cause silent data loss.

**Design:** Set retention periods that are longer than your worst-case recovery time. Consider what happens to messages that expire — do you need to re-generate them?

### Duplicate Processing at Scale
With at-least-once delivery, duplicates happen — especially during failure recovery. If consumers are not idempotent, this causes data corruption.

**This is the most common production bug with message queues.** Always ask: what happens if this message is processed twice?

---

## 10. How Queues Connect to Other Building Blocks

```
Producer Service ───────────────────────────────────────────────────────►
  Sends messages to queue.
  Decoupled from consumer — doesn't know how many consumers exist.

Queue (Kafka / SQS) ────────────────────────────────────────────────────►
  Stores messages durably.
  Handles delivery guarantees and retry logic.

Consumer Service ────────────────────────────────────────────────────────►
  Reads from queue at its own pace.
  Must be idempotent for at-least-once delivery.
  Scales independently of producer.

Dead Letter Queue ───────────────────────────────────────────────────────►
  Receives messages that exceed max retries.
  Enables visibility into failures without blocking the main queue.

Distributed Monitoring ──────────────────────────────────────────────────►
  Monitor queue depth (growing = consumers can't keep up).
  Monitor consumer lag (how far behind are consumers?).
  Alert on DLQ growth (messages failing to process).

Task Scheduler ──────────────────────────────────────────────────────────►
  Scheduled jobs often publish to a queue for worker processing.
  Queue handles the work distribution; scheduler handles the timing.
```

---

## 11. Self-Check

1. What are the four problems that a messaging queue solves compared to direct service calls?
2. What is the difference between at-most-once, at-least-once, and exactly-once delivery? Which is most common in practice, and why?
3. Why must consumers be idempotent when using at-least-once delivery?
4. What is a Dead Letter Queue, and why is it essential in production systems?
5. What is backpressure, and how does a queue provide it? Give a concrete example where it saves a system from failure.
6. What is the fundamental difference between Kafka and SQS? When would you choose Kafka over SQS?
7. You're designing an e-commerce order system. A user places an order. The system needs to: (a) charge the card, (b) update inventory, (c) send a confirmation email, (d) notify the warehouse. Which of these should be synchronous direct calls, and which should go through a queue? Why?

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 11 (Kleppmann)](https://dataintensive.net) | Stream processing and messaging — the most thorough treatment available |
| 🔧 [Apache Kafka Documentation](https://kafka.apache.org/documentation/) | Core concepts: topics, partitions, consumer groups, offsets |
| 📬 [ByteByteGo — Message Queue Design](https://bytebytego.com) | Visual breakdown of queue internals and patterns |
| 📝 [AWS SQS vs Kafka — When to Use Each](https://aws.amazon.com/compare/the-difference-between-sqs-and-kafka/) | Side-by-side comparison with concrete use cases |

---

*⬅️ Previous: [Communication Overview](Communication.md) &nbsp;|&nbsp; ➡️ Next: [Pub-Sub](pub-sub.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Communication.</sub>