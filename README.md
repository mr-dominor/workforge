# workforge
# Distributed Job Queue

A distributed job processing system built from scratch to explore the fundamentals of **job queues, worker systems, reliability, concurrency, and distributed systems**.

The goal is not to build another production-ready Celery/RabbitMQ/BullMQ clone.

The goal is to understand **why these systems are designed the way they are**.

---

## Architecture

The system will evolve incrementally.

### Initial architecture

```text
                         ┌──────────────┐
                         │     API      │
                         │   Producer   │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │    Queue     │
                         └──────┬───────┘
                                │
                    ┌───────────┼───────────┐
                    │           │           │
                    ▼           ▼           ▼
                ┌───────┐   ┌───────┐   ┌───────┐
                │Worker │   │Worker │   │Worker │
                │   1   │   │   2   │   │   3   │
                └───┬───┘   └───┬───┘   └───┬───┘
                    │           │           │
                    └───────────┼───────────┘
                                ▼
                         ┌──────────────┐
                         │    Results   │
                         └──────────────┘
```

The architecture will become progressively more sophisticated as new failure modes and scalability problems are discovered.

---

## Why build this?

Modern job-processing systems hide a surprising amount of distributed-systems complexity behind simple APIs.

For example:

```text
enqueue(job)
```

looks simple.

But what happens when:

* a worker crashes while processing the job?
* the queue process crashes?
* a job takes 30 minutes?
* the worker finishes but crashes before acknowledging?
* the same job is delivered twice?
* 100 workers compete for the same queue?
* jobs arrive faster than workers can process them?
* one job keeps failing?
* a high-priority job is stuck behind thousands of low-priority jobs?
* the system needs to recover after a network partition?

This project explores those problems by implementing the underlying mechanisms rather than immediately relying on an existing queue system.

---

# Project Goals

The system will eventually support:

* [ ] Producers
* [ ] Consumers
* [ ] Worker pools
* [ ] Job persistence
* [ ] Acknowledgements
* [ ] Job states
* [ ] Retries
* [ ] Exponential backoff
* [ ] Dead-letter queues
* [ ] Job priorities
* [ ] Scheduled jobs
* [ ] Idempotency
* [ ] Worker failure recovery
* [ ] Graceful shutdown
* [ ] Horizontal scaling
* [ ] Observability
* [ ] Load testing
* [ ] Failure injection

---

# Development Roadmap

The project will be developed incrementally.

## Phase 0 — In-memory queue

Start with the simplest possible implementation.

```text
Producer
   │
   ▼
┌─────────────┐
│ In-Memory   │
│ Queue       │
└──────┬──────┘
       │
       ▼
    Worker
```

Implement:

* enqueue
* dequeue
* workers
* basic concurrency

### Question

How many jobs can the system process per second?

---

## Phase 1 — Worker Pool

Introduce multiple workers.

```text
             ┌── Worker 1
             │
Queue ───────┼── Worker 2
             │
             └── Worker 3
```

Experiment with:

```text
1 worker
5 workers
10 workers
50 workers
100 workers
```

Measure:

* throughput
* queue depth
* latency
* CPU usage
* memory usage

---

## Phase 2 — Job States

Introduce explicit job lifecycle states.

```text
PENDING
   │
   ▼
RUNNING
   │
   ├──────────► FAILED
   │
   ▼
COMPLETED
```

The system should be able to answer:

> What happened to job `123`?

---

## Phase 3 — Acknowledgements

Introduce explicit acknowledgement.

```text
Queue
  │
  │ reserve
  ▼
Worker
  │
  │ process
  ▼
Worker
  │
  │ ACK
  ▼
Queue
```

This introduces an important distinction between:

```text
job received
```

and

```text
job successfully completed
```

---

## Phase 4 — Worker Failure Recovery

Intentionally kill workers during job execution.

```text
Queue
  │
  ▼
Worker
  │
  │ processing
  │
  X 💥
```

The system must determine what happened to the job.

This phase introduces concepts such as:

* leases
* visibility timeouts
* heartbeats
* stale workers
* job reclamation

---

## Phase 5 — Retries

Failed jobs can be retried.

```text
Job
 │
 ▼
Attempt 1 ──X
 │
 ▼
Attempt 2 ──X
 │
 ▼
Attempt 3 ──✓
```

Introduce:

* maximum attempts
* retry policies
* retryable vs permanent failures

---

## Phase 6 — Exponential Backoff

Avoid immediately retrying failing jobs.

Example:

```text
Attempt 1
   │
   └── wait 1s

Attempt 2
   │
   └── wait 2s

Attempt 3
   │
   └── wait 4s

Attempt 4
   │
   └── wait 8s
```

Experiment with different backoff strategies and observe their effects on queue pressure.

---

## Phase 7 — Dead-Letter Queue

Jobs that repeatedly fail should eventually stop blocking normal processing.

```text
              ┌──────────────┐
              │    Queue     │
              └──────┬───────┘
                     │
                     ▼
                  Worker
                     │
                failures
                     │
                     ▼
              ┌──────────────┐
              │     DLQ      │
              └──────────────┘
```

The DLQ will allow failed jobs to be inspected and potentially replayed.

---

## Phase 8 — Priorities

Introduce multiple priority levels.

```text
HIGH
MEDIUM
LOW
```

Investigate:

