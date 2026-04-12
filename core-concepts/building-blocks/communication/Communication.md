# 📨 Communication

> *"Synchronous calls couple services together. Asynchronous communication sets them free."*

---

## The Problem This Group Solves

When Service A needs to tell Service B something happened, the naive approach is a direct synchronous call — A calls B, waits for a response, and continues. This works fine at small scale. At large scale, it creates a web of problems:

- If B is slow, A is slow
- If B is down, A fails
- If a million things happen simultaneously, B is overwhelmed
- If you want C and D to also know what happened, A has to call all three

Asynchronous communication breaks these couplings. Instead of calling B directly, A publishes a message to an intermediary. B (and C and D) pick up that message when they're ready. A doesn't wait. A doesn't know or care whether B is healthy. A doesn't know how many consumers exist.

This group covers the two asynchronous communication building blocks — and understanding the difference between them is one of the most important distinctions in distributed systems.

---

## The Components

| Building Block | Solves | Guide |
|---------------|--------|-------|
| **Distributed Messaging Queue** | Delivering work from one producer to one consumer reliably | [Read →](distributed-messaging-queue.md) |
| **Pub-Sub** | Broadcasting events from one producer to many consumers simultaneously | [Read →](pub-sub.md) |

---

## The Critical Distinction: Queue vs Pub-Sub

This is the distinction most people get fuzzy on. Both are asynchronous. Both involve messages. But they solve fundamentally different problems.

```
MESSAGING QUEUE                         PUB-SUB
───────────────                         ───────
One producer → One consumer             One producer → Many consumers

Message is "work to be done"            Message is "event that happened"

Message is consumed and gone            Message is broadcast to all subscribers

Consumer pulls when ready               All subscribers receive independently

Example: "Process this payment"         Example: "Order was placed"
         → One payment processor                 → Inventory service (update stock)
            handles it                           → Notification service (email user)
                                                 → Analytics service (log the event)
                                                 → Fraud service (check patterns)
```

**The key test:** does the message need to be processed by *one specific thing* (queue) or *everything that cares* (pub-sub)?

A payment processing job goes to a queue — you want exactly one processor to handle it, not four simultaneously each charging the customer.

An order-placed event goes to pub-sub — inventory, notifications, analytics, and fraud detection all need to know independently.

---

## How They Relate to Each Other

Pub-sub and messaging queues aren't alternatives — they're often used together. A common pattern:

```
Order Service
     │
     │  publishes "order_placed" event
     ▼
  Pub-Sub Topic
  /      |      \
 /       |       \
▼        ▼        ▼
Queue A  Queue B  Queue C
  │        │        │
  ▼        ▼        ▼
Inventory  Email    Analytics
Service   Service   Service

Each subscriber gets the event via its own queue.
Each queue ensures the message is processed at least once.
Failures in one subscriber don't affect others.
```

Pub-sub handles the fan-out. Each subscriber's queue handles the reliable delivery and retry logic for that specific consumer.

---

## When You Reach for This Group

A slow operation (video transcoding, email sending, report generation) is blocking user responses → **Queue** it, process async.

Multiple services need to react to the same event → **Pub-Sub** to fan out without coupling.

A service is getting overwhelmed with requests → **Queue** acts as a buffer, smoothing the load.

You need to decouple services so they can scale and fail independently → **Either**, depending on the consumer pattern.

Traffic spikes need to be absorbed without dropping work → **Queue** to buffer the excess.

---

*⬅️⬅️ Previous Group: [Speed](../speed/Speed.md) &nbsp;| &nbsp; ➡️ Deep Dive this group starting with: [Distributed Cache](distributed-cache.md)  | &nbsp; ➡️➡️ Next Group: [Processing](../processing/Processing.md)*