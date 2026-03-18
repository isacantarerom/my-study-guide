# 🌐 Network Abstraction: Remote Procedure Calls

> *"What if calling a function on another machine felt exactly like calling one on your own?"*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #2:** Preliminary Concepts

---

## Table of Contents

1. [Why Network Abstraction Exists](#1-why-network-abstraction-exists)
2. [What Is an RPC?](#2-what-is-an-rpc)
3. [How an RPC Call Actually Works](#3-how-an-rpc-call-actually-works)
4. [The Illusion and Its Limits](#4-the-illusion-and-its-limits)
5. [RPC vs REST vs Message Queues](#5-rpc-vs-rest-vs-message-queues)
6. [gRPC: The Modern Standard](#6-grpc-the-modern-standard)
7. [Failure Modes You Must Understand](#7-failure-modes-you-must-understand)
8. [Idempotency: The Key to Safe Retries](#8-idempotency-the-key-to-safe-retries)
9. [Worked Example: Ride-Sharing Dispatch](#9-worked-example-ride-sharing-dispatch)
10. [Self-Check](#10-self-check)
11. [References](#11-references)

---

## 1. Why Network Abstraction Exists

In [Abstraction](abstraction.md), we established that abstraction hides complexity so the layer above can focus on its own problem. Network abstraction applies that idea to one of the hardest problems in distributed systems: **making two machines talk to each other without the developer having to think about what that actually involves.**

What does it actually involve? Every time Service A needs to call Service B over a network, someone has to handle:

- Opening and managing a TCP connection
- Serializing the function arguments into bytes
- Sending those bytes over the wire
- Waiting for a response (or handling the case where one never comes)
- Deserializing the response back into usable data
- Handling partial failures — did the other side receive the request? Did it process it? Did only the response get lost?

In a system with 5 microservices, maybe you write this boilerplate manually. In a system with 50, it becomes thousands of lines of repetitive, error-prone infrastructure code in every service, maintained by every team, diverging over time. That's unsustainable.

**Network abstraction — and RPC specifically — exists to absorb all of that so developers can write a function call and trust the framework to handle the rest.**

---

## 2. What Is an RPC?

**RPC stands for Remote Procedure Call.**

It's a communication model that lets a program call a function on a *different machine* using the same syntax as a local function call. The network communication is entirely hidden.

```
// This looks like a local call...
UserProfile profile = userService.getProfile(userId);

// ...but userService lives on a completely different server,
// possibly in a different data center, on a different continent.
```

The calling side is the **client**. The side being called is the **server**. The piece of code that intercepts the function call, serializes it, sends it, and returns the result is called a **stub** (on the client side) and a **skeleton** (on the server side). These are generated automatically by the RPC framework — you write a service definition, the framework generates the plumbing.

> 🏦 **Analogy:** Think of an ATM. You press "Withdraw $100" and money comes out. You have no idea whether that request was processed locally or routed to a data center across the country. The ATM interface abstracts the entire banking network. The RPC stub is the ATM — it takes your input, handles the complexity underneath, and hands you back a result.

---

## 3. How an RPC Call Actually Works

It's worth understanding the actual lifecycle, because this is where failure modes come from. Every RPC call goes through these steps:

```
CLIENT SIDE                              SERVER SIDE
──────────────────────────────────────────────────────────────
1. Developer calls:
   userService.getProfile(userId)
        │
        ▼
2. Client Stub intercepts the call
   Serializes arguments into bytes
   (JSON, Protobuf, Thrift, etc.)
        │
        ▼
3. Network Transport
   Opens TCP connection (or reuses one)  ───────────────────►
   Sends bytes over the wire                                 │
                                                             ▼
                                           4. Server Stub receives bytes
                                              Deserializes into arguments
                                                             │
                                                             ▼
                                           5. Server executes the function
                                              getProfile(userId) runs locally
                                                             │
                                                             ▼
                                           6. Server Stub serializes the result
   ◄───────────────────                       Sends response bytes back
        │
        ▼
7. Client Stub deserializes the response
   Returns result as if it were a
   normal return value
        │
        ▼
8. Developer receives:
   UserProfile{ name: "Isa", ... }
```

Steps 2–7 are entirely invisible to the developer writing the code. That invisibility is the abstraction working as intended. The problem — as we'll see — is that the things happening in those invisible steps can fail in ways that local function calls never do.

---

## 4. The Illusion and Its Limits

RPC creates a useful illusion: remote calls feel like local calls. But this illusion has real limits, and understanding those limits is what separates engineers who build reliable distributed systems from those who don't.

Peter Deutsch documented the classic version of this in 1994 as the **Fallacies of Distributed Computing** — assumptions developers tend to make about networks that simply aren't true:

| Fallacy | Reality |
|---------|---------|
| The network is reliable | Packets drop. Connections reset. Routers fail. |
| Latency is zero | A local call takes nanoseconds. A network call takes milliseconds — up to 1,000,000× slower. |
| Bandwidth is infinite | Serializing large objects has real cost. Large payloads slow everything down. |
| The network is secure | Data in transit can be intercepted. TLS isn't optional in production. |
| Topology doesn't change | Servers go down. IPs change. DNS propagates slowly. |
| There is one administrator | Every team owns their service. No one owns the whole network. |

The core danger of RPC is precisely that it makes remote calls *look* like local calls, which tempts developers into treating them the same. Local calls never fail mid-execution. They don't have timeouts. They can't partially succeed. Remote calls absolutely can — and do — and they fail in ambiguous ways that are genuinely hard to reason about.

Understanding this isn't pessimism. It's the foundation for designing systems that handle failures gracefully instead of being blindsided by them.

---

## 5. RPC vs REST vs Message Queues

These are the three dominant patterns for service-to-service communication. They're not interchangeable — each exists because it solves a different problem well.

```
         RPC / gRPC              REST / HTTP             Message Queue
         ──────────              ──────────────          ─────────────
         Synchronous             Synchronous             Asynchronous
         Tight coupling          Loose coupling          Very loose coupling
         Fast, typed             Flexible, universal     Decoupled, resilient
         Internal services       Public APIs             Event-driven workflows
         "Call this function"    "Act on this resource"  "This event happened"
```

**RPC / gRPC** is the right tool when two internal services need to communicate frequently, performance matters, and you control both ends of the connection. The tight coupling is acceptable because you own both sides. The speed benefit is real.

**REST** is the right tool when you're building a public-facing API, need broad compatibility across any client or language, or want requests that are human-readable, cacheable, and debuggable. The looser coupling is worth it when you don't control who's calling you.

**Message Queue** is the right tool when the producer and consumer don't need to be alive simultaneously, or when you need one event to fan out to many consumers without the producer knowing who they are. The asynchrony is the feature, not a limitation.

Choosing between them is about asking: does the caller need an immediate answer? Do I own both sides of this communication? Can this work happen later?

---

## 6. gRPC: The Modern Standard

**gRPC** is Google's open-source RPC framework and the de facto standard for internal microservice communication in high-scale systems.

It's built on two technologies that matter:

- **HTTP/2** — allows multiplexing multiple calls over a single TCP connection, bidirectional streaming, and header compression. This means many simultaneous RPC calls don't each need their own connection.
- **Protocol Buffers (Protobuf)** — a binary serialization format that is significantly smaller and faster to parse than JSON. You define your data schema in a `.proto` file; Protobuf handles the rest.

### What a gRPC service definition looks like

You define your service contract in a `.proto` file. This file is the source of truth — the RPC framework generates both client and server code from it:

```protobuf
// user_service.proto
service UserService {
  rpc GetProfile (UserRequest) returns (UserProfile);
  rpc ListUsers  (ListRequest) returns (stream UserProfile);
}

message UserRequest {
  string user_id = 1;
}

message UserProfile {
  string user_id  = 1;
  string name     = 2;
  string email    = 3;
}
```

The developer never writes serialization code. The `.proto` file is also the API contract between teams — change it carefully, because both sides depend on it.

### gRPC streaming modes

One meaningful advantage gRPC has over REST is native support for streaming:

| Mode | Description | Use Case |
|------|-------------|----------|
| Unary | One request → one response | Standard function call |
| Server streaming | One request → stream of responses | Live feed, log tailing |
| Client streaming | Stream of requests → one response | File upload, sensor data |
| Bidirectional streaming | Stream both ways simultaneously | Chat, real-time collaboration |

---

## 7. Failure Modes You Must Understand

This is where network abstraction gets genuinely hard. When an RPC call fails, it fails in ambiguous ways that local calls never do.

There are three possible outcomes of any RPC call:

```
RPC Call Sent
      │
      ├──► ✅ Success
      │       Server received, processed, responded. Easy.
      │
      ├──► ❌ Fail before execution
      │       Network failed before server received anything.
      │       Safe to retry — server never saw the request.
      │
      └──► ❓ Fail during or after execution
              The dangerous case.
              Did the server receive it?
              Did it process it?
              Did only the response get lost?
              You don't know. Retrying might duplicate the action.
```

The third case — **ambiguous failure** — is the hard one. Your payment was charged and the response was lost. Do you retry and charge again? Do you give up and tell the user it failed when it didn't? Neither is good without a deliberate strategy for handling this.

This is why the patterns below exist.

### Timeouts
Never make an RPC call without a timeout. A hung server without a timeout causes the caller to wait indefinitely, holding threads and connections until the entire caller is exhausted and fails too. This is how one slow service cascades into a full system outage. Set timeouts everywhere.

### Retries with Exponential Backoff
On failure, wait before retrying, and increase the wait each time (1s → 2s → 4s → 8s). Add random jitter so thousands of clients don't all retry at the same millisecond and overwhelm an already struggling service.

### Circuit Breaker
If a downstream service is consistently failing, stop sending it requests entirely for a cooldown period rather than hammering it into complete failure. The circuit breaker pattern gives the struggling service time to recover.

```
         ┌─────────┐   failures > threshold   ┌──────────┐
         │  CLOSED │ ─────────────────────────► │   OPEN   │
         │ (normal)│                            │(no calls)│
         └─────────┘                            └──────────┘
               ▲                                      │
               │      success                         │ cooldown expires
               │   ┌────────────┐                     │
               └── │ HALF-OPEN  │ ◄───────────────────┘
                   │(probe calls)│
                   └────────────┘
```

**Closed** — normal operation, traffic flows freely.
**Open** — too many failures detected, calls are blocked immediately without trying.
**Half-Open** — cooldown expired, a small number of probe calls go through to test if the service recovered.

---

## 8. Idempotency: The Key to Safe Retries

**An operation is idempotent if calling it multiple times produces the same result as calling it once.**

This is the solution to the ambiguous failure problem. If your operations are idempotent, retrying after any failure is always safe — even if you don't know whether the first attempt succeeded.

| Operation | Idempotent? | Why |
|-----------|-------------|-----|
| `GET /users/123` | ✅ Yes | Reading doesn't change state |
| `DELETE /users/123` | ✅ Yes | Deleting something already deleted is a no-op |
| `PUT /users/123 {name: "Isa"}` | ✅ Yes | Setting a value to the same value is a no-op |
| `POST /orders` | ❌ No | Calling twice creates two orders |
| `POST /payment/charge` | ❌ No | Charging twice bills the customer twice |

### Making non-idempotent operations safe: Idempotency Keys

For operations that are inherently non-idempotent (like payments), you can add an **idempotency key** — a unique ID the client generates and sends with the request. The server stores it, and if it sees the same key again, returns the original response without re-executing.

```
Client sends:
  POST /payments
  Idempotency-Key: "order-98765-attempt-1"
  Body: { amount: 50.00, user: "isa" }

Server logic:
  Have I seen "order-98765-attempt-1" before?
  → No:  process payment, store result under that key, return result
  → Yes: return the stored result, do NOT charge again
```

This pattern completely resolves the ambiguous failure problem for payments. The client can retry as many times as it wants. The server guarantees the operation only ever executes once. Stripe uses exactly this pattern in their production API.

---

## 9. Worked Example: Ride-Sharing Dispatch

Let's tie this all together in a real system. A rider requests a ride. The system needs to find and assign a driver.

```
Rider App
    │
    │  POST /rides  (REST — public-facing, broad compatibility)
    ▼
API Gateway
    │
    │  RideService.CreateRide(rideRequest)
    │  ← gRPC: internal, performance-sensitive, we own both sides
    ▼
Ride Service
    │
    ├──► DriverService.FindNearbyDrivers(location)
    │    ← gRPC, synchronous: caller needs the answer right now
    │      Returns: [driver_1, driver_2, driver_3]
    │
    ├──► DriverService.AssignDriver(rideId, driverId)
    │    ← gRPC with idempotency key = rideId
    │      Safe to retry: assigning the same driver twice is a no-op
    │
    └──► Message Queue (Kafka): "ride_created" event
         ← Async: Ride Service doesn't need to wait for these
           Consumers:
             - NotificationService  (push alerts to rider + driver)
             - BillingService       (pre-authorize payment)
             - AnalyticsService     (log the event for data pipeline)
```

Notice the reasoning behind each choice:

- **REST** at the public boundary — any mobile client, any language, cacheable, debuggable
- **gRPC** for internal calls — performance matters, type safety matters, we control both services
- **Idempotency key** on assignment — network between services can fail; we need retries to be safe
- **Message queue** for fan-out — three downstream services care about this event, but Ride Service shouldn't be coupled to any of them or blocked waiting for them

Every choice follows from a concrete constraint, not a preference.

---

## 10. Self-Check

1. What problem does RPC solve, and what complexity does it actually hide?
2. Walk through the lifecycle of an RPC call from function invocation to response.
3. What are the three possible outcomes of any RPC call? Which is dangerous, and why?
4. What is idempotency? Give one idempotent and one non-idempotent example, and explain *why* each is or isn't.
5. When would you choose gRPC over REST? REST over gRPC? A message queue over either?
6. What are the three states of a circuit breaker, and what triggers each transition?

---

## 11. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 4](https://dataintensive.net) | Deep dive on serialization, Protobuf, and RPC evolution across versions |
| 📝 [Fallacies of Distributed Computing — Peter Deutsch](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing) | The eight assumptions that break RPC's illusion — short and essential |
| 🔧 [gRPC Official Documentation](https://grpc.io/docs/) | Concepts, quickstarts, and language guides |
| 💳 [Stripe Idempotency Keys](https://stripe.com/docs/api/idempotent_requests) | Real-world idempotency in a production payment API — worth reading |
| 📬 [ByteByteGo — How do microservices communicate?](https://bytebytego.com) | Visual breakdown of RPC, REST, and messaging patterns |

---

*⬅️ Previous: [Abstraction](abstraction.md) &nbsp;|&nbsp; ➡️ Next: [Spectrum of Consistency Models](consistencyModels.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Preliminary System Design Concepts.</sub>