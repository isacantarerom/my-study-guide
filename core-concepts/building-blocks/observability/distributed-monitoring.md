# 📊 Distributed Monitoring

> *"You can't manage what you can't measure. In a distributed system, you can't even see what you can't measure."*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Observability

---

## Table of Contents

1. [What Distributed Monitoring Does](#1-what-distributed-monitoring-does)
2. [The Three Types of Metrics](#2-the-three-types-of-metrics)
3. [How Metrics Are Collected](#3-how-metrics-are-collected)
4. [Alerting — Turning Metrics Into Action](#4-alerting--turning-metrics-into-action)
5. [Dashboards — Making Metrics Human-Readable](#5-dashboards--making-metrics-human-readable)
6. [The Four Golden Signals](#6-the-four-golden-signals)
7. [SLIs, SLOs, and SLAs](#7-slis-slos-and-slas)
8. [Distributed Tracing — Following a Request](#8-distributed-tracing--following-a-request)
9. [Monitoring Failure Modes](#9-monitoring-failure-modes)
10. [How Monitoring Connects to Other Building Blocks](#10-how-monitoring-connects-to-other-building-blocks)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What Distributed Monitoring Does

A distributed system running across dozens of services and hundreds of servers is completely opaque without instrumentation. When something goes wrong — and it always does — you need to know:

- Is the system healthy right now?
- Which component is degraded?
- How long has it been degraded?
- Is it getting better or worse?

Distributed monitoring is the infrastructure that answers these questions continuously, in real time, without waiting for a user to complain.

The alternative — finding out about problems from user complaints — is called **alert-by-complaint**. It's the worst possible monitoring strategy. By the time users report problems, many are already affected, the damage is done, and you have no data about what happened leading up to the failure.

Good monitoring tells you before users know.

---

## 2. The Three Types of Metrics

### Counters
A value that only ever increases. Resets to zero when the service restarts.

```
http_requests_total: 4,829,103  (total requests since startup)
errors_total: 1,247              (total errors since startup)
bytes_sent_total: 48,291,033,920 (total bytes sent)
```

Counters are most useful when you look at their **rate of change** — requests per second, errors per minute. The absolute number is less meaningful than how fast it's growing.

### Gauges
A value that can go up or down. Represents a current state.

```
active_connections: 342
memory_usage_bytes: 2,147,483,648
queue_depth: 1,847
cpu_usage_percent: 67.3
cache_hit_ratio: 0.94
```

Gauges represent snapshots of a system's current condition. They're what you look at to know the system's state right now.

### Histograms
Distributions of values over time. Record how many observations fell into each "bucket."

```
http_request_duration_seconds:
  bucket[0.01]: 12,847  (requests completed in <10ms)
  bucket[0.05]: 48,293  (requests completed in <50ms)
  bucket[0.1]:  89,341  (requests completed in <100ms)
  bucket[0.5]:  98,102  (requests completed in <500ms)
  bucket[1.0]:  98,847  (requests completed in <1s)
  bucket[inf]:  99,001  (all requests)
```

From histograms you can calculate percentiles: p50, p95, p99 — the latency metrics that matter. We covered why percentiles matter more than averages in [Percentiles & Latency Metrics](../../extras/percentiles-and-latency-metrics.md).

---

## 3. How Metrics Are Collected

Two models for getting metrics from services into a central monitoring system.

### Push Model
Services actively send metrics to a central collector at regular intervals.

```
Service A → every 10 seconds → sends metrics → Monitoring System
Service B → every 10 seconds → sends metrics → Monitoring System
Service C → every 10 seconds → sends metrics → Monitoring System
```

**Pros:** Simple for services. Works well across network boundaries and firewalls.
**Cons:** If the monitoring system is down, metrics are lost. Services must know the monitoring endpoint.

**Used by:** StatsD, InfluxDB, Datadog Agent.

### Pull Model (Scraping)
The monitoring system periodically fetches metrics from each service's exposed endpoint.

```
Monitoring System → every 15 seconds → scrapes /metrics → Service A
                  → every 15 seconds → scrapes /metrics → Service B
                  → every 15 seconds → scrapes /metrics → Service C
```

Each service exposes a `/metrics` endpoint in a standard format (Prometheus exposition format):
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 12847
http_requests_total{method="POST",status="201"} 3291
http_requests_total{method="GET",status="500"} 47
```

**Pros:** Monitoring system controls the collection schedule. Easy to see which services are up (if scrape fails, service is down). No buffering needed on the service side.
**Cons:** Monitoring system must be able to reach all services. Doesn't work well for short-lived jobs.

**Used by:** Prometheus (the most widely used monitoring system).

---

## 4. Alerting — Turning Metrics Into Action

Metrics are only useful if someone acts on them. Alerting is the system that watches metrics and notifies humans when something requires attention.

### Alert Anatomy

A good alert has four properties:

**Condition** — what metric threshold triggers the alert?
```
error_rate > 1%  for more than  5 minutes
p99_latency > 500ms  for more than  2 minutes
queue_depth > 10000  for more than  10 minutes
```

**Severity** — how urgent is this?
```
P1 (Critical): page the on-call engineer immediately, any time of day
P2 (High): page during business hours, Slack alert any time
P3 (Low): Slack notification only, ticket created for next sprint
```

**Runbook link** — what should the engineer do when they receive this alert?

**Description** — what is this alert measuring and why does it matter?

### Alert Fatigue — The Biggest Monitoring Problem

Alert fatigue occurs when too many alerts fire, most of them not actionable, and engineers start ignoring them. A monitoring system where engineers habitually dismiss alerts is worse than no monitoring — it creates false confidence while genuine emergencies go unnoticed.

**The test for a good alert:** would waking an engineer at 3am for this alert be justified? If not, it shouldn't be a P1.

```
Good alert: "Error rate on checkout service is 15% — users can't complete purchases"
→ Urgent, user-impacting, requires immediate action

Bad alert: "CPU usage on batch processing server is 85%"
→ CPU running high during batch jobs is expected and normal
→ This fires nightly, engineers ignore it
→ Alert fatigue builds
```

### Alert Design Principles

**Alert on symptoms, not causes.** Alert when users are affected, not when an internal metric is unusual.

```
Better: "checkout_error_rate > 5%"  (user-facing symptom)
Worse:  "database_cpu > 80%"        (internal cause — may or may not affect users)
```

**Use burn rates for SLO alerts.** Instead of "error rate > 1%," alert on how fast you're burning through your error budget. This prevents both false positives (brief spikes that don't matter) and missed gradual degradations.

**Reduce noise before adding alerts.** Every new alert should be evaluated for its signal-to-noise ratio. A noisy alert (fires frequently without requiring action) should be removed or tuned before adding new ones.

---

## 5. Dashboards — Making Metrics Human-Readable

Raw metrics in a database are not useful to humans responding to incidents. Dashboards visualize metrics in a way that lets engineers quickly understand system state.

### The Four Key Dashboard Types

**Service health dashboard** — one screen showing the health of a single service. Error rate, latency, throughput, saturation. The first thing you open when investigating a problem.

**System overview dashboard** — the big picture. All services, their dependencies, key metrics for each. Used for situational awareness during incidents.

**Capacity dashboard** — resource usage trends over time. Are we approaching limits? Do we need to provision more capacity? Used for planning.

**Business metrics dashboard** — connecting technical metrics to business outcomes. Orders per minute, active users, revenue per hour. Used by leadership and for correlating technical incidents with business impact.

### Dashboard Principles

**Show context, not just current state.** A metric of 500 errors/minute means nothing without knowing if that's normal, better than usual, or a 10× spike.

**Align time ranges.** All graphs on a dashboard should show the same time window. Comparing a 1-hour graph to a 24-hour graph during an incident is confusing.

**Put the most important metrics first.** Engineers scanning during an incident look at the top-left first. Put error rate and latency there.

---

## 6. The Four Golden Signals

Google's SRE book identified four signals that collectively describe the health of any service. If you instrument nothing else, instrument these.

### Latency
How long requests take. Track separately for successful and failed requests — a spike in errors that fail fast looks like improved latency in the average, masking the real problem.

```
p50_latency: 12ms   (typical user experience)
p99_latency: 340ms  (worst-case user experience)
p99_error_latency: 2ms  (errors fail fast — don't let these skew your latency)
```

### Traffic
How much demand is on the system. Requests per second, queries per second, messages per second.

```
http_requests_per_second: 4,823
active_websocket_connections: 12,847
messages_processed_per_second: 28,391
```

### Errors
The rate of failed requests. Track by error type — 5xx server errors, 4xx client errors, timeouts, and business logic failures (payment declined, validation failed).

```
error_rate_5xx: 0.2%   (server errors — our fault)
error_rate_4xx: 1.8%   (client errors — usually expected)
timeout_rate: 0.05%    (requests that took too long)
```

### Saturation
How "full" the system is — how close to its capacity limit. CPU, memory, disk, connection pool usage.

```
cpu_utilization: 67%
memory_utilization: 78%
connection_pool_utilization: 45%
disk_utilization: 52%
```

Saturation is your leading indicator. If saturation is at 90%, latency and errors will soon follow. Address saturation before it becomes a user-facing problem.

---

## 7. SLIs, SLOs, and SLAs

These three terms are often confused. They form a hierarchy that connects technical metrics to business commitments.

**SLI (Service Level Indicator)** — a quantitative measurement of service behavior. A metric.
```
SLI: "the percentage of requests that complete successfully in under 200ms"
Current value: 99.7%
```

**SLO (Service Level Objective)** — the target value for an SLI. An internal goal.
```
SLO: "99.5% of requests complete successfully in under 200ms, measured over 30 days"
```

**SLA (Service Level Agreement)** — a contractual commitment to customers. Usually less stringent than the SLO, with financial consequences for violation.
```
SLA: "99% of requests complete in under 500ms. Violation triggers service credits."
```

```
SLI measures reality.
SLO sets the internal target.
SLA commits to customers.

SLO is stricter than SLA so violations are caught internally before customers are affected.
```

**Error budget** — the inverse of the SLO. If your SLO is 99.5% availability, your error budget is 0.5% — the amount of downtime you're "allowed" before violating the SLO.

```
Monthly error budget = (1 - 0.995) × 30 days × 24 hours × 60 minutes
                     = 0.005 × 43,200 minutes
                     = 216 minutes of allowed downtime per month

If you've used 180 minutes in the first 2 weeks:
→ Remaining budget: 36 minutes for the next 2 weeks
→ Freeze risky deployments
→ Focus on reliability improvements
```

---

## 8. Distributed Tracing — Following a Request

In a microservices system, a single user request may touch 10 services. When something is slow, which service added the latency? Distributed tracing answers this.

A **trace** is the complete journey of a request through the system. It's composed of **spans** — individual units of work within each service.

```
User Request: GET /checkout  (total: 450ms)
│
├── API Gateway                    5ms
│
├── Auth Service                   15ms
│
├── Cart Service                   25ms
│   └── Database query             20ms
│
├── Inventory Service              200ms  ← SLOW
│   ├── Cache lookup               1ms
│   └── Database query (miss)      195ms  ← ROOT CAUSE
│
├── Payment Service                180ms
│   └── External API call          175ms
│
└── Response Assembly              25ms
```

Without tracing, you'd know the checkout endpoint is slow. With tracing, you know exactly which service, which operation, and which dependency is the root cause.

Each request gets a unique **trace ID** at the entry point. This ID is passed through every service call (in HTTP headers, message metadata, etc.). Each service creates a span with its own **span ID** and records the trace ID. All spans for the same trace are stored together and can be visualized as the tree above.

**Real-world tools:** Jaeger, Zipkin, AWS X-Ray, Datadog APM.

---

## 9. Monitoring Failure Modes

### The Monitoring System Itself Fails
If your monitoring system is down, you're blind. Monitor your monitoring. Have separate, simple health checks that alert you if the main monitoring system stops working.

### Metric Cardinality Explosion
Labels/tags on metrics multiply storage and query cost. A metric with labels for `user_id`, `endpoint`, and `status_code` could have millions of unique combinations.

```
BAD:  http_requests_total{user_id="12345", endpoint="/api/orders", status="200"}
      → one time series per user — millions of unique series

GOOD: http_requests_total{endpoint="/api/orders", status="200"}
      → reasonable cardinality
```

High-cardinality labels explode the number of time series, making queries slow and storage expensive.

### Alert Fatigue (Revisited)
A monitoring system that engineers stop trusting is worse than no monitoring. Regularly audit alert quality: which alerts fired in the last 30 days, which were actionable, which were noise.

### Missing the Slow Burn
Monitoring is often calibrated for sudden failures. Gradual degradations — latency creeping up 5ms per day, error rate slowly increasing over weeks — can go unnoticed until they become crises.

**Solution:** Trend alerts — alert not just on absolute thresholds but on rate of change.

---

## 10. How Monitoring Connects to Other Building Blocks

```
Every Building Block ────────────────────────────────────────────────────►
  Exposes metrics: request rate, error rate, latency, saturation.
  Monitoring scrapes or receives these metrics continuously.

Distributed Cache ───────────────────────────────────────────────────────►
  Monitor: hit rate, eviction rate, memory utilization.
  Alert: hit rate drops below 80% (unexpected cache misses).

Message Queue / Pub-Sub ─────────────────────────────────────────────────►
  Monitor: queue depth, consumer lag, DLQ depth.
  Alert: consumer lag grows beyond acceptable threshold.

Distributed Task Scheduler ──────────────────────────────────────────────►
  Monitor: task completion rate, failure rate, DLQ depth.
  Alert: task failure rate exceeds threshold.

Distributed Logging ─────────────────────────────────────────────────────►
  Complement to monitoring — logs provide the detail behind metric anomalies.
  When a metric spikes, logs explain what happened.

Distributed Tracing ─────────────────────────────────────────────────────►
  When a latency metric spikes, traces show which service is responsible.
```

---

## 11. Self-Check

1. What are the three types of metrics? Give an example of each from a web application.
2. What is the difference between push and pull metric collection? Which does Prometheus use?
3. What are the Four Golden Signals? Why is saturation considered a "leading indicator"?
4. What is alert fatigue, and how do you prevent it?
5. What is the difference between SLI, SLO, and SLA? If your SLO is 99.9% availability over 30 days, how many minutes of downtime is your error budget?
6. What is distributed tracing, and what problem does it solve that metrics alone can't?
7. A user reports that checkout is slow. You open your monitoring dashboard. Walk through how you'd use the Four Golden Signals to narrow down where the problem is.

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) | The definitive guide to monitoring philosophy — free online |
| 🔧 [Prometheus Documentation](https://prometheus.io/docs/) | The most widely used open-source monitoring system |
| 📬 [ByteByteGo — Monitoring Design](https://bytebytego.com) | Visual breakdown of monitoring architecture |
| 📝 [Brendan Gregg — USE Method](https://www.brendangregg.com/usemethod.html) | Systematic approach to finding performance bottlenecks |

---

*⬅️ Previous: [Observability Overview](Observability.md) &nbsp;|&nbsp; ➡️ Next: [Monitor Server-Side Errors](monitor-server-side-errors.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Observability.</sub>