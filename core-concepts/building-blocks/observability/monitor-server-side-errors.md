# 🔴 Monitor Server-Side Errors

> *"Every error your server throws is a conversation your system is trying to have with you. Error monitoring is how you listen."*

**⏱ Reading time:** ~10 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Observability

---

## Table of Contents

1. [What Server-Side Error Monitoring Does](#1-what-server-side-error-monitoring-does)
2. [What Counts as a Server-Side Error](#2-what-counts-as-a-server-side-error)
3. [Error Capture — How Errors Are Collected](#3-error-capture--how-errors-are-collected)
4. [Error Enrichment — Context Is Everything](#4-error-enrichment--context-is-everything)
5. [Error Grouping and Deduplication](#5-error-grouping-and-deduplication)
6. [Error Rate vs Error Volume](#6-error-rate-vs-error-volume)
7. [The Error Lifecycle](#7-the-error-lifecycle)
8. [Alerting on Errors](#8-alerting-on-errors)
9. [How Server-Side Error Monitoring Connects to Other Building Blocks](#9-how-server-side-error-monitoring-connects-to-other-building-blocks)
10. [Self-Check](#10-self-check)
11. [References](#11-references)

---

## 1. What Server-Side Error Monitoring Does

Metrics tell you *that* errors are happening — the error rate is 2%, up from 0.1% five minutes ago. But metrics don't tell you *what* errors are happening, *where* in the code they're occurring, or *why*.

Server-side error monitoring fills this gap. It captures individual error events — the exception type, the stack trace, the request context, the user affected — and aggregates them into an actionable view: which errors are most common, which are new, which are getting worse.

Without error monitoring, debugging a production incident means:
- Searching through logs hoping to find a relevant error
- Guessing which code path is failing
- Having no idea how many users are affected

With error monitoring, you open a dashboard and immediately see: "NullPointerException in PaymentService.charge() — affecting 3% of checkout requests — started 7 minutes ago — 2,347 occurrences."

---

## 2. What Counts as a Server-Side Error

### HTTP 5xx Errors
The most visible category. The server encountered a problem it couldn't recover from.

```
500 Internal Server Error  → unhandled exception, bug
502 Bad Gateway            → upstream service returned invalid response
503 Service Unavailable    → service is overloaded or down
504 Gateway Timeout        → upstream service didn't respond in time
```

5xx errors are always worth monitoring — they represent failures that are the system's responsibility, not the user's.

### Unhandled Exceptions
Code that throws an exception that wasn't caught and handled. These may or may not result in an HTTP 5xx — they could also crash a background worker, corrupt a data pipeline, or silently fail.

```python
# This exception might not produce a 5xx if caught at a high level
# but still represents an error worth knowing about
def process_payment(order):
    account = get_account(order.user_id)  # returns None for deleted users
    charge = account.balance - order.total  # NullPointerException here
```

### Slow Queries and Timeouts
When a database query takes 10 seconds instead of 10 milliseconds, that's an error even if it eventually succeeds. Timeout errors — when the system gives up waiting — are especially important because they propagate: Service A times out waiting for Service B, causing A to fail too.

### Business Logic Errors
Not all errors are exceptions. Some are expected failures that still warrant monitoring:

```
Payment declined rate: 8%  (normal is 3% — is there a fraud wave?)
Inventory reservation failures: 15%  (is stock running out?)
Authentication failures: 500/minute  (is someone brute-forcing logins?)
```

These aren't code bugs, but anomalies in business metrics can signal system problems or attacks.

---

## 3. Error Capture — How Errors Are Collected

### SDK Integration
The most common approach. An error monitoring SDK (Sentry, Datadog, Rollbar) is integrated into the application. It automatically captures unhandled exceptions and can be used to capture handled errors explicitly.

```python
import sentry_sdk

# Automatic capture of unhandled exceptions
sentry_sdk.init(dsn="https://...")

# Manual capture of handled errors
try:
    charge_card(order)
except PaymentDeclinedException as e:
    sentry_sdk.capture_exception(e)
    return {"error": "payment_declined"}
```

### Middleware / Interceptor Pattern
A global error handler in the web framework catches all unhandled exceptions before they reach the HTTP response layer.

```python
@app.middleware("http")
async def error_handler(request, call_next):
    try:
        return await call_next(request)
    except Exception as e:
        error_monitor.capture(e, request_context=request)
        return JSONResponse({"error": "internal_error"}, status_code=500)
```

This ensures every unhandled exception is captured, even if individual endpoints don't have explicit error handling.

### Log Parsing
Some teams capture errors by parsing logs — running a log processing pipeline that identifies error patterns and sends them to an error tracking system. Less precise than SDK integration (stack traces may be incomplete) but works with legacy systems that can't be easily modified.

---

## 4. Error Enrichment — Context Is Everything

A bare exception with a stack trace is useful. An exception with full context is invaluable.

**What to capture with every error:**

```
Error: NullPointerException at PaymentService.charge() line 47

Context:
  user_id: 12345
  order_id: 98765
  order_total: 149.99
  user_account_type: "free"
  request_id: "req_abc123"  ← links to trace
  environment: "production"
  server: "app-server-07"
  timestamp: 2024-01-15T10:30:00Z
  git_commit: "a3f2c1b"  ← which version of code
  
Request:
  method: POST
  path: /api/v2/orders/98765/charge
  headers: {authorization: "[redacted]", user-agent: "..."}
  
Stack trace:
  PaymentService.charge() line 47
  OrderController.processPayment() line 123
  ...
```

With this context, debugging becomes specific: "This only fails for free-tier users — probably a null check on the account type." Without context, you have a stack trace pointing to a line of code and no idea what data caused it.

**Important:** Redact sensitive fields before capturing. Never log passwords, credit card numbers, SSNs, or authentication tokens. Error monitoring systems are not designed as secure vaults.

---

## 5. Error Grouping and Deduplication

If an error occurs 10,000 times, you don't want 10,000 separate alerts — you want one alert saying "this error occurred 10,000 times."

Error monitoring systems group similar errors into **issues**. Two occurrences are grouped together if they represent the same error:

```
Occurrence 1: NullPointerException at PaymentService.charge():47
Occurrence 2: NullPointerException at PaymentService.charge():47
Occurrence 3: NullPointerException at PaymentService.charge():47

→ Grouped into one issue:
  NullPointerException at PaymentService.charge():47
  First seen: 10:23:00
  Last seen: 10:30:47
  Occurrences: 3,847
  Users affected: 231
```

**Grouping fingerprint** — the algorithm that determines whether two errors are "the same." Typically based on error type + key frames of the stack trace. Tunable to avoid over-grouping (different errors merged) or under-grouping (one error split into hundreds of issues).

**Regression detection** — if an issue was marked as resolved but starts occurring again, the system automatically reopens it and alerts. This catches cases where a fix was deployed but didn't actually fix the underlying issue.

---

## 6. Error Rate vs Error Volume

Both matter but for different reasons.

**Error rate** (errors / total requests) tells you the health of the system relative to its load. A rate of 0.1% means 1 in 1,000 requests fails — regardless of whether you're getting 100 requests/second or 100,000.

**Error volume** (absolute count of errors) tells you the impact. At 0.1% error rate:
- 100 req/sec → 0.1 errors/sec → 6 errors/minute (low impact)
- 100,000 req/sec → 100 errors/sec → 6,000 errors/minute (significant impact)

```
Alert on rate when: you want to know if the system is degrading
Alert on volume when: you want to know how many users are affected

Best practice: alert on both with different thresholds and severities
```

A high error rate during very low traffic (e.g., a batch job at 4am) may not warrant waking anyone up. The same error rate during peak traffic affecting thousands of users per minute absolutely does.

---

## 7. The Error Lifecycle

Errors in a monitoring system move through states:

```
NEW ────────────────────────────────────────────────────────────►
  First time this error has been seen.
  Alert fires. Engineer is notified.
        │
        ▼
INVESTIGATED ───────────────────────────────────────────────────►
  Engineer has looked at it. Assigned to a developer.
        │
        ▼
IN PROGRESS ────────────────────────────────────────────────────►
  Fix is being developed.
        │
        ├──► RESOLVED (fix deployed, no new occurrences)
        │
        └──► REGRESSED (fix deployed but error returned)
                  │
                  ▼
             NEW (cycle restarts, alert fires again)
```

Tracking error lifecycle is what makes error monitoring actionable rather than just observational. Errors that are acknowledged but never resolved accumulate into technical debt. A healthy error monitoring practice includes regular triage — reviewing new errors, prioritizing fixes, closing old ones.

---

## 8. Alerting on Errors

Error alerting should be tiered:

**Immediate alert (P1):** new error type appearing for the first time in production + affecting more than 1% of requests.

**Urgent alert (P2):** existing error rate spiking significantly above its baseline.

**Daily digest (P3):** list of new errors seen in the past 24 hours, even low-volume ones.

**The new error rule:** any error type appearing for the first time in production should be immediately investigated, even if it's only happening once. A single occurrence of a new error often indicates a code path that was never tested, or a newly introduced bug that will get worse as traffic increases.

---

## 9. How Server-Side Error Monitoring Connects to Other Building Blocks

```
Distributed Monitoring ──────────────────────────────────────────────────►
  Monitoring shows error rate spiking.
  Error monitoring shows which specific errors are causing the spike.
  Complementary: monitoring shows the "what", error monitoring shows the "which".

Distributed Logging ─────────────────────────────────────────────────────►
  Error monitoring captures structured error events.
  Logging captures the surrounding context — what happened before the error.
  Request ID links error events to log lines for full context.

Distributed Tracing ─────────────────────────────────────────────────────►
  Trace ID links an error event to the full distributed trace.
  "This NullPointerException happened during this specific checkout request
   that touched these 8 services."

Message Queue / Task Scheduler ─────────────────────────────────────────►
  Background job errors are invisible without explicit error capture.
  No HTTP response means no 5xx counter — you must instrument workers
  explicitly to capture and report their errors.
```

---

## 10. Self-Check

1. What is the difference between error monitoring and the error rate metric in general monitoring?
2. Why is context (user ID, request ID, environment) more important than the stack trace alone?
3. What is error grouping, and why is it essential at high error volumes?
4. Why should you alert on both error rate AND error volume? When would you want to alert on one but not the other?
5. A background job that sends daily email digests starts failing silently. It's not an HTTP service so there are no 5xx metrics. How do you detect and monitor this?
6. An error is marked as resolved after a fix is deployed. Three days later, it starts occurring again. What should your error monitoring system do, and what does this indicate?
7. You deploy a new feature at 2pm. At 2:15pm, a new error type appears: "IndexOutOfBoundsException in RecommendationService." It's only occurring 3 times per minute, affecting 0.003% of requests. Do you alert on this? Why?

---

## 11. References

| Resource | Why it's worth it |
|----------|-------------------|
| 🔧 [Sentry Documentation](https://docs.sentry.io/) | The most widely used error monitoring platform — excellent conceptual docs |
| 📘 [Google SRE Book — Being On-Call](https://sre.google/sre-book/being-on-call/) | How to operationalize error response effectively |
| 📬 [ByteByteGo — Observability Stack](https://bytebytego.com) | How metrics, logs, traces, and error monitoring work together |

---

*⬅️ Previous: [Distributed Monitoring](distributed-monitoring.md) &nbsp;|&nbsp; ➡️ Next: [Monitor Client-Side Errors](monitor-client-side-errors.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Observability.</sub>