# ⚡ gRPC vs REST (and Protobuf vs JSON)

> *"When services talk to each other thousands of times per second, efficiency stops being optional."*

**⏱ Reading time:** ~6 minutes &nbsp;|&nbsp; **📍 Series:** Extras &nbsp;|&nbsp; **Referenced from:** Throughput & Latency

---

## Table of Contents

1. [What Problem This Solves](#1-what-problem-this-solves)
2. [The Two Stacks Compared](#2-the-two-stacks-compared)
3. [What is REST + JSON](#3-what-is-rest--json)
4. [What is gRPC + Protobuf](#4-what-is-grpc--protobuf)
5. [Why gRPC is Faster](#5-why-grpc-is-faster)
6. [When to Use Each](#6-when-to-use-each)
7. [What Good Looks Like](#7-what-good-looks-like)

---

## 1. What Problem This Solves

When systems grow, they stop being a single service and become **many services talking to each other**.
API → Auth → User → Feed → Database


Each hop adds:
- Network cost
- Serialization/deserialization cost
- Latency

At scale, even tiny inefficiencies multiply.

---

## 2. The Two Stacks Compared

Think in layers:

| Layer        | REST stack        | gRPC stack        |
|--------------|------------------|------------------|
| Protocol     | HTTP/1.1         | HTTP/2           |
| Data format  | JSON (text)      | Protobuf (binary)|
| Style        | Flexible         | Strict (schema)  |

---

## 3. What is REST + JSON

This is the **most common approach**, especially for public APIs.

### REST
- Uses HTTP (GET, POST, etc.)
- Resource-based (URLs)

### JSON
- Text-based
- Human-readable
- Flexible structure

Example:

```json
{
  "userId": 123,
  "name": "Isa"
}
```

Strengths:
Easy to debug
Widely supported
Great for external clients

---

## 4. What is gRPC + Protobuf

This is optimized for internal service communication.


gRPC
Remote procedure calls (RPC)
Built on HTTP/2
Supports streaming
Protobuf (Protocol Buffers)
Binary format
Requires a schema definition


Example schema:
```
message User {
  int32 user_id = 1;
  string name = 2;
}
```

Data is sent as compact binary — not human-readable.

---

## 5. Why gRPC is Faster

1. Smaller payloads

- JSON repeats field names:

```
"userId": 123
```

- Protobuf encodes compactly:
```
[1][123]
```

→ Less data over the network


2. Faster parsing

- JSON:

```
string → parse → interpret → object
```

- Protobuf:

```
binary → decode → object
```

→ Less CPU work

3. Better networking (HTTP/2)

- HTTP/1.1:
```
Request → Response → Request → Response
```

- HTTP/2:
```
Multiple requests in parallel over one connection
```

→ Lower latency and better throughput

---

## 6. When to Use Each

*Use REST + JSON when:*

- Public APIs
- Browser/mobile clients
- Human readability matters
- Flexibility is needed

*Use gRPC + Protobuf when:*

- Internal microservices
- High request volume
- Low latency is critical
- Strong contracts are preferred

---

## 7. What Good Looks Like

Modern systems often use both:

- External (clients) → REST + JSON
- Internal (services) → gRPC + Protobuf

---

## Final intuition

- REST is designed for humans and flexibility
- gRPC is designed for machines and efficiency

---

*↩ Back to [Throughput & Latency](../non-functional-system-characteristics/throughputAndLatency.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Extras.</sub>

