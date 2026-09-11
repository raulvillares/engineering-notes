---
title: Multi-Tenant Workload Isolation
tags:
  - distributed-systems
  - queues
  - multi-tenancy
  - backpressure
  - reliability
---

In a multi-tenant system, one customer can consume enough shared capacity to degrade the experience of everyone else.

This is commonly known as the **noisy neighbour problem**.

A useful way to frame it is:

> A tenant generates a disproportionate amount of work and competes with other tenants for shared resources.

One way to mitigate this is to move that workload to dedicated queues and worker pools.

But that immediately raises a more important question:

> Which resources are actually isolated, and which ones are still shared?

That distinction matters because moving work to a different queue does not automatically isolate the whole system.

## The problem that made this concrete for me

I worked on a payments system where some customers could create very large batches of payments.

Normal payment traffic and batch traffic followed much of the same processing lifecycle:

```text
payment created
      │
      ▼
validation
      │
      ▼
funds / business checks
      │
      ▼
state transitions
      │
      ▼
transfer creation
      │
      ▼
send to external system
```

A large batch could therefore consume a significant amount of processing capacity.

The product requirement was not necessarily to process batch traffic as quickly as possible.

It was closer to:

> Batch traffic may take longer, but it should not significantly degrade normal traffic from other customers.

This makes the trade-off explicit:

```text
higher latency for batch traffic
                │
                ▼
better isolation and fairness
```

## Dedicated queues and worker pools

A straightforward first step is to route batch traffic through a dedicated queue:

```text
normal traffic ─────► normal queue ─────► normal workers

batch traffic  ─────► batch queue  ─────► limited batch workers
```

The batch worker pool can deliberately have limited concurrency.

This gives us a separate capacity envelope for that workload.

If a large batch arrives:

```text
batch queue
████████████████████████████████
              │
              ▼
         few workers
              │
              ▼
       controlled output
```

The backlog grows, but normal workers are no longer directly consumed by the batch.

This is **workload isolation**.

However, it is only isolation of the resources behind that queue.

## Isolation has a boundary

One of the most useful questions I learned to ask is:

> Where exactly does the isolated workload begin and end?

For example, the outbound lifecycle may be isolated:

```text
validation
    │
    ▼
business checks
    │
    ▼
state transitions
    │
    ▼
external transfer
```

while other work remains shared:

```text
HTTP ingestion
database writes
customer notifications
analytics
provider responses
```

This means that the system may look isolated when viewed from the queue layer while still sharing important bottlenecks elsewhere.

So I prefer to think about isolation as a property of a **resource or workload**, not of an entire tenant.

## A resource-by-resource view

A simplified architecture might look like this:

```text
                         ┌──────────────┐
Client ────────────────► │ HTTP ingress │
                         └──────┬───────┘
                                │
                       ┌────────▼────────┐
                       │   PostgreSQL    │
                       └────────┬────────┘
                                │
              ┌─────────────────┴────────────────┐
              │                                  │
              ▼                                  ▼
      shared side effects                 batch processing
              │                                  │
              ▼                                  ▼
       shared queues                      dedicated queue
                                                   │
                                                   ▼
                                            limited workers
                                                   │
                                                   ▼
                                           downstream system
```

Now the isolation question becomes much more precise.

| Resource | Isolated? | Possible remaining risk |
| --- | --- | --- |
| Batch processing workers | Yes | Batch latency |
| Batch queue | Yes | Backlog growth |
| Downstream sending capacity | Largely | Provider limits |
| HTTP workers | No | Request bursts |
| Database | No | CPU, I/O, locks, connections |
| Shared notification queues | No | Cross-tenant delays |
| External provider | Partially | Downstream saturation |

The key insight is:

> A dedicated queue isolates only the resources that are actually downstream of that routing decision.

## Queue leakage

Isolation can also fail accidentally.

Suppose the initial job is correctly routed:

```text
batch workload
     │
     ▼
dedicated queue
```

but a retry later goes to:

```text
shared queue
```

The workload has escaped the isolation boundary.

I think of this as **queue leakage**.

It can happen through:

- retries;
- delayed re-enqueues;
- contention handling;
- callbacks;
- secondary jobs;
- queue selection logic that does not preserve tenant or workload classification.

This means isolation has to be checked across the entire lifecycle, not just at the first enqueue.

A useful invariant is:

> Once work belongs to an isolated workload class, retries and re-enqueues should preserve that classification unless there is a deliberate reason not to.

## Ingestion rate vs processing rate

