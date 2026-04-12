# 📡 Pub-Sub — Publish-Subscribe System

> *"A queue asks: who will do this work? Pub-sub asks: who needs to know this happened?"*

**⏱ Reading time:** ~11 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Communication

---

## Table of Contents

1. [What Pub-Sub Is](#1-what-pub-sub-is)
2. [How Pub-Sub Works](#2-how-pub-sub-works)
3. [Pub-Sub vs Messaging Queue — The Critical Distinction](#3-pub-sub-vs-messaging-queue--the-critical-distinction)
4. [Topics and Subscriptions](#4-topics-and-subscriptions)
5. [Push vs Pull Delivery](#5-push-vs-pull-delivery)
6. [Fan-Out — The Core Pattern](#6-fan-out--the-core-pattern)
7. [Ordering and Delivery Guarantees](#7-ordering-and-delivery-guarantees)
8. [Pub-Sub Failure Modes](#8-pub-sub-failure-modes)
9. [Real-World: How Uber Uses Pub-Sub](#9-real-world-how-uber-uses-pub-sub)
10. [How Pub-Sub Connects to Other Building Blocks](#10-how-pub-sub-connects-to-other-building-blocks)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What Pub-Sub Is

**Pub-Sub** (Publish-Subscribe) is an asynchronous messaging pattern where **publishers** send events to named **topics**, and **subscribers** receive those events from the topics they care about. Publishers and subscribers don't know about each other — the topic is the intermediary.

The defining characteristic: **one event, many consumers**. When something happens in your system, pub-sub broadcasts that event to every service that cares about it, simultaneously, without the publisher needing to know who those services are.

This is architecturally significant. Without pub-sub, if you want three services to know about an event, the publishing service must explicitly call all three — tightly coupling it to every consumer. With pub-sub, the publisher emits one event; the infrastructure handles delivery to all subscribers. Add a fourth subscriber? Zero changes to the publisher.

---

## 2. How Pub-Sub Works

```
Publisher                  Topic                  Subscribers
    │                        │                   ┌── Subscriber A
    │  publish(event)         │  deliver(event)   │
    ├────────────────────────►│──────────────────►├── Subscriber B
    │                         │                   │
    │                         │──────────────────►└── Subscriber C
```

**Publisher** — produces events and sends them to a topic. Doesn't know or care who's subscribed.

**Topic** — a named channel. The publisher writes to it; subscribers read from it. The topic handles routing and delivery.

**Subscriber** — registers interest in a topic. Receives every event published to that topic. Multiple subscribers can read the same event independently.

A critical property: **each subscriber gets its own copy of every event.** Unlike a queue where one consumer processes a message and it's gone, pub-sub delivers the same event to all subscribers independently.

---

## 3. Pub-Sub vs Messaging Queue — The Critical Distinction

This is the distinction from the [Communication overview](Communication.md) worth making concrete.

```
MESSAGING QUEUE                        PUB-SUB
──────────────                         ───────
One message → one consumer             One event → all subscribers

Message represents "work to do"        Event represents "something happened"

Message deleted after processing       Event delivered to all, independently

Consumers compete for messages         Subscribers are independent

Adding consumer: shares the work       Adding subscriber: gets its own copy

"Process this payment"                 "Payment was completed"
  → one payment processor                → billing service logs it
    handles it                           → inventory updates
                                         → notification service emails
                                         → analytics records it
```

**The mental model:**
- Queue: a todo list that workers pick tasks from
- Pub-Sub: a news broadcast that everyone watching receives

A payment processor queue has workers competing to process jobs — you want each payment processed once. A payment-completed pub-sub topic broadcasts the event to every interested service — you want everyone to know.

---

## 4. Topics and Subscriptions

### Topics
A topic is a named stream of events of a specific type. Good topic design mirrors real-world events in your system:

```
Good topic names (event-based, past tense):
  order.placed
  user.registered
  payment.completed
  video.uploaded
  driver.location.updated

Bad topic names (vague, action-based):
  notifications
  updates
  events
  service_a_to_service_b
```

Topic naming should reflect what happened, not what to do about it. The publisher doesn't decide what subscribers do — that's the subscriber's concern.

### Subscriptions
A subscription is a named association between a subscriber and a topic. Each subscription has its own delivery cursor — the subscriber's position in the event stream.

```
Topic: order.placed
  Subscription: inventory-service       → reads all events, at its own pace
  Subscription: notification-service    → reads all events, at its own pace
  Subscription: analytics-service       → reads all events, at its own pace
  Subscription: fraud-detection-service → reads all events, at its own pace
```

The topic doesn't coordinate between subscriptions. Each subscriber's progress is tracked independently. A slow subscriber doesn't slow down fast ones.

### Consumer Groups (Kafka)
In Kafka, a consumer group is a set of consumer instances that collectively process a subscription. Messages within each partition go to one instance in the group — enabling parallel processing without duplicate processing within the group.

```
Topic: order.placed (4 partitions)
Consumer Group: inventory-service (3 instances)

Partition 0 → Instance 1
Partition 1 → Instance 2
Partition 2 → Instance 3
Partition 3 → Instance 1 (wraps around)

All instances together process all partitions.
Each partition goes to exactly one instance in the group.
Scale the group by adding instances (up to partition count).
```

---

## 5. Push vs Pull Delivery

Two models for how subscribers receive events.

### Pull (Consumer Polls)
Subscribers actively poll the topic for new events. The subscriber controls the rate.

```
Subscriber asks: "Any new events on order.placed since offset 4?"
Topic responds: "Yes, here are events 5, 6, 7, 8"
Subscriber processes them
Subscriber asks again...
```

**Pros:** Subscriber controls rate — processes at its own pace, no risk of being overwhelmed. Natural backpressure.
**Cons:** Latency — subscriber may wait before polling again. Small delay between event published and subscriber receiving it.

**Kafka uses pull.**

### Push (Topic Pushes to Subscriber)
The topic proactively delivers events to subscribers as they arrive.

```
Event published → Topic immediately pushes to all subscribers
Subscribers receive events as they happen (low latency)
```

**Pros:** Low latency — events delivered immediately.
**Cons:** Subscriber can be overwhelmed if events arrive faster than it can process. Requires subscriber to handle backpressure somehow.

**Google Cloud Pub/Sub and AWS SNS support push delivery.**

**In practice:** Pull is more common for high-throughput systems because it gives the subscriber control. Push is useful for low-latency scenarios where subscribers can keep up with the event rate.

---

## 6. Fan-Out — The Core Pattern

**Fan-out** is the defining pattern of pub-sub: one event triggers actions in many services simultaneously.

```
User places order
        │
        ▼
Order Service publishes: order.placed
  {
    order_id: 98765,
    user_id: 123,
    items: [...],
    total: 149.99,
    timestamp: "2024-01-15T10:30:00Z"
  }
        │
        ├──────────────────────────────────────────────────────────►
        │                         │                                 │
        ▼                         ▼                                 ▼
Inventory Service         Notification Service              Analytics Service
  Reserves items            Sends confirmation email           Logs the event
  Updates stock             Sends SMS if opted in              Updates dashboards
  Checks reorder levels     Schedules warehouse alert          Feeds ML model
```

Three things happen in parallel. Order Service doesn't call any of them directly — it just publishes the event. Each subscriber acts independently.

**What this architecture gains:**

**Decoupling** — Order Service has zero knowledge of Inventory, Notification, or Analytics. Adding a new service (Fraud Detection, Loyalty Points) requires zero changes to Order Service — just add a new subscription.

**Independent scaling** — each subscriber scales based on its own processing needs, not the publisher's volume.

**Independent failure** — if Notification Service crashes, orders still get placed and inventory still updates. The failed subscriber can catch up when it recovers.

**Independent deployment** — each subscriber deploys independently. Rolling out a change to Notification Service doesn't affect Order Service or Inventory Service.

---

## 7. Ordering and Delivery Guarantees

Pub-sub systems generally offer the same delivery guarantees as queues — the same tradeoffs apply.

**At-least-once** is the most common. Events may be delivered more than once; subscribers must be idempotent.

**Ordering** is topic/partition specific. Within a Kafka partition, order is guaranteed. Across partitions, it isn't. For pub-sub use cases, strict global ordering is rarely needed — each subscriber processes events for specific entities (a user's events, an order's events) where per-entity ordering is maintained through partition key assignment.

```
Partition key = user_id
All events for user 123 → Partition 3 → processed in order
All events for user 456 → Partition 7 → processed in order
User 123 and user 456 events may interleave across their partitions → fine
```

---

## 8. Pub-Sub Failure Modes

### Slow Subscriber (Consumer Lag)
A subscriber processes events slower than they're published. The subscription's lag grows. In Kafka, events are retained for a configurable period — if the subscriber catches up within that window, no events are lost. If it falls too far behind and retention expires, events are gone.

**Monitor:** subscriber lag per subscription. Alert when lag grows beyond acceptable thresholds.

### Missing Subscriber
An event is published, but the relevant subscriber doesn't exist yet (not deployed, subscription not created). In most pub-sub systems, events published before a subscription exists are not delivered to that subscription.

**Design implication:** create subscriptions before starting the subscriber service. For Kafka, consumer groups start reading from a configured position (beginning, end, or specific offset).

### Thundering Herd on Startup
When a subscriber service restarts after being down, it may have a large backlog of events to process. Processing the entire backlog as fast as possible can overwhelm downstream dependencies.

**Solution:** implement rate limiting in the consumer. Process events at a controlled rate even when catching up.

### Event Schema Evolution
The publisher changes the event structure (renames a field, removes a field). Subscribers break if they depend on the removed/renamed field.

**Solution:** treat event schemas as APIs — backwards compatible changes only (add fields, don't remove). Version event schemas when breaking changes are unavoidable. Use a schema registry (Confluent Schema Registry for Kafka) to enforce compatibility.

---

## 9. Real-World: How Uber Uses Pub-Sub

Uber's core infrastructure heavily uses pub-sub for the events that flow through a ride.

When a rider requests a ride:
```
RideRequest Service publishes: ride.requested
  {
    ride_id: "uuid",
    rider_id: "...",
    pickup: {lat, lng},
    destination: {lat, lng},
    timestamp: "..."
  }

Subscribers:
  Dispatch Service      → finds nearby drivers, sends them the request
  Pricing Service       → calculates surge pricing for the area
  ETA Service           → computes estimated arrival time
  Analytics Service     → records request for demand modeling
  Fraud Service         → checks for suspicious patterns
```

When a driver accepts:
```
DriverAcceptance Service publishes: ride.accepted
  { ride_id, driver_id, eta }

Subscribers:
  Rider Notification    → "Your driver is on the way, ETA 4 min"
  Driver Navigation     → provides routing directions
  Tracking Service      → starts real-time location sharing
  Billing Service       → pre-authorizes payment
```

The key insight: at every meaningful state transition in a ride, an event is published. Each domain service subscribes only to the events relevant to it. No service needs to know about all the others. The system is as loosely coupled as a distributed system can be.

---

## 10. How Pub-Sub Connects to Other Building Blocks

```
Any Service (Publisher) ────────────────────────────────────────────────►
  Publishes events when meaningful state changes occur.
  Knows nothing about subscribers.

Pub-Sub Topic ───────────────────────────────────────────────────────────►
  Stores events durably (Kafka: append-only log with configurable retention)
  Routes to all subscriptions independently.

Subscriber Services ─────────────────────────────────────────────────────►
  Each subscribes to relevant topics.
  Processes events idempotently.
  Scales independently.

Messaging Queue (per subscriber) ────────────────────────────────────────►
  Common pattern: pub-sub topic → per-subscriber queue → worker pool
  Pub-sub handles fan-out; queue handles reliable per-subscriber delivery.

Dead Letter Queue ───────────────────────────────────────────────────────►
  Per subscription: events that fail to process go to a DLQ.
  Each subscriber has its own DLQ — failures are isolated.

Distributed Monitoring ──────────────────────────────────────────────────►
  Monitor subscriber lag per subscription.
  Alert on DLQ growth.
  Track event publish rate vs consume rate.
```

---

## 11. Self-Check

1. What is the core difference between pub-sub and a messaging queue? When does the distinction actually matter?
2. What does "fan-out" mean, and what architectural properties does it enable?
3. Why should event topics be named in the past tense (e.g., `order.placed` not `process-order`)?
4. What is the difference between push and pull delivery? Which gives the subscriber more control, and why?
5. A new "Loyalty Points" service needs to award points whenever an order is placed. The Order Service already publishes an `order.placed` event. What changes need to be made to the Order Service to enable this?
6. A subscriber falls behind and its consumer lag grows to 2 million events. The Kafka topic has a retention of 7 days. The subscriber has been down for 3 days. What happens when it recovers?
7. You're designing a food delivery app. A driver completes a delivery. List at least four services that need to know about this event and what each one does with it.

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 11 (Kleppmann)](https://dataintensive.net) | The best treatment of event streams, pub-sub, and their place in system architecture |
| 🔧 [Apache Kafka Documentation — Consumer Groups](https://kafka.apache.org/documentation/#intro_consumers) | How consumer groups and offsets work in practice |
| 📬 [ByteByteGo — Pub-Sub Pattern](https://bytebytego.com) | Visual walkthrough of pub-sub architecture and fan-out |
| 📝 [Google Cloud Pub/Sub Overview](https://cloud.google.com/pubsub/docs/overview) | Real-world managed pub-sub with concrete delivery guarantees |

---

*⬅️ Previous: [Distributed Messaging Queue](distributed-messaging-queue.md) &nbsp;|&nbsp; ➡️ Next Group: [Processing](../processing/Processing.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Communication.</sub>