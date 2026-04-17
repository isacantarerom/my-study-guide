# ⏰ Distributed Task Scheduler

> *"Some work needs to happen now. Some work needs to happen later. Some work needs to happen every Tuesday at 3am. A task scheduler handles all three."*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Processing

---

## Table of Contents

1. [What a Distributed Task Scheduler Does](#1-what-a-distributed-task-scheduler-does)
2. [The Three Types of Tasks](#2-the-three-types-of-tasks)
3. [Core Components of a Task Scheduler](#3-core-components-of-a-task-scheduler)
4. [Task Lifecycle](#4-task-lifecycle)
5. [Scheduling Strategies](#5-scheduling-strategies)
6. [Exactly-Once Execution — The Hard Problem](#6-exactly-once-execution--the-hard-problem)
7. [Worker Design — Stateless and Idempotent](#7-worker-design--stateless-and-idempotent)
8. [Priority Queues and Fairness](#8-priority-queues-and-fairness)
9. [Task Scheduler Failure Modes](#9-task-scheduler-failure-modes)
10. [How Task Schedulers Connect to Other Building Blocks](#10-how-task-schedulers-connect-to-other-building-blocks)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What a Distributed Task Scheduler Does

A task scheduler manages the execution of work that needs to happen outside the normal request-response cycle. It answers three questions:

**When** should this work run? (immediately, at a specific time, on a recurring schedule)
**Where** should it run? (which worker, on which machine)
**What** happens if it fails? (retry, skip, alert)

In a single-server world, cron jobs handle scheduled tasks and background threads handle async work. In a distributed system, these simple solutions break down: multiple servers means multiple cron jobs running the same task simultaneously, no coordination means no guarantee that any task runs at all, and no persistence means tasks are lost when servers restart.

A distributed task scheduler solves these problems by centralizing task management — tracking what needs to run, assigning it to available workers, and ensuring it completes reliably even when individual machines fail.

---

## 2. The Three Types of Tasks

### Immediate Async Tasks
Work that should happen as soon as possible but doesn't need to block the user's request.

```
User uploads a video
API returns "Upload successful" immediately
Task: transcode video to multiple resolutions
Scheduler: assign to first available transcoding worker
```

These are essentially the same as messages in a queue — the scheduler routes them to workers. The distinction from a pure queue is that the scheduler can add priority, rate limiting, and retry policies on top of raw delivery.

### Scheduled Tasks (One-Time)
Work that should happen at a specific future time.

```
User schedules a tweet for tomorrow at 9am
Task: publish tweet
Scheduled for: 2024-01-16T09:00:00Z
Scheduler: store the task, wake up at the right time, execute
```

### Recurring Tasks (Cron Jobs)
Work that happens on a repeating schedule.

```
Daily: generate usage reports for all users
Weekly: send newsletter digest
Hourly: check for expired sessions and clean them up
Every 5 minutes: sync inventory counts from warehouse system
```

These are the distributed equivalent of cron jobs — but unlike cron, the scheduler ensures the task runs exactly once across the entire fleet, retries on failure, and logs execution history.

---

## 3. Core Components of a Task Scheduler

```
┌─────────────────────────────────────────────────────────┐
│                  Task Scheduler System                   │
│                                                          │
│  ┌──────────────┐    ┌──────────────┐                   │
│  │  Task Store  │    │   Scheduler  │                   │
│  │  (database)  │    │   (engine)   │                   │
│  │              │    │              │                   │
│  │ Pending tasks│◄───│ Polls for    │                   │
│  │ Running tasks│    │ due tasks    │                   │
│  │ Completed    │    │ Assigns to   │                   │
│  │ Failed tasks │    │ workers      │                   │
│  └──────────────┘    └──────┬───────┘                   │
│                             │                            │
└─────────────────────────────┼────────────────────────────┘
                              │ dispatch
                    ┌─────────┼──────────┐
                    │         │          │
                    ▼         ▼          ▼
               Worker 1  Worker 2   Worker 3
              (execute)  (execute)  (execute)
```

**Task Store** — a durable database holding all task definitions, their state, scheduled times, retry counts, and execution history. This is the source of truth. Even if the scheduler crashes, task definitions survive.

**Scheduler Engine** — polls the task store for tasks that are due, selects an available worker, and dispatches the task. Also handles retry logic and dead letter handling.

**Workers** — stateless processes that receive task assignments and execute them. Can be scaled horizontally. Don't need to know about the scheduler — they just pick up work and report results.

---

## 4. Task Lifecycle

Every task moves through a well-defined set of states:

```
                 ┌──────────┐
                 │ PENDING  │  Task created, waiting to be scheduled
                 └────┬─────┘
                      │ scheduler picks it up
                      ▼
                 ┌──────────┐
                 │ QUEUED   │  Assigned to a worker queue
                 └────┬─────┘
                      │ worker picks it up
                      ▼
                 ┌──────────┐
                 │ RUNNING  │  Worker is executing
                 └────┬─────┘
                      │
             ┌────────┼────────┐
             │        │        │
             ▼        ▼        ▼
        ┌─────────┐  ┌──────┐  ┌───────────┐
        │COMPLETED│  │FAILED│  │  TIMEOUT  │
        │(success)│  │(error)│  │(too slow) │
        └─────────┘  └──┬───┘  └─────┬─────┘
                        │            │
                        └─────┬──────┘
                              │ retry if attempts remaining
                              ▼
                         ┌──────────┐
                         │ PENDING  │  Back to queue for retry
                         └──────────┘
                              │ max retries exceeded
                              ▼
                         ┌──────────┐
                         │   DLQ    │  Dead letter — needs human attention
                         └──────────┘
```

The **timeout** state is important. If a worker picks up a task and disappears (crashes, gets disconnected) without completing or failing, the task would be stuck in RUNNING forever. Schedulers use a **heartbeat mechanism**: running workers must periodically signal they're still alive. If the heartbeat stops, the scheduler marks the task as timed out and re-queues it.

---

## 5. Scheduling Strategies

### Cron Expression Scheduling
The most common format for recurring tasks. A cron expression is a compact string defining when to run:

```
Format: minute hour day-of-month month day-of-week

"0 9 * * 1-5"      → 9:00am, Monday through Friday
"*/5 * * * *"      → Every 5 minutes
"0 0 1 * *"        → Midnight on the 1st of every month
"0 2 * * 0"        → 2:00am every Sunday
```

### Delay Scheduling
Run a task N seconds/minutes/hours from now.

```
send_email_in_10_minutes(user_id, email_content)
→ schedules task for: now() + 600 seconds
```

Useful for: reminder emails, follow-up notifications, delayed processing after a trigger event.

### Priority-Based Scheduling
Tasks have priorities. Higher-priority tasks are dispatched before lower-priority ones.

```
Priority 1: Critical alerts, payment retries
Priority 2: User-initiated exports and reports
Priority 3: Bulk background processing
Priority 4: Analytics aggregation, cleanup jobs
```

When workers are busy, Priority 1 tasks jump the queue.

---

## 6. Exactly-Once Execution — The Hard Problem

The hardest problem in distributed task scheduling is ensuring each task runs **exactly once** — not zero times (missed) and not twice (duplicate execution).

This is difficult because:

**Network partitions** — the scheduler assigns a task to Worker A. The network hiccups. The scheduler doesn't get confirmation. Did Worker A receive it? The scheduler re-assigns to Worker B. Now both A and B are running the same task.

**Worker crashes** — Worker A starts a task, completes the work, crashes before sending completion status. Scheduler sees task still RUNNING, times out, re-assigns to Worker B. Task runs twice.

### The Standard Solution: Idempotent Tasks + At-Least-Once Execution

Rather than solving exactly-once at the infrastructure level (expensive, complex), design tasks to be idempotent and use at-least-once execution:

```
Task: "send welcome email to user 123"

Idempotent implementation:
  IF NOT EXISTS (SELECT 1 FROM sent_emails WHERE user_id=123 AND type='welcome'):
    send_email(user_id=123, template='welcome')
    INSERT INTO sent_emails (user_id, type, sent_at) VALUES (123, 'welcome', now())

Result: task can run twice safely — second execution is a no-op
```

The same principle applies to database operations (use upsert instead of insert), payments (idempotency keys), and file operations (check if file exists before creating).

### Distributed Locking for Critical Tasks

For tasks that absolutely cannot run in parallel (like a financial reconciliation job), use a distributed lock:

```
TASK: "run monthly billing for account 456"

Before execution:
  acquired = redis.SET("lock:billing:456", worker_id, NX EX 3600)
  IF NOT acquired: exit (another worker has the lock)

Execute billing...

After execution:
  redis.DELETE("lock:billing:456")
```

Only one worker can hold the lock. If the worker crashes while holding it, the lock expires (EX 3600 = 1 hour TTL) and the task can be retried.

---

## 7. Worker Design — Stateless and Idempotent

Good workers share two properties:

**Stateless** — workers don't hold task state in memory. All state lives in the task store. If a worker crashes, another worker can pick up the task from where it was last checkpointed, not from scratch (for long tasks) or from the beginning (for short tasks).

**Idempotent** — running the same task twice produces the same result. This is the practical solution to the exactly-once problem — if duplicates are harmless, you don't need to prevent them.

```
Bad worker (stateful):
  def process_order(order_id):
    order = fetch_order(order_id)
    self.current_order = order    # stored in worker memory
    charge_card(order)
    update_inventory(order)
    self.current_order = None

If worker crashes after charge_card, current_order is lost.
On retry: new worker has no knowledge of what happened.
charge_card runs again → customer charged twice.

Good worker (stateless + idempotent):
  def process_order(order_id):
    order = fetch_order(order_id)
    
    if order.status != 'payment_pending':  # already processed
      return
    
    charge_card(order, idempotency_key=order_id)  # safe to retry
    update_inventory(order)  # idempotent upsert
    mark_order_processed(order_id)  # atomic status update
```

---

## 8. Priority Queues and Fairness

In a shared task scheduler serving multiple teams or customers, two problems emerge:

**Priority inversion** — low-priority bulk jobs fill all workers, starving high-priority interactive jobs.

**Starvation** — low-priority jobs never run because high-priority jobs always exist.

Solutions:

**Separate queues per priority tier** — dedicate some workers to high-priority queues exclusively. Low-priority jobs use remaining workers.

```
Workers: 100 total
  20 workers: Priority 1 (critical) queue only
  30 workers: Priority 1 + Priority 2 queues
  50 workers: all priority queues
```

**Aging** — gradually increase a task's priority the longer it waits. A Priority 3 task that's waited 24 hours becomes Priority 1.

**Fair queuing** — ensure each customer/team gets a proportional share of worker capacity, preventing one team's bulk jobs from starving another's.

---

## 9. Task Scheduler Failure Modes

### Scheduler as Single Point of Failure
If there's only one scheduler instance and it goes down, no new tasks are dispatched. Tasks already running continue; new scheduling stops.

**Solution:** Run multiple scheduler instances with leader election. One is active; others are standby. If the leader fails, a standby takes over automatically.

### Clock Skew
In a distributed system, different servers have slightly different clocks. A task scheduled for 09:00:00 may run at 08:59:59 on one worker and 09:00:01 on another.

**Solution:** Use a single authoritative time source (database timestamp, NTP-synchronized time). Never rely on local worker clocks for scheduling decisions.

### Task Explosion on Catch-Up
After a scheduler outage, all tasks that should have run during the downtime become due simultaneously. Processing the backlog all at once can overwhelm workers and dependencies.

**Solution:** Implement catch-up rate limiting. Process backlog tasks at a controlled rate. For recurring tasks, decide whether to run missed executions or skip them and resume normal schedule.

### Poison Tasks
A task that consistently fails (bad input, logic bug) will retry until it hits max retries and goes to the DLQ. If the failure rate is high, it consumes retry capacity that legitimate tasks need.

**Solution:** Circuit breaker per task type. If a task type fails at a high rate, pause that task type automatically and alert engineers.

---

## 10. How Task Schedulers Connect to Other Building Blocks

```
Task Store (Database) ──────────────────────────────────────────────────►
  Persistent storage for all task definitions and state.
  Source of truth — scheduler rebuilt from here after crash.

Message Queue ────────────────────────────────────────────────────────────►
  Scheduler often uses a queue as the dispatch mechanism.
  Puts task IDs on the queue; workers consume and execute.
  Queue handles delivery guarantees; scheduler handles scheduling logic.

Sequencer ────────────────────────────────────────────────────────────────►
  Task IDs need to be globally unique and often time-ordered.
  Sequencer generates IDs that are unique across all scheduler instances.

Distributed Cache (Redis) ───────────────────────────────────────────────►
  Distributed locks for preventing duplicate execution.
  Worker heartbeats stored in Redis with TTL.
  Priority queue implementation using Redis sorted sets.

Distributed Monitoring ──────────────────────────────────────────────────►
  Track: tasks created, tasks completed, tasks failed, DLQ depth.
  Alert: task latency exceeding SLA, DLQ growing, scheduler downtime.
  Dashboard: worker utilization, queue depth per priority tier.

Blob Store ──────────────────────────────────────────────────────────────►
  Large task outputs (generated reports, processed files) stored in blob store.
  Task record holds a reference key, not the content itself.
```

---

## 11. Self-Check

1. What are the three types of tasks a scheduler handles? Give a real-world example of each.
2. What is the heartbeat mechanism, and why is it necessary?
3. Why is exactly-once execution hard in a distributed system? What is the practical solution most systems use?
4. What is a distributed lock, and when would you use one instead of relying on idempotency?
5. Why must workers be stateless and idempotent? What goes wrong if they're not?
6. You're designing a task scheduler for a multi-tenant SaaS platform. One customer submits 10,000 bulk export jobs. How do you prevent their jobs from starving other customers' real-time tasks?
7. Your task scheduler goes down for 2 hours. When it comes back, 5,000 overdue tasks are due immediately. What problems can this cause, and how do you handle it?

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 11 (Kleppmann)](https://dataintensive.net) | Stream processing and batch job patterns |
| 🔧 [Celery Documentation](https://docs.celeryq.dev/) | The most widely used Python task scheduler — excellent conceptual overview |
| 📬 [ByteByteGo — Distributed Job Scheduler](https://bytebytego.com) | Visual walkthrough of task scheduler architecture |
| 📝 [Airflow Architecture Overview](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/overview.html) | How a production-grade distributed scheduler works internally |

---

*⬅️ Previous: [Processing Overview](Processing.md) &nbsp;|&nbsp; ➡️ Next: [Sequencer](sequencer.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Processing.</sub>