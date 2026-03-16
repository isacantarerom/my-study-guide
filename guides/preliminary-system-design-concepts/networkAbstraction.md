# 🌐 Network Abstraction: Remote Procedure Calls

> *"What if calling a function on another machine felt exactly like calling one on your own?"*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #1:** Preliminary Concepts

---

## Table of Contents

1. [Why Network Abstraction Exists](#1-why-network-abstraction-exists)
2. [What Is an RPC?](#2-what-is-an-rpc)
3. [How an RPC Call Actually Works](#3-how-an-rpc-call-actually-works)
4. [The Illusion and Its Limits](#4-the-illusion-and-its-limits)
5. [RPC vs REST vs Message Queues](#5-rpc-vs-rest-vs-message-queues)
6. [gRPC: The Modern Standard](#6-grpc-the-modern-standard)
7. [Failure Modes You Must Know](#7-failure-modes-you-must-know)
8. [Idempotency: The Key to Safe Retries](#8-idempotency-the-key-to-safe-retries)
9. [Applying This in an Interview](#9-applying-this-in-an-interview)
10. [Worked Example: Ride-Sharing Dispatch](#10-worked-example-ride-sharing-dispatch)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. Why Network Abstraction Exists

In [Abstraction](abstraction.md), we established that abstraction hides complexity behind a simpler interface. Network abstraction takes that idea and applies it to one of the hardest problems in distributed systems: **making two machines talk to each other as if they were one.**

Without network abstraction, every service-to-service call would require engineers to manually handle:

- Opening and closing TCP sockets
- Serializing data into bytes and deserializing responses
- Setting timeouts and retrying on failure
- Handling partial failures (did the other side receive it or not?)
- Versioning the communication protocol

That's thousands of lines of boilerplate for every pair of services that need to communicate. In a system with 50 microservices, that's untenable.

**Network abstraction — and RPC specifically — hides all of that.** The engineer writes a function call. The framework handles the rest.

---

## 2. What Is an RPC?

**RPC stands for Remote Procedure Call.**

It's a communication model that lets a program call a function (procedure) on a *different machine* using the same syntax as a local function call. The network communication is entirely hidden from the developer.

```
// This looks like a local call...
UserProfile profile = userService.getProfile(userId);

// ...but userService lives on a completely different server,
// possibly in a different data center, on a different continent.
```

The calling service is called the **client**. The service being called is the **server**. The piece of code that makes this magic happen — that intercepts the function call, serializes it, sends it over the network, and returns the result — is called a **stub** (on the client side) and a **skeleton** (on the server side).

> 🏦 **Analogy:** Think of an ATM. You press "Withdraw $100" and money comes out. You have no idea whether that request was processed by a server in the same building or routed to a data center across the country. The ATM interface *abstracts* the banking network entirely.

---

## 3. How an RPC Call Actually Works

Under the hood, every RPC call goes through this lifecycle. Understanding this is what lets you reason about failure modes.

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
   Opens TCP connection (or reuses one)     ──────────────────►
   Sends bytes over the wire                                  │
                                                              ▼
                                            4. Server Stub receives bytes
                                               Deserializes into arguments
                                                              │
                                                              ▼
                                            5. Server executes the actual function
                                               getProfile(userId) runs locally
                                                              │
                                                              ▼
                                            6. Server Stub serializes the result
   ◄──────────────────                        Sends response bytes back
        │
        ▼
7. Client Stub deserializes the response
   Returns result to the developer as if
   it were a normal return value
        │
        ▼
8. Developer receives:
   UserProfile{ name: "Isa", ... }
```

Steps 2–7 are entirely invisible to the developer. That invisibility **is** the abstraction.

---

## 4. The Illusion and Its Limits

RPC creates a powerful illusion: that remote calls are the same as local calls. But this illusion has cracks, and knowing where it cracks is the difference between a junior and senior answer in interviews.

Peter Deutsch's **"Fallacies of Distributed Computing"** (1994) lists the assumptions developers mistakenly make when building distributed systems. Most of them are broken by the RPC abstraction leaking:

| Fallacy | Reality |
|---------|---------|
| The network is reliable | Packets drop. Connections reset. Routers fail. |
| Latency is zero | A local call takes nanoseconds. A network call takes milliseconds — 1,000,000× slower. |
| Bandwidth is infinite | Serializing large objects and sending them over a network has real cost. |
| The network is secure | Data in transit can be intercepted. TLS isn't optional. |
| Topology doesn't change | Servers go down. IPs change. DNS propagates slowly. |
| There is one administrator | In a microservices world, every team owns their service. No one owns the whole network. |

> ⚠️ **The core danger of RPC:** It makes remote calls *look* like local calls, which tempts developers to treat them the same. Local calls never fail mid-execution. Remote calls absolutely do — and in ambiguous ways.

---

## 5. RPC vs REST vs Message Queues

These are the three dominant patterns for service-to-service communication. You need to know when to use each one.

```
                    ┌──────────────────────────────────────────────┐
                    │         Communication Patterns               │
                    └──────────────────────────────────────────────┘

         RPC / gRPC              REST / HTTP             Message Queue
         ──────────              ──────────────          ─────────────
         Synchronous             Synchronous             Asynchronous
         Tight coupling          Loose coupling          Very loose coupling
         Fast, typed             Flexible, universal     Decoupled, resilient
         Internal services       Public APIs             Event-driven workflows
         "Call this function"    "Act on this resource"  "This event happened"
```

### When to reach for each one

**RPC / gRPC** — when two internal services need to communicate frequently, performance matters, and you control both ends of the connection. Good for: service meshes, internal microservices, streaming data.

**REST** — when you're building a public-facing API, need broad compatibility (any client, any language), or want human-readable and cache-friendly requests. Good for: public APIs, web/mobile backends, third-party integrations.

**Message Queue** — when the producer and consumer don't need to be alive at the same time, or when you need to fan out one event to many consumers. Good for: async jobs, notifications, event sourcing, decoupling services that scale independently.

> 💡 **Interview tip:** If an interviewer asks "how do your services communicate?" — don't just say "REST." Ask: Is this synchronous or async? Internal or external? Latency-sensitive? Your answer should follow from the constraints.

---

## 6. gRPC: The Modern Standard

**gRPC** is Google's open-source RPC framework and the de facto standard for internal microservice communication in high-scale systems.

It's built on two technologies:

- **HTTP/2** — allows multiplexing multiple calls over a single TCP connection, bidirectional streaming, and header compression
- **Protocol Buffers (Protobuf)** — a binary serialization format that is smaller and faster to parse than JSON

### What a gRPC definition looks like

You define your service in a `.proto` file — this is the **contract**, the abstraction boundary:

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

gRPC generates client and server code in your language of choice from this file. The developer never touches serialization.

### gRPC streaming modes

One major advantage gRPC has over REST is native support for streaming:

| Mode | Description | Use Case |
|------|-------------|----------|
| Unary | One request → one response | Standard function call (like REST) |
| Server streaming | One request → stream of responses | Live feed, log tailing |
| Client streaming | Stream of requests → one response | File upload, sensor data ingestion |
| Bidirectional streaming | Stream both ways simultaneously | Chat, real-time collaboration |

---

## 7. Failure Modes You Must Know

This is where network abstraction gets dangerous. When an RPC call fails, it fails in ambiguous ways that local calls never do. There are three possible outcomes of any RPC call:

```
RPC Call Sent
      │
      ├──► ✅ Success         — Server received, processed, responded. Easy.
      │
      ├──► ❌ Fail before     — Network failed before server received anything.
      │       execution          Safe to retry. Server never saw the request.
      │
      └──► ❓ Fail during /   — The scary one. Did the server receive it?
              after execution    Did it process it? Did only the response get lost?
                                 You don't know. Retrying might duplicate the action.
```

The third case — **ambiguous failure** — is why distributed systems are hard. It's also why idempotency matters so much.

### Key failure handling strategies

**Timeouts** — Never make an RPC call without a timeout. Without one, a hung server causes your client to wait forever, holding resources and eventually cascading into a full outage.

**Retries with exponential backoff** — On failure, wait before retrying. Double the wait each time (e.g., 1s → 2s → 4s → 8s). Add jitter (random variation) so thousands of clients don't all retry at the exact same moment.

**Circuit Breaker** — If a service is failing consistently, stop sending it requests for a period of time. This prevents a struggling service from being hammered into complete failure. After a cooldown, allow a small number of probe requests through to check recovery.

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

---

## 8. Idempotency: The Key to Safe Retries

**An operation is idempotent if calling it multiple times produces the same result as calling it once.**

This is the solution to the ambiguous failure problem. If your RPC operations are idempotent, retrying after a failure is always safe.

| Operation | Idempotent? | Why |
|-----------|-------------|-----|
| `GET /users/123` | ✅ Yes | Reading doesn't change state |
| `DELETE /users/123` | ✅ Yes | Deleting something already deleted is a no-op |
| `PUT /users/123 {name: "Isa"}` | ✅ Yes | Setting a value to the same value changes nothing |
| `POST /orders` | ❌ No | Calling twice creates two orders |
| `POST /payment/charge` | ❌ No | Charging twice bills the customer twice |

### Making non-idempotent operations safe: Idempotency Keys

For operations that are inherently non-idempotent (like payments), you can add an **idempotency key** — a unique ID the client generates and sends with the request. The server stores it and, if it sees the same key again, returns the original response without re-executing.

```
Client sends:
  POST /payments
  Idempotency-Key: "order-98765-attempt-1"
  Body: { amount: 50.00, user: "isa" }

Server checks: have I seen "order-98765-attempt-1" before?
  → No: process payment, store result under that key
  → Yes: return stored result, don't charge again
```

Stripe uses exactly this pattern in their API. You'll see it come up whenever payments or financial transactions appear in an interview.

---

## 9. Applying This in an Interview

When communication between services comes up in your design, walk through this mental checklist:

- [ ] **Sync or async?** Does the caller need to wait for a response, or can it fire and forget?
- [ ] **Internal or external?** Internal → gRPC. External/public → REST.
- [ ] **What's the failure model?** Name it: timeouts, retries, circuit breakers.
- [ ] **Are mutations idempotent?** If not, how are you making them safe to retry?
- [ ] **What's the contract?** Define the interface: what goes in, what comes out, what errors are possible.

Even if you don't go deep into every point, *raising* these questions signals that you think in systems, not just in happy paths.

---

## 10. Worked Example: Ride-Sharing Dispatch

Let's ground all of this in a real system. You're designing the dispatch component of a ride-sharing app (think Uber/Lyft). A rider requests a ride. The system needs to find and assign a driver.

```
Rider App
    │
    │  POST /rides  (REST — public-facing API)
    ▼
API Gateway
    │
    │  RideService.CreateRide(rideRequest)  ← gRPC, internal
    ▼
Ride Service
    ├──► DriverService.FindNearbyDrivers(location)  ← gRPC, sync, needs fast response
    │         Returns: [driver_1, driver_2, driver_3]
    │
    ├──► DriverService.AssignDriver(rideId, driverId)  ← gRPC, must be idempotent
    │         Idempotency key: rideId
    │         (retry-safe: assigning same driver twice is a no-op)
    │
    └──► Message Queue (Kafka)  ← async, fire and forget
              Event: "ride_created"
              Consumers:
                - NotificationService  (push to rider + driver)
                - BillingService       (pre-authorize payment)
                - AnalyticsService     (log the event)
```

Notice the pattern:
- **REST** at the public boundary (flexible, cacheable, universally compatible)
- **gRPC** for internal calls where latency and type safety matter
- **Message Queue** for fan-out events where multiple consumers care but Ride Service doesn't need to wait on them
- **Idempotency key** on the assignment to prevent double-assignment on retry

---

## 11. Self-Check

Answer these without looking:

1. What problem does RPC solve, and what complexity does it hide?
2. Walk through the lifecycle of an RPC call from function invocation to response.
3. What are the three possible outcomes of any RPC call? Which one is dangerous, and why?
4. What is idempotency? Give one example of an idempotent and one non-idempotent operation.
5. When would you choose gRPC over REST? REST over gRPC? A message queue over either?
6. What is a circuit breaker, and when does it activate?

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 4 (Encoding & Evolution)](https://dataintensive.net) | Deep dive on serialization, Protobuf, and RPC evolution |
| 📝 [Fallacies of Distributed Computing — Peter Deutsch](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing) | The eight assumptions that break RPC's illusion |
| 🔧 [gRPC Official Documentation](https://grpc.io/docs/) | Concepts, quickstarts, and language guides |
| 📬 [ByteByteGo — How do microservices communicate?](https://bytebytego.com) | Visual breakdown of RPC, REST, and messaging patterns |
| 💳 [Stripe Idempotency Keys](https://stripe.com/docs/api/idempotent_requests) | Real-world idempotency in a production payment API |

---

*⬅️ Previous: [Abstraction](abstraction.md) &nbsp;|&nbsp; ➡️ Next: [Spectrum of Consistency Models](consistencyModels.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Preliminary System Design Concepts.</sub>