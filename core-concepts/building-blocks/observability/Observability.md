# 🔭 Observability

> *"You can't fix what you can't see. And in a distributed system, you can't see anything without deliberately building the tools to look."*

---

## The Problem This Group Solves

A distributed system running across dozens or hundreds of servers, processing millions of requests per second, is completely opaque without observability infrastructure. When something goes wrong — and it will — you need to be able to answer:

- Is the system healthy right now? (Monitoring)
- What went wrong on the server side? (Server-Side Error Monitoring)
- What went wrong on the client side? (Client-Side Error Monitoring)
- What exactly happened, step by step, leading up to the failure? (Logging)

Without these four building blocks, operating a distributed system in production is like flying blind. You know the plane is in the air, but you have no instruments. When something goes wrong, you're guessing.

Observability is what turns "something is broken" into "this specific service started returning 500 errors at 14:32:07 UTC when the database connection pool was exhausted because a slow query on the recommendations table held connections for 8 seconds each."

---

## The Components

| Building Block | Answers | Guide |
|---------------|---------|-------|
| **Distributed Monitoring** | "Is the system healthy right now? Are any metrics outside normal bounds?" | [Read →](distributed-monitoring.md) |
| **Monitor Server-Side Errors** | "What errors are happening inside my services, and how often?" | [Read →](monitor-server-side-errors.md) |
| **Monitor Client-Side Errors** | "What errors are users experiencing in their browsers and apps?" | [Read →](monitor-client-side-errors.md) |
| **Distributed Logging** | "What exactly happened, in what order, across which services?" | [Read →](distributed-logging.md) |

---

## How the Four Work Together

These aren't alternatives — they're complementary layers of visibility that each tell you something different:

```
Something is wrong...

MONITORING alerts you: "p99 latency spiked to 3,200ms at 14:32"
                              │
                              ▼ (you investigate)
SERVER-SIDE ERRORS show you: "Payment service throwing NullPointerException
                               at rate of 2,400/minute starting 14:31"
                              │
                              ▼ (you dig deeper)
LOGGING tells you: "14:31:47 - UserService called PaymentService
                    14:31:47 - PaymentService fetched user account: null
                    14:31:47 - NullPointerException: account.getBalance()
                    14:31:47 - Request failed after 0ms"
                              │
                              ▼ (root cause found)
"A database migration at 14:30 nulled the account field for 
 users created before 2020. 847,000 users affected."

CLIENT-SIDE ERRORS show: "Users are seeing 'Payment failed' modal,
                           bounce rate on checkout page up 340%"
```

Each layer narrows down the problem. Monitoring tells you something is wrong. Error monitoring tells you where. Logging tells you what exactly happened. Client-side monitoring tells you what users are experiencing as a result.

---

## The Three Pillars of Observability Revisited

We introduced these in [Maintainability](../../non-functional-system-characteristics/maintainability.md). The building blocks in this group implement them:

```
Metrics  → Distributed Monitoring
           (numerical measurements over time — latency, error rate, CPU)

Logs     → Distributed Logging
           (event records — what happened, when, with what context)

Traces   → Spans across Server-Side + Client-Side Error Monitoring
           (the path a single request took through the system)
```

A well-observable system has all three. Missing any one of them creates blind spots that make incidents harder to diagnose and resolve.

---

## The Cost of Skipping Observability

Observability infrastructure feels like overhead — it doesn't ship features, it doesn't serve users directly, it consumes storage and processing. Teams under pressure to move fast often defer it.

This is one of the most expensive mistakes in distributed systems engineering. The cost isn't paid when observability is skipped — it's paid during every production incident that takes 4 hours to debug instead of 20 minutes, and every outage whose root cause is never fully understood and therefore repeats.

Observability is not a feature you add later. By the time you need it, it's too late to build it.

---

## When You Reach for This Group

Any production system → **Distributed Monitoring** + **Distributed Logging** are non-negotiable.

Your system has multiple services → **Server-Side Error Monitoring** to track errors across all of them centrally.

Your system has a frontend (web or mobile) → **Client-Side Error Monitoring** to see what users are actually experiencing.

You're designing a new system → plan observability into the architecture from day one, not as an afterthought.

---

*⬅️ Previous Group: [Processing](../processing/Processing.md) &nbsp;|&nbsp; ➡️ Next Group: [Discovery](../discovery/Discovery.md)*