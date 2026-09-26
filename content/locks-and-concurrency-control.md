---
title: Locks and Concurrency Control
tags:
  - system-design
  - databases
  - concurrency
  - postgresql
  - background-jobs
---

Concurrency bugs often appear when individually valid operations overlap in a way that breaks a system invariant.

A common shape is **check-then-act**:

```text
Worker A                   Worker B

reads state = pending      reads state = pending
checks transition          checks transition
writes processing          writes processing
```

Or, for money movement:

```text
Available balance: 100

Worker A                   Worker B

reads 100                  reads 100
wants to debit 80          wants to debit 80
decides it is allowed      decides it is allowed
creates debit              creates debit
```

The core design question is not simply:

> Which lock should we use?

It is:

> **What resource are we protecting, which actors can access it, and where should the serialization guarantee live?**

Different concurrency-control mechanisms answer that question at different layers.

## Pessimistic row locking

A database row lock serializes access to a specific record.

In PostgreSQL:

```sql
SELECT ...
FOR UPDATE;
```

In Rails:

```ruby
record.with_lock do
  # validate current state
  # mutate record
end
```

A typical use case is a state transition:

```text
pending -> processing
```

With a row lock:

```text
Worker A
  acquires row lock
  sees pending
  changes to processing
  commits

Worker B
  waits
  acquires row lock
  reloads the row
  sees processing
  rejects the transition
```

This works well when the protected resource maps naturally to **one database row**.

### Good fit

- State-machine transitions
- Counters
- Reservations represented by one record
- Read-modify-write operations on one entity

### Trade-offs

- The lock is held for the duration of the transaction.
- Contending transactions wait.
- Locking multiple resources can introduce deadlocks.
- The real business resource may span more than one row.

## Optimistic locking

Optimistic locking allows concurrent work and detects a conflict only when writing.

A common implementation uses a version column such as:

```text
lock_version
```

Conceptually:

```text
Worker A reads version 5
Worker B reads version 5

A writes WHERE version = 5
  -> succeeds, version becomes 6

B writes WHERE version = 5
  -> no longer matches
  -> conflict
```

The distinction is:

```text
Pessimistic locking:
  wait before doing conflicting work

Optimistic locking:
  allow the work, detect the race when committing it
```

Optimistic locking is attractive when contention is expected to be rare and retrying or rejecting a conflicting operation is cheap.

## Advisory locks

A PostgreSQL advisory lock protects a **logical resource defined by the application**, rather than a row.

For example:

```text
ledger_account_balance:123
```

The protected resource is not necessarily the `ledger_accounts` row itself. It may be the invariant:

> Only one flow at a time may calculate and consume the spendable balance of ledger account 123.

That flow might involve several reads and writes:

```text
advisory lock: ledger account 123
│
├── read confirmed balance
├── read pending debits
├── calculate available balance
├── validate debit
└── create debit / reservation
```

This is useful when the invariant spans multiple rows or tables.

### Advisory locks are cooperative

The database does not automatically force every writer to acquire the advisory lock.

The guarantee exists only if **every relevant code path follows the same locking protocol**.

This is one of the most important trade-offs: an advisory lock can express a business resource very well, but correctness depends on all participants respecting it.

## Transaction-scoped vs session-scoped locks

Where the wait happens matters.

A transaction-scoped pattern may look like:

```text
BEGIN
  wait for lock
  wait...
  do work
COMMIT
```

The transaction stays open while waiting.

An alternative is:

```text
wait for session-scoped advisory lock

lock acquired
↓
BEGIN
  read
  validate
  write
COMMIT
↓
release advisory lock
```

This avoids keeping an open database transaction while another worker is holding the resource.

The general principle is:

> **Acquire the coordination primitive at the narrowest useful scope, and keep database transactions as short as correctness allows.**

## Queue concurrency as serialization

A background-job system can also provide serialization.

For example, a queue may support:

```text
concurrency = 1
key = ledger_account_id
```

Conceptually:

```text
ledger account 123

Job A ───────────── running

Job B               blocked
Job C               blocked

A finishes
          ↓

Job B ───────────── running
```

This behaves like a distributed semaphore with one permit per key.

It can sometimes replace an application-level advisory lock.

## Queue concurrency vs advisory locking

The important difference is **where the guarantee lives**.

### Advisory lock

```text
HTTP request ─┐
Job A ────────┤
Job B ────────┼── logical resource lock
Rake task ────┤
other worker ─┘
```

The protection is attached to the **resource**.

### Queue concurrency

```text
Job A ─┐
Job B ─┼── queue concurrency key
Job C ─┘

HTTP request ─── not participating
Rake task ────── not participating
```

The protection is attached to the **execution mechanism**.

A queue concurrency limit can replace an advisory lock only if all relevant operations are guaranteed to go through that queue and use the same concurrency key and group.

That changes the architectural assumption from:

> Every mutation of this resource must acquire its lock.

to:

> Every mutation of this resource must enter through this execution path.

Both can be valid. The second is easier to bypass accidentally when a new entry point is added later.

## Semaphore-style concurrency limits

A lock is effectively a semaphore with capacity one:

```text
capacity = 1
```

A general concurrency limit allows:

```text
capacity = N
```

For example:

```text
company A -> max 5 concurrent webhook deliveries
```

This does not necessarily protect a data invariant. It may protect **capacity**:

- avoid overwhelming a customer endpoint;
- avoid one tenant monopolizing workers;
- cap pressure on a downstream service.