Queues make the difference between **ingestion rate** and **processing rate** especially visible.

Suppose a customer creates:

```text
10,000 jobs
```

very quickly.

But the isolated worker pool can process only:

```text
4 jobs / second
```

Then:

```text
10,000 / 4 = 2,500 seconds
               ≈ 42 minutes
```

That may be completely acceptable.

The queue absorbs the difference:

```text
ingestion rate  >>>  processing rate
```

But this reveals an important limitation.

The queue only protects resources used **after enqueueing**.

If accepting those 10,000 items already requires:

- HTTP capacity;
- authentication;
- database connections;
- database writes;
- locks;
- job creation;

then those resources can still experience the full ingestion burst.

## Buffering is not backpressure

A queue provides **buffering**.

It allows producers and consumers to operate at different rates for a while.

But an arbitrarily large queue does not automatically provide **backpressure**.

If the system continues accepting:

```text
10k
50k
100k
500k
```

items while consumers remain slower than producers, the backlog simply moves the problem elsewhere.

Backpressure answers a different question:

> What should happen when incoming work exceeds processing capacity for too long?

Possible mechanisms include:

- rate limiting;
- admission control;
- quotas;
- concurrency limits;
- backlog limits;
- temporary rejection;
- dynamic ingestion reduction.

So I keep these concepts separate:

```text
queue
  │
  └── buffers excess work

backpressure
  │
  └── prevents unlimited excess work from entering
```

## Rate limiting is not the same as fairness

Rate limiting can protect shared infrastructure.

For example:

```text
tenant A → 100 requests/min
tenant B → 100 requests/min
```

But fairness is a broader concept.

Questions include:

- Should all tenants get the same limit?
- Should limits depend on customer tier?
- How large can bursts be?
- Which resources are protected by the limiter?
- Does rejected traffic occur before or after expensive database work?
- Can one tenant still monopolize another shared resource?

A rate limiter controls request admission.

It does not automatically guarantee fair resource allocation throughout the whole system.

## Shared side effects can reintroduce contention

Another subtle case is side effects generated before or after isolated processing.

Imagine every accepted payment generates an event or notification:

```text
10,000 payments
       │
       ▼
10,000 notification jobs
```

If those jobs use a shared queue, a batch tenant can still create substantial cross-tenant impact.

For example:

```text
batch notifications
████████████████████████

important notification from another tenant
                         ▲
                         │
                         waits behind them
```

The payment processing itself may be isolated while the externally visible notification is delayed.

This introduces an important product distinction.

## Internal latency vs perceived latency

Suppose the system marks an operation complete at:

```text
10:00:00
```

but the customer learns about it through a callback delivered at:

```text
10:08:00
```

Internally:

```text
processing latency = acceptable
```

From the customer's perspective:

```text
effective latency = 8 minutes
```

This means architecture metrics should often follow the user-visible lifecycle rather than stopping at an internal state transition.

A technically isolated processing queue can still produce a poor customer experience if another shared stage becomes congested.

## Priority scheduling

Not every shared queue necessarily needs full physical isolation.

Another tool is priority scheduling.

Conceptually:

```text
high priority
    critical completion events

normal priority
    regular work

lower priority
    high-volume non-critical work
```

This allows different service classes to share infrastructure.

But priorities introduce their own questions:

- Can high-priority traffic starve lower-priority traffic?
- Should priority depend on event type, tenant or both?
- How does sustained high-priority load behave?
- Is physical isolation simpler and safer?

Priority and isolation solve related but different problems.

## Limited worker pools as pacing

A small worker pool can also protect downstream systems.

Suppose a huge internal backlog exists:

```text
████████████████████████████████
```

but only a few workers can send work downstream:

```text
large backlog
     │
     ▼
 2 workers
     │
     ▼
█  █  █  █  █  █  █
     │
     ▼
external system
```

The workers act as a **pacing mechanism**.

The upstream workload may arrive as a burst while the downstream dependency sees a controlled flow.

This can protect:

- external APIs;
- databases;
- internal services;
- rate-limited providers.

But increasing the worker count changes that protection.

## Capacity planning moves the bottleneck

Suppose we increase:

```text
2 workers → 20 workers
```

Processing throughput may increase.

But the extra concurrency also increases pressure on other resources:

```text
workers
   │
   ├──► database
   ├──► business-rule service
   ├──► downstream API
   └──► shared queues
```

The bottleneck may simply move.

This is why worker count is not just an application tuning parameter.

It is part of system capacity planning.