* priority inversion
* starvation
* fairness
* scheduling strategies

---

## Phase 9 — Scheduling

Support jobs that should execute in the future.

```text
schedule(job, 10:30)

        │
        │ wait
        ▼

    10:30 AM

        │
        ▼

      Queue
        │
        ▼
      Worker
```

---

## Phase 10 — Idempotency

Distributed systems can deliver the same job more than once.

Therefore:

```text
Job A
 │
 ├── Worker 1
 │
 └── Worker 2
```

must not necessarily produce two side effects.

Explore:

* idempotency keys
* deduplication
* exactly-once vs at-least-once processing
* transactional boundaries

---

## Phase 11 — Persistence

Move beyond an in-memory queue.

The system should survive:

```text
Queue crash
Worker crash
Machine restart
```

The exact persistence mechanism will be introduced after the failure modes of the in-memory implementation are understood.

---

## Phase 12 — Horizontal Scaling

Move from:

```text
             Queue
               │
             Worker
```

to:

```text
                  Queue
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
    Worker 1     Worker 2     Worker 3
       │            │            │
       ▼            ▼            ▼
    Worker 4     Worker 5     Worker 6
```

Measure how the system behaves as workers increase.

---

# Experiments

A major part of this project is experimentation.

Rather than assuming an architecture scales, the system will be measured under increasing load.

## Worker scaling

```text
1 worker
10 workers
25 workers
50 workers
100 workers
```

Measure:

| Metric        | Description                          |
| ------------- | ------------------------------------ |
| Throughput    | Jobs processed per second            |
| p50 latency   | Median job latency                   |
| p95 latency   | Tail latency                         |
| p99 latency   | Extreme tail latency                 |
| Failure rate  | Percentage of failed jobs            |
| Queue depth   | Number of waiting jobs               |
| Recovery time | Time required to recover failed work |
| CPU           | CPU utilization                      |
| Memory        | Memory utilization                   |

---

# Failure Experiments

The system will intentionally be broken.

Examples:

### Worker crash

```text
Worker
  │
  │ processing job
  X
 💥
```

Questions:

* Is the job lost?
* When is it detected?
* Who recovers it?
* Can another worker process it?

---

### Queue crash

```text
Producer ──► Queue 💥
                  │
                  ▼
              restart
```

Questions:

* Which jobs survive?
* Which jobs are lost?
* Can producers safely retry?

---

### Slow worker

```text
Worker 1 ────────────────►
Worker 2 ──►
Worker 3 ──►
```

Questions:

* Does one slow worker reduce overall throughput?
* Should jobs have leases?
* How should work be redistributed?

---

### Job failure storm

```text
1000 jobs
   │
   ▼
All failing
   │
   ▼
Retries
   │
   ▼
More load
```

Questions:

* Can retries overload the system?
* Does exponential backoff help?
* When should jobs enter the DLQ?

---

# Performance

The goal is not simply:

> "Make it fast."

The goal is to understand **why performance changes**.

For each experiment, we will investigate:

```text
Load
  ↓
Queue depth
  ↓
Worker utilization
  ↓
Latency
  ↓
Throughput
  ↓
System bottleneck
```

The project will document where adding more workers stops improving throughput and what resource becomes the limiting factor.

---

# Design Principles

### 1. Build before abstracting

Don't start with a large framework.

First understand the primitive.

### 2. Find the failure before solving it

For example:

```text
Worker crashes
      ↓
Job disappears
      ↓
Why?
      ↓
Introduce acknowledgement / lease
```

The implementation should emerge from observed problems.

### 3. Measure everything important

Performance claims should come from experiments rather than assumptions.

### 4. Prefer explicit guarantees

Every feature should answer:

> What guarantee does this provide?

Examples:

```text
ACK
→ completed work can be distinguished from reserved work

Persistence
→ state can survive process failure

Retry
→ transient failures don't necessarily lose jobs

Idempotency
→ duplicate delivery doesn't necessarily duplicate side effects
```

---

# Eventually: Compare Against Existing Systems

Once the core implementation works, it can be compared conceptually and experimentally with established technologies such as:

* RabbitMQ
* Redis-based queues
* BullMQ
* Kafka
* Celery
* Sidekiq

The goal isn't to replace them.

The goal is to understand the engineering decisions behind them.

---

# Tech Stack

The initial implementation is intentionally lightweight.

Planned technologies may include:

```text
Go
Docker
PostgreSQL
Redis
Prometheus
Grafana
k6
```

Infrastructure will be introduced only when the project reaches the point where it is useful for demonstrating a specific systems concept.

Everything can be run locally for development and experimentation.

---

# Running the Project

> This section will be updated as the implementation develops.

Eventually the goal will be to make the entire system reproducible locally:

```bash
git clone <repository>

cd distributed-job-queue

docker compose up
```

and then run producers and workers against the local queue.

---

# What I Want to Learn

This project is primarily an exploration of:

* concurrency
* distributed systems
* failure handling
* reliability
* queues
* scheduling
* backpressure
* consistency
* idempotency
* observability
* performance engineering
* horizontal scaling

The final implementation matters.

But the **experiments, failures, measurements, and lessons learned** matter just as much.

---

## Status

🚧 **Work in progress**

Currently starting with the fundamental queue and worker model.

The architecture will intentionally evolve as new failure modes are discovered.

---

## License

MIT