The same mechanism can serve different purposes:

```text
to: 5 per company
  -> capacity control

to: 1 per ledger account
  -> serialization / correctness
```

So the mechanism alone does not tell us the architectural intent.

## Database constraints as concurrency control

Not every race needs an explicit lock.

Sometimes the safest solution is to express the invariant directly in the database.

Examples:

```sql
UNIQUE(company_id, external_reference)
```

or:

- `CHECK`
- foreign keys
- exclusion constraints
- `INSERT ... ON CONFLICT`

Two workers may race, but the database prevents the invalid state from being committed.

This is often preferable when the invariant can be represented declaratively.

A useful rule is:

> **If an invariant can be enforced by the database schema, make the database the final line of defence.**

Application-level coordination may still improve behaviour or error handling, but it should not be the only protection when a strong database constraint is available.

## Process-local mutexes

A mutex can serialize threads inside a process:

```text
mutex.synchronize {
  ...
}
```

But in a multi-process or Kubernetes deployment:

```text
Pod A -> Mutex A
Pod B -> Mutex B
```

They do not coordinate with each other.

Process-local locks are useful for in-memory resources, but usually not for protecting persisted business invariants in a distributed application.

## Distributed locks

When participants do not share the same database, coordination may require an external system such as:

- Redis
- etcd
- Consul
- ZooKeeper

These locks operate on logical resources across multiple systems, but introduce additional failure modes:

- lease expiry;
- network partitions;
- paused processes;
- split brain;
- stale lock holders;
- the need for fencing tokens.

A distributed system does not automatically require a distributed lock. If every participant already shares PostgreSQL, database-backed coordination is often simpler.

## `FOR UPDATE SKIP LOCKED`

`SKIP LOCKED` solves a different problem.

Instead of waiting for a row another worker already owns:

```sql
SELECT ...
FOR UPDATE
SKIP LOCKED;
```

the worker skips it and looks for other work.

Example:

```text
Row A -> locked by Worker 1
Row B -> available
Row C -> available

Worker 2
  SKIP LOCKED
  -> processes Row B
```

This is particularly useful for:

- database-backed queues;
- batch processing;
- multiple workers consuming independent records.

The objective is not to serialize all work. It is to **distribute work without unnecessary waiting**.

## Transaction isolation

Locking is only one part of concurrency control.

PostgreSQL also provides isolation levels such as:

```text
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

Higher isolation levels prevent more anomalies, but can reduce effective concurrency and cause transactions to abort and require retries.

`SERIALIZABLE`, for example, attempts to make concurrent transactions behave as if they had run in some serial order.

Isolation level, locks, constraints and retry policies should therefore be considered together rather than independently.

## Choosing the mechanism

A useful first question is:

> **What exactly is the shared resource?**

| Resource / requirement | Typical mechanism |
|---|---|
| One database row | Pessimistic row lock |
| One row, rare conflicts | Optimistic locking |
| Business resource spanning several rows | Advisory lock |
| Jobs that must not overlap per key | Queue concurrency |
| Limited parallelism per tenant/resource | Semaphore / concurrency limit |
| Persisted data invariant | Database constraint |
| Parallel workers consuming independent rows | `FOR UPDATE SKIP LOCKED` |
| In-memory resource inside one process | Mutex |
| Cross-system coordination without a shared database | Distributed lock |

The choice should follow the invariant, not the API.

## Mental model

Several mechanisms can produce the visible behaviour:

```text
only one operation at a time
```

but they encode different architectural guarantees.

### Row lock

```text
The entity protects its state.
```

### Advisory lock

```text
A logical business resource is serialized.
```

### Queue concurrency

```text
The execution system prevents commands from overlapping.
```

### Database constraint

```text
The invalid result cannot be persisted.
```

The central design question is therefore:

> **Where should the exclusivity guarantee live?**

## Failure modes to think about

Before adding a lock or concurrency limit, ask:

1. What invariant am I protecting?
2. What is the actual shared resource?
3. Which actors can modify it?
4. Do all of those actors participate in this mechanism?
5. Do I need exclusivity (`1`) or only bounded concurrency (`N`)?
6. How long can the lock or lease be held?
7. What happens if the process dies?
8. What happens under contention?
9. Can multiple locks be acquired in conflicting orders?
10. Can a database constraint enforce the invariant instead?
11. Does the guarantee belong near the data or in the execution layer?
12. What happens when a new endpoint, job, script or consumer is introduced?

That last question is especially important.

A concurrency strategy may be correct today and silently become unsafe when a new entry point mutates the same resource without participating in the original protocol.

## Reusable principles

- Identify the **business invariant** before choosing the lock.
- Protect the smallest resource that correctly represents that invariant.
- Keep locked transactions short.
- Prefer database constraints for invariants that can be expressed declaratively.
- Treat advisory locks as a cooperative protocol.
- Treat queue concurrency as an execution-layer guarantee, not automatically as a data-layer guarantee.
- A concurrency limit of `1` can provide serialization; a limit of `N` usually provides capacity control.
- Avoid holding database transactions open while merely waiting for work when the coordination mechanism allows otherwise.
- Design the failure and retry behaviour together with the locking strategy.
- Re-evaluate the protection whenever a new entry point can modify the resource.

## Related concepts

- [[Database transactions]]
- [[Transaction isolation]]
- [[Idempotency]]
- [[Background jobs]]
- [[Multi-tenant workload isolation]]
- [[Race conditions]]
- [[Distributed systems]]
