# 📋 Distributed Logging

> *"Metrics tell you something is wrong. Errors tell you what is wrong. Logs tell you the story of how it got that way."*

**⏱ Reading time:** ~11 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Observability

---

## Table of Contents

1. [What Distributed Logging Does](#1-what-distributed-logging-does)
2. [Structured vs Unstructured Logs](#2-structured-vs-unstructured-logs)
3. [Log Levels — Severity and Signal](#3-log-levels--severity-and-signal)
4. [What to Log — and What Not To](#4-what-to-log--and-what-not-to)
5. [Correlation IDs — The Thread That Connects Everything](#5-correlation-ids--the-thread-that-connects-everything)
6. [Log Aggregation — Collecting Logs at Scale](#6-log-aggregation--collecting-logs-at-scale)
7. [Log Storage and Retention](#7-log-storage-and-retention)
8. [Searching and Querying Logs](#8-searching-and-querying-logs)
9. [Logging Failure Modes](#9-logging-failure-modes)
10. [How Logging Connects to Other Building Blocks](#10-how-logging-connects-to-other-building-blocks)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What Distributed Logging Does

Logs are the detailed record of what happened in a system — a timestamped sequence of events, decisions, errors, and state changes. While metrics tell you *that* something changed and error monitoring tells you *what* broke, logs tell you the *story* — the sequence of events leading to a failure, with enough detail to reconstruct exactly what happened.

In a single-server application, logs live in a file on that server. You SSH in and read them. In a distributed system with 200 servers each producing logs, you need infrastructure to collect those logs, ship them to a central location, and make them searchable. That infrastructure is distributed logging.

Without centralized logging, investigating an incident across a microservices system means:
- SSH-ing into dozens of servers
- Manually searching through log files
- Trying to correlate timestamps across services
- Hoping the relevant logs haven't been rotated out

With distributed logging, you open a search interface, type a request ID, and see every log line from every service that touched that request — in chronological order.

---

## 2. Structured vs Unstructured Logs

This is the most important design decision in logging.

### Unstructured Logs
Plain text strings. Human-readable but machine-unfriendly.

```
[2024-01-15 10:30:47] INFO  User 12345 placed order 98765 for $149.99
[2024-01-15 10:30:47] ERROR Failed to charge card for order 98765: timeout
[2024-01-15 10:30:48] INFO  Retrying payment for order 98765, attempt 2
```

Humans can read this. But to search for "all errors related to order 98765" programmatically, you'd need regex parsing, which is fragile and breaks when the format changes.

### Structured Logs
Logs emitted as JSON (or another structured format). Every field is a named key-value pair.

```json
{
  "timestamp": "2024-01-15T10:30:47.123Z",
  "level": "ERROR",
  "message": "Failed to charge card",
  "service": "payment-service",
  "order_id": 98765,
  "user_id": 12345,
  "amount": 149.99,
  "error_type": "PaymentTimeout",
  "attempt": 1,
  "request_id": "req_abc123",
  "trace_id": "trace_xyz789",
  "duration_ms": 5001
}
```

Now searching becomes trivial:
```
order_id:98765 AND level:ERROR
user_id:12345 AND service:payment-service
error_type:PaymentTimeout AND attempt:>2
```

**Always use structured logging in production systems.** The marginally more verbose output pays for itself ten times over during incident investigation.

---

## 3. Log Levels — Severity and Signal

Log levels create a hierarchy for filtering and alerting. Every log statement has a level that indicates its severity.

```
TRACE   → Most granular. Every function call, every iteration.
          Use in development only. Never in production.
          "Entering processPayment() with params: {order_id: 98765}"

DEBUG   → Detailed diagnostic information.
          Useful for debugging specific issues in production when needed.
          "Cache lookup for user:12345 returned null — going to database"

INFO    → Normal operational events worth recording.
          The default level for production systems.
          "Order 98765 placed successfully — amount: $149.99"

WARN    → Something unexpected happened but the system recovered.
          Worth investigating during low-priority review.
          "Payment attempt 1 timed out — retrying"

ERROR   → Something failed and requires attention.
          Should trigger investigation soon.
          "Payment failed after 3 attempts — order 98765 moving to manual review"

FATAL   → Critical failure — service is about to crash or has crashed.
          Requires immediate response.
          "Database connection pool exhausted — service shutting down"
```

**In production, typically log at INFO level and above.** DEBUG and TRACE generate enormous log volumes that overwhelm storage and make searching slow. Enable DEBUG temporarily for specific investigations.

**The signal-to-noise ratio matters.** If ERROR logs are firing for expected conditions (a user entering wrong credentials logs as ERROR instead of INFO), engineers learn to ignore error logs — destroying the signal.

---

## 4. What to Log — and What Not To

### Log These

**Request entry and exit:**
```json
{"event": "request_received", "method": "POST", "path": "/api/orders", "request_id": "req_abc"}
{"event": "request_completed", "status": 201, "duration_ms": 47, "request_id": "req_abc"}
```

**State transitions:**
```json
{"event": "order_state_changed", "order_id": 98765, "from": "pending", "to": "paid"}
```

**External service calls:**
```json
{"event": "external_call", "service": "stripe", "method": "charge", "duration_ms": 230, "result": "success"}
```

**Errors and exceptions:**
```json
{"event": "error", "error_type": "PaymentTimeout", "order_id": 98765, "attempt": 2}
```

**Security events:**
```json
{"event": "login_failed", "user_email": "[redacted]", "ip": "192.168.1.1", "reason": "invalid_password"}
```

### Never Log These

**Passwords and credentials:**
```
BAD:  "Login attempt for user@email.com with password: abc123"
GOOD: "Login attempt for user@email.com — result: failed"
```

**Payment card data:**
```
BAD:  "Processing card: 4111-1111-1111-1111, CVV: 123"
GOOD: "Processing card ending in 1111"
```

**Personal health information (PHI):**
```
BAD:  "User 123 diagnosed with diabetes, prescription: metformin"
GOOD: "Prescription record updated for user 123"
```

**Authentication tokens and session IDs:**
```
BAD:  "Request with token: eyJhbGciOiJSUzI1NiJ9.eyJ1c2VyX2lkIjo..."
GOOD: "Authenticated request — user_id: 123"
```

Log what is needed for debugging. Redact everything that would be a liability if the logs were breached.

---

## 5. Correlation IDs — The Thread That Connects Everything

In a microservices system, a single user request touches many services. Each service generates its own logs. Without a shared identifier, correlating logs across services is impossible.

A **correlation ID** (also called request ID or trace ID) is a unique identifier generated at the entry point of a request and passed through every subsequent service call.

```
User → API Gateway
  → generates request_id: "req_abc123"
  → passes it in downstream calls

API Gateway → Order Service
  → passes X-Request-ID: req_abc123 in headers

Order Service → Payment Service
  → passes X-Request-ID: req_abc123 in headers

Payment Service → Stripe API
  → logs include request_id: req_abc123

All logs for this request:
  [API Gateway]      req_abc123  "POST /api/orders received"
  [Order Service]    req_abc123  "Creating order for user 12345"
  [Payment Service]  req_abc123  "Charging card — amount: 149.99"
  [Payment Service]  req_abc123  "ERROR: PaymentTimeout after 5001ms"
  [Order Service]    req_abc123  "Payment failed — order status: pending"
  [API Gateway]      req_abc123  "POST /api/orders → 500 after 5048ms"

Search: request_id:req_abc123
→ See the complete story in chronological order across all services
```

Correlation IDs are the single most important practice in distributed logging. Without them, cross-service debugging is guesswork. With them, incident investigation that would take hours takes minutes.

---

## 6. Log Aggregation — Collecting Logs at Scale

200 servers each writing to their own log files is useless for distributed investigation. Log aggregation collects all logs from all sources into a central searchable system.

### The Standard Pipeline

```
Service A → Log Agent → Message Queue → Log Processor → Log Storage → Search Interface
Service B → Log Agent ↗
Service C → Log Agent ↗

(e.g., application → Filebeat → Kafka → Logstash → Elasticsearch → Kibana)
```

**Log Agent (Filebeat, Fluentd, Vector)** — runs alongside each service. Reads log files (or receives logs via syslog/stdout), batches them, and ships them to the processing layer. Lightweight, handles back-pressure if the pipeline is slow.

**Message Queue (Kafka)** — optional but common in high-volume systems. Decouples log producers from consumers. Absorbs log bursts without dropping events. Provides replay capability.

**Log Processor (Logstash, Fluentd)** — parses, enriches, and routes logs. Adds fields (environment, datacenter), redacts sensitive data, filters out noise, routes different log types to different storage.

**Log Storage (Elasticsearch, Loki, Clickhouse)** — indexes logs for fast search. Elasticsearch is the most widely used; Grafana Loki is cheaper at scale (stores logs without full indexing, searches slower).

**Search Interface (Kibana, Grafana)** — the interface engineers use during incidents. Kibana (for Elasticsearch) and Grafana (for Loki) are the dominant choices.

---

## 7. Log Storage and Retention

Logs are high volume. A system generating 10,000 log lines per second at 500 bytes each produces 5 MB/second — 432 GB/day — 158 TB/year. Storage is a real cost.

**Retention tiers:**

```
Hot storage (0-7 days):
  Full-text searchable in Elasticsearch
  Fast queries, immediate access
  Expensive: ~$0.30/GB/month on cloud

Warm storage (7-90 days):
  Compressed, less frequently queried
  Slower queries (seconds not milliseconds)
  Medium cost: ~$0.05/GB/month

Cold storage (90+ days):
  Blob storage (S3 Glacier)
  Rarely accessed, requires restore before querying
  Cheap: ~$0.004/GB/month

Archive (1-7 years):
  Regulatory/compliance requirement for some industries
  Effectively write-only — only accessed for audits or legal holds
```

**What drives retention decisions:**
- Regulatory requirements (HIPAA requires 6 years, PCI-DSS requires 1 year)
- Incident investigation needs (most incidents are investigated within 7 days)
- Audit requirements (security logs often retained longer)
- Cost (longer retention = higher cost)

**Log sampling** — for extremely high-volume, low-value log types, only keep a percentage. Log 1% of successful INFO-level request logs but 100% of WARN and ERROR logs. Reduces storage cost while preserving the signals that matter.

---

## 8. Searching and Querying Logs

During an incident, the ability to search logs quickly is the difference between a 10-minute resolution and a 2-hour one.

### Common Search Patterns

**Find all events for a specific request:**
```
request_id:"req_abc123"
```

**Find all errors in the last hour from a specific service:**
```
service:"payment-service" AND level:"ERROR" AND timestamp:[now-1h TO now]
```

**Find all orders above $1000 that failed:**
```
order_total:>1000 AND event:"payment_failed"
```

**Find requests that took over 5 seconds:**
```
duration_ms:>5000 AND level:"INFO"
```

**Count errors by type in the last 24 hours:**
```
level:"ERROR" | stats count() by error_type
```

### Log-Based Metrics

Modern logging systems can generate metrics from log data. Instead of instrumenting your code with both logs and metrics separately:

```
# From logs, automatically derive:
# - request rate (count of request_completed events)
# - error rate (count where level=ERROR / count where event=request_completed)
# - p99 latency (histogram of duration_ms field)
```

This reduces instrumentation overhead and ensures metrics and logs are always consistent.

---

## 9. Logging Failure Modes

### Log Volume Overwhelming Storage
A bug causes excessive logging (infinite loop emitting logs, DEBUG accidentally enabled in production). Log volume spikes 100×, filling disks or exhausting Elasticsearch capacity.

**Prevention:** Log rate limiting per service. Alert on log volume anomalies. Test log levels before deployment.

### Sensitive Data in Logs
A developer logs a request body that contains a password or credit card number. The log is now in Elasticsearch, potentially replicated to multiple regions, accessible to anyone with log access.

**Prevention:** Pre-production log audits. Automated PII scanning in the log pipeline. Code review checklist including "what are we logging?"

### Missing Correlation IDs
Services that don't propagate request IDs make cross-service correlation impossible. This often happens with third-party libraries or legacy services that don't follow the correlation ID convention.

**Prevention:** Enforce correlation ID propagation in service templates. Middleware that automatically reads and propagates the ID.

### Log Aggregation Pipeline Failure
If the Kafka or Logstash layer fails, logs buffer on local agents. If buffering exceeds capacity, logs are dropped. You lose the log data for the outage you were trying to investigate.

**Prevention:** Adequate buffering capacity. Monitor the pipeline itself. Fall back to local log files when the pipeline is degraded.

---

## 10. How Logging Connects to Other Building Blocks

```
Distributed Monitoring ──────────────────────────────────────────────────►
  Monitoring shows metrics — error rate spiked at 10:30.
  Logs show the story — what happened between 10:29 and 10:31 in detail.
  Complementary: monitoring for detection, logging for investigation.

Server-Side Error Monitoring ────────────────────────────────────────────►
  Error monitoring captures structured error events.
  Logging captures surrounding context — the events before and after.
  Request ID links error events to full log context.

Distributed Tracing ─────────────────────────────────────────────────────►
  Traces show service-level timing and dependencies.
  Logs show detailed events within each service.
  Trace ID is embedded in logs — clicking a span in a trace opens its logs.

Message Queue / Pub-Sub ─────────────────────────────────────────────────►
  Log each message: published, received, processed, failed.
  Correlate across producer and consumer logs with message ID.
  Essential for debugging async workflows where the request path is non-linear.

Sequencer ────────────────────────────────────────────────────────────────►
  Correlation IDs often generated by the sequencer.
  Unique, time-ordered IDs make log correlation and chronological sorting natural.
```

---

## 11. Self-Check

1. What is the difference between structured and unstructured logging? Why does it matter in production?
2. What are the six log levels, in order of severity? Which level should most production logs default to?
3. What is a correlation ID, and why is it the single most important practice in distributed logging?
4. What types of data must never appear in logs? Why is this a security risk even in internal systems?
5. Describe the standard log aggregation pipeline from service to search interface. What does each component contribute?
6. An incident occurred at 2pm yesterday. Your logs are retained in "hot" storage for 7 days and "cold" storage for 90 days. Which storage tier do you query, and why?
7. A background job that processes user payouts is failing for some users but not others. There are no HTTP endpoints — it's a scheduled job. Walk through how you'd use logging to diagnose the problem.

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 🔧 [Elasticsearch Documentation](https://www.elastic.co/guide/index.html) | The ELK stack (Elasticsearch + Logstash + Kibana) is the most widely used logging infrastructure |
| 📘 [Designing Data-Intensive Applications — Ch. 11](https://dataintensive.net) | Event logs and their role in distributed system design |
| 📬 [ByteByteGo — Logging Best Practices](https://bytebytego.com) | Visual breakdown of distributed logging architecture |
| 📝 [The Twelve-Factor App — Logs](https://12factor.net/logs) | How logs should be treated in modern cloud-native applications |

---

*⬅️ Previous: [Monitor Client-Side Errors](monitor-client-side-errors.md) &nbsp;|&nbsp; ➡️ Next Group: [Discovery](../discovery/Discovery.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Observability.</sub>