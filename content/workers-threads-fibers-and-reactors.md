---
title: Workers, Threads, Fibers, and Reactors
tags:
  - system-design
  - concurrency
  - background-jobs
  - ruby
  - rails
---

# Workers, Threads, Fibers, and Reactors

Background job systems often mix concepts from several layers:

- infrastructure,
- operating system processes,
- job runners,
- concurrency primitives,
- database connection pools.

That can make statements such as “move this job from threaded workers to the fiber reactor” sound more complicated than they are.

This note builds a practical mental model for reasoning about those systems.

---

## 1. Start with the execution hierarchy

A useful simplified hierarchy is:

```text
Kubernetes cluster
    │
    └── Pod
         │
         └── Ruby process / job worker
              │
              ├── Thread(s)
              │     │
              │     └── Fiber(s), when using a reactor
              │
              └── Database connection pool
```

These concepts belong to different layers.

### Pod

A **pod** is a Kubernetes deployment unit.

It contains one or more containers and therefore one or more application processes.

A pod owns resources such as:

- CPU,
- memory,
- network namespace,
- process lifecycle.

A pod is not a job worker, although a deployment may use one worker process per pod.

### Worker

A **worker** is an application process responsible for executing background jobs.

Conceptually:

```text
job queue
   │
   ├── Job A
   ├── Job B
   └── Job C
        │
        ▼
      worker
```

The worker decides how much concurrency it provides: multiple threads, fibers, processes, or some combination.

---

## 2. Thread-based workers

A process can contain multiple threads.

```text
Worker process
│
├── Thread 1 → Job A
├── Thread 2 → Job B
├── Thread 3 → Job C
└── Thread 4 → Job D
```

If all threads are busy, new jobs have to wait.

```text
Thread 1   busy
Thread 2   busy
Thread 3   busy
Thread 4   busy

Job E      waiting
```

This creates an important distinction:

> Queue latency is not the same as execution time.

A job may only need 200 ms of actual work but still start several seconds later because no execution slot is available.

Thread pools therefore introduce a bounded concurrency limit:

```text
maximum concurrent jobs ≈ available worker threads
```

This is simple and predictable, but threads are relatively expensive resources.

---

## 3. Fibers

A **fiber** is a lightweight cooperative execution unit.

The key idea is not that fibers are "smaller threads".

The important difference is that a fiber can yield while waiting for I/O.

For example:

```text
Fiber A
  │
  ├── query PostgreSQL
  │
  └── waits for response
          │
          ▼
      Fiber B runs
```

Then:

```text
Fiber B
  │
  ├── HTTP request
  │
  └── waits for network
          │
          ▼
      Fiber C runs
```

One thread can therefore keep many operations in progress when those operations spend much of their lifetime waiting.

---

## 4. The reactor

The **reactor** coordinates those fibers.

A simplified model:

```text
             Reactor thread
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
    Fiber A     Fiber B     Fiber C
```

Execution may look like:

```text
Fiber A → work
Fiber A → wait for PostgreSQL

Fiber B → work
Fiber B → wait for HTTP

Fiber C → work
Fiber C → finish

PostgreSQL responds

Fiber A → continue
```

The reactor is not necessarily running all fibers on CPU at the same time.

Instead, it keeps the thread productive by switching to another fiber when one becomes blocked on I/O.

This is **concurrency**, not necessarily CPU parallelism.

---

## 5. Threads vs fibers

### Threads

```text
Process
│
├── Thread 1 → Job A
├── Thread 2 → Job B
├── Thread 3 → Job C
└── Thread 4 → Job D
```

Four jobs in progress generally require four execution threads.

### Fibers

```text
Process
│
└── Reactor thread
      │
      ├── Fiber A → Job A
      ├── Fiber B → Job B
      ├── Fiber C → Job C
      └── Fiber D → Job D
```

A small number of threads can keep many jobs in progress when those jobs mostly wait on I/O.

That makes fibers especially attractive for:

- webhooks,
- HTTP integrations,
- database-heavy jobs,
- network-bound work,
- other I/O-heavy workloads.

---

## 6. Good and bad workloads for fibers

A good fiber workload looks like this:

```text
small amount of Ruby work
↓
wait for PostgreSQL
↓
small amount of Ruby work
↓
wait for HTTP
↓
small amount of Ruby work
```

A bad one looks like this:

```text
CPU
CPU
CPU
CPU
CPU
CPU
```

For example:

```ruby
1_000_000.times do
  expensive_calculation
end
```

If one fiber monopolizes the reactor thread:

```text
Fiber A → CPU CPU CPU CPU CPU

Fiber B → waiting to run
Fiber C → waiting to run
Fiber D → waiting to run
```

The entire reactor suffers.

### Rule of thumb

```text
Lots of I/O + little CPU  → good fiber candidate
Lots of CPU               → poor fiber candidate
```

---

## 7. Database pools become critical

Fibers make it cheap to have many jobs in progress.

Database connections do not become equally cheap.

Imagine:

```text
50 fibers
5 PostgreSQL connections
```

This can work well if fibers use connections briefly:

```text
acquire connection
↓
query
↓
release connection
```

The same small pool can serve many fibers over time.

The problem starts when fibers retain connections.

---

## 8. Long transactions and connection starvation

A transaction normally keeps a database connection checked out for its lifetime.

This pattern is dangerous in a fiber-based worker:

```text
acquire connection
↓
BEGIN
↓
query
↓
Ruby work
↓
wait
↓
another query
↓
more work
↓
COMMIT
↓
release connection
```

With a small pool:

```text
DB pool: 5 connections

Fiber A → connection 1
Fiber B → connection 2
Fiber C → connection 3
Fiber D → connection 4
Fiber E → connection 5

Fiber F → blocked
Fiber G → blocked
...
```

The reactor may theoretically support high concurrency, but the real bottleneck becomes the database pool.

### Better shape

Move work that does not need transaction semantics outside the transaction:

```text
queries
↓
checks
↓
prepare work
↓
BEGIN
↓
atomic operations only
↓
COMMIT
```

This reduces how long each fiber owns a connection.

---

## 9. Preparing a job for a reactor

Moving a job from threaded execution to a fiber reactor is not just a queue configuration change.

The job itself should be reviewed.

Useful questions include:

- Does it spend most of its time waiting on I/O?
- Does it perform expensive Ruby work?
- Does it repeatedly instantiate objects that could be reused?
- Does it perform unnecessary queries?
- Does it keep database connections checked out between queries?
- Are transactions larger than the atomic operation actually requires?
- Can downstream systems handle higher concurrency?

Typical improvements include:

### Reduce repeated computation

Avoid recomputing static information on every execution.

```text
Before:
job → build metadata → query → calculate → dispatch

After:
process startup → build metadata once

job → lookup → dispatch
```

### Reduce unnecessary queries

Replace database reads with in-memory lookups when the data is immutable or process-static.

### Ask existence questions as existence questions

If the code only needs:

```text
Does this record exist?
```

prefer an existence query over loading a complete record.

### Minimize transaction scope

Do not keep lookups, object construction, or unrelated work inside a transaction merely because the final write needs atomicity.

---

## 10. Why webhooks are a natural example

Webhook workloads are commonly I/O-bound:

```text
database
↓
enqueue / persistence
↓
network
↓
remote API
↓
database
```

Most of their lifetime may be spent waiting rather than computing.

That makes them strong candidates for cooperative concurrency.

The goal is not necessarily:

> make one webhook execute faster

but rather:

> keep many webhooks in progress efficiently without requiring one operating-system thread per job

---

## 11. Concurrency vs parallelism

These concepts are related but different.

### Concurrency

Multiple tasks are making progress during the same period.

```text
A → waiting for DB
B → running
C → waiting for HTTP
```

### Parallelism

Multiple tasks are executing CPU instructions at the same instant.

Fibers mainly improve **concurrency for I/O-bound workloads**.

They do not automatically provide more CPU parallelism.

---

## 12. Where the bottleneck moves

Changing the concurrency model changes which resources become scarce.

### Thread-based worker

A common bottleneck is:

```text
available worker threads
```

### Fiber-based reactor

The important constraints become more likely to be:

```text
reactor CPU time
database connections
non-cooperative I/O
downstream capacity
rate limits
```

This is why a job can be perfectly acceptable in a threaded worker but problematic inside a reactor.

---

## 13. Failure modes

### CPU-heavy fiber

One job monopolizes the reactor thread and delays unrelated jobs.

### Connection starvation

Too many fibers hold PostgreSQL connections at once.

### Oversized transactions

Connections remain checked out while code performs unrelated work.

### Non-cooperative libraries

An operation blocks the underlying thread instead of yielding to the reactor.

### Downstream overload

Increasing local concurrency can overload:

- databases,
- third-party APIs,
- webhook consumers,
- rate-limited services.

Concurrency must therefore be designed end to end, not only inside the worker.

---

## 14. Mental model

A useful shorthand is:

```text
Solid Queue
= stores and distributes jobs

Worker
= application process executing jobs

Pod
= Kubernetes deployment unit containing the process

Thread
= relatively expensive execution resource inside a process

Fiber
= lightweight cooperative task sharing a thread

Reactor
= loop coordinating fibers around I/O waits

DB pool
= limited set of database connections available to the process
```

And:

```text
Threads
→ concurrency is often limited by thread count

Fibers
→ concurrency is cheap,
  but CPU time and external resources become more important
```

---

## 15. Design questions

When evaluating a job for fiber-based execution:

1. Is the workload mostly I/O-bound?
2. How much uninterrupted Ruby/CPU work does one execution perform?
3. How long does it hold a database connection?
4. Are transactions broader than necessary?
5. How many fibers can compete for the same DB pool?
6. Does every I/O library cooperate with the reactor?
7. Which downstream service becomes the next bottleneck?
8. What metrics will confirm the change worked?

---

## Reusable principles

- Queue wait time and job execution time are different metrics.
- A pod and a worker belong to different abstraction layers.
- Fibers improve I/O concurrency, not CPU parallelism.
- High fiber concurrency makes connection discipline more important.
- Transaction scope is also a concurrency design decision.
- Optimizing a job for fibers often means reducing Ruby work, queries, and connection ownership time.
- Increasing concurrency locally can simply move the bottleneck downstream.
- The execution model should match the workload: I/O-bound and CPU-bound jobs have different needs.

---

## Related concepts

- [[Background Jobs]]
- [[Concurrency Control]]
- [[Database Connection Pools]]
- [[Database Transactions]]
- [[I-O Bound vs CPU Bound Workloads]]
- [[Kubernetes Pods]]
- [[Rate Limiting]]
- [[Webhooks]]