The useful question is:

> If I increase throughput here, which resource becomes the next constraint?

## Fairness

Throughput is not the only objective in a multi-tenant system.

We also need **fairness**:

> How do we prevent one tenant from consuming a disproportionate share of shared capacity?

Possible strategies include:

### Dedicated capacity

```text
batch workload → dedicated workers
normal traffic → normal workers
```

### Per-tenant rate limits

```text
tenant A → X requests/min
tenant B → Y requests/min
```

### Concurrency limits

```text
maximum N concurrent operations per tenant
```

### Priority scheduling

Some work is processed before other work.

### Weighted fairness

Different tenants or service classes receive different shares of available capacity.

No mechanism is universally correct.

Each expresses a product decision about how capacity should be shared.

## Isolation vs utilisation

Dedicated capacity improves isolation but can reduce resource utilisation.

Shared workers:

```text
all tenants
    │
    ▼
shared capacity
```

can use spare capacity efficiently.

Dedicated workers:

```text
workload A → workers A
workload B → workers B
```

provide stronger isolation, but can produce:

```text
workers A: saturated
workers B: idle
```

even though unused capacity exists elsewhere.

So there is a classic trade-off:

> Stronger isolation usually reduces the efficiency of shared resource utilisation.

The right design depends on how expensive cross-tenant interference is.

## Batch APIs

Another possible approach is to change the ingestion model itself.

Instead of:

```text
POST /payments
POST /payments
POST /payments
...
```

a system might expose:

```text
POST /payment_batches
```

This can reduce:

- HTTP overhead;
- repeated authentication;
- round trips;
- pressure on web workers.

But it creates a new domain model.

Questions immediately appear:

- What happens if some items are invalid?
- Is idempotency defined per batch, per item, or both?
- Does the endpoint respond synchronously or with `202 Accepted`?
- How does the client observe progress?
- How are retries handled?
- What does partial completion mean?

A batch API may reduce ingestion cost, but it is not merely an optimization.

It introduces new semantics that the system must maintain.

## Observability

Isolation is not successful because a dedicated queue exists.

It is successful if other tenants remain protected.

Useful metrics include:

### Queue

- queue depth;
- oldest job age;
- enqueue rate;
- processing rate;
- retry rate.

### Workload

- end-to-end processing latency;
- batch completion time;
- latency by tenant.

### API

- request latency;
- request rate by tenant;
- rejected requests;
- connection pool usage;
- database saturation.

### Side effects

- notification queue depth;
- delivery latency;
- latency by tenant;
- latency by event type.

The most important validation metric may be:

> Does normal-tenant latency increase when a large batch arrives?

That directly measures whether the isolation is achieving its purpose.

## Different workloads can have different SLOs

A normal interactive operation and a batch operation do not necessarily need the same latency target.

For example:

```text
interactive payment
p95 < X seconds
```

while:

```text
batch workload
completion < Y minutes
```

This makes the architectural trade-off explicit:

> Batch traffic accepts greater latency so that interactive traffic can preserve its service objective.

The infrastructure then reflects a product decision.

## Mental model

The model I keep is:

```text
                  LARGE TENANT BURST
                          │
                          ▼
                   shared ingress
                  ┌──────────────┐
                  │ HTTP + DB    │
                  └──────┬───────┘
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
       shared workloads        isolated workload
                                     │
                                     ▼
                               dedicated queue
                                     │
                                     ▼
                               limited workers
                                     │
                                     ▼
                                downstream
```

For each stage I ask:

1. What resource does this work consume?
2. Is that resource shared?
3. Can one tenant monopolize it?
4. What is its capacity?
5. What happens when capacity is exceeded?
6. Do we need isolation, prioritisation, backpressure, rate limiting or simply better observability?

That is more useful than asking only:

> Which queue should this job use?

## Related concepts

- [[Noisy Neighbour]]
- [[Workload Isolation]]
- [[Backpressure]]
- [[Buffering]]
- [[Rate Limiting]]
- [[Admission Control]]
- [[Fairness]]
- [[Priority Scheduling]]
- [[Pacing]]
- [[Capacity Planning]]
- [[Multi-Tenancy]]
- [[Resource Contention]]
- [[Queue Depth]]
- [[Queue Lag]]
- [[Service Level Objectives]]

---

This note came from analysing a payments workload where large customer batches were moved to dedicated processing capacity. The interesting part was not creating another queue, but determining which resources were actually isolated, which remained shared, and where the next cross-tenant bottleneck could appear.