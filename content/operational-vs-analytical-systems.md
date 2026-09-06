---
title: Operational vs Analytical Systems
tags:
  - data-systems
  - databases
  - analytics
---

Operational and analytical systems may work with the same underlying business data, but they are optimized for very different workloads.

Understanding that distinction is useful because many architectural decisions are really about preventing one kind of workload from harming another.

## Operational systems: OLTP

Operational systems are where data is created and modified as part of the normal behavior of an application.

Typical characteristics include:

- many small queries;
- point lookups;
- inserts, updates and deletes;
- low-latency requirements;
- predictable access patterns;
- a strong focus on the current state of the system.

A typical query might be:

```sql
SELECT *
FROM transfers
WHERE id = ?;
```

The application needs one specific record, usually quickly, because the result affects some user-facing or business operation.

This workload is usually described as **OLTP — Online Transaction Processing**.

Here, _transaction_ should not be understood only as an ACID database transaction. The term describes the broader pattern of interactive, low-latency reads and writes performed by an operational application.

## Analytical systems: OLAP

Analytical systems are used to inspect and aggregate data that has already been produced by operational systems.

Typical characteristics include:

- fewer queries;
- queries that touch many rows;
- aggregations;
- historical analysis;
- ad hoc exploration;
- mostly read-oriented workloads.

For example:

```sql
SELECT customer_id, COUNT(*), AVG(processing_time)
FROM verifications
WHERE created_at >= ?
GROUP BY customer_id;
```

This query might scan hundreds of thousands or millions of rows.

Its purpose is not to decide how a single request should behave. It is to understand patterns across a large dataset.

This workload is commonly described as **OLAP — Online Analytical Processing**.

## Why the distinction matters

An operational query and an analytical query may use the same database engine and even the same tables, but their resource profiles are very different.

A large analytical query can consume:

- CPU;
- memory;
- disk I/O;
- database connections;
- buffer cache.

If it competes directly with latency-sensitive application queries, analytics can degrade the production system.

This leads to an important architectural idea:

> Separate workloads when their performance characteristics and priorities are different.

The separation does not necessarily require completely different database technologies. Sometimes a read replica is enough. In other systems, the analytical workload is moved into a dedicated warehouse or real-time analytics platform.

The important question is not only:

> What database technology is this?

but:

> What workload is this system serving?

## A system I have worked with

In one payments platform I worked on, the architecture contained three distinct ways of using related business data.

The main PostgreSQL database served the application itself:

```text
Application / workers
        │
        ▼
   PostgreSQL
 operational data
```

This was the authoritative source for much of the application's current state.

Internal operational and business teams also needed dashboards, statistics and ad hoc queries. Those were provided through a BI tool connected to a read-oriented datasource:

```text
Operational database
        │
        ▼
Read-oriented datasource
        │
        ▼
       BI
        │
        ▼
Operations / business
```

Separately, business events were continuously propagated through an event pipeline into a real-time analytical system that powered customer-facing dashboards:

```text
Application
    │
    ▼
Business events
    │
    ▼
Event pipeline
    │
    ▼
Real-time analytics
    │
    ▼
Customer-facing dashboards
```

These three paths were all related to the same underlying business activity, but they served different purposes:

- the operational database supported the product itself;
- BI supported internal analysis;
- real-time analytics supported analytical features inside the product.

That distinction helped me understand that **operational vs analytical is primarily a workload distinction, not a product-category distinction**.

## System of record vs derived data

A related distinction is between a **system of record** and **derived data**.

### System of record

The system of record contains the authoritative representation of a fact.

If two systems disagree, this is the system whose value is considered correct.

For example, an operational database might be authoritative for the current state of a payment.

### Derived data

Derived data is produced from other data.

Examples include:

- caches;
- indexes;
- materialized views;
- search indexes;
- analytical datasets;
- data warehouses.

Derived data can often be rebuilt from its source.

This leads to a useful debugging question.

Suppose a customer dashboard reports:

```text
10,001 operations
```

while the operational database contains:

```text
10,003 operations
```

Before investigating the discrepancy, I need to know:

> Which system is authoritative?

If the operational database is the system of record, the mismatch indicates that something went wrong while producing or transporting the derived representation.

This distinction is useful when reasoning about consistency, recovery and failure modes.

## A read replica is not a data warehouse

A read replica is a replicated copy of a database, commonly used to offload read traffic.

It may be useful for:

- reporting;
- analytics;
- reducing load on the primary;
- availability.

But a read replica is not automatically a data warehouse.

It can preserve exactly the same operational schema and simply serve a different workload.

A **data warehouse** usually goes further:

```text
Operational systems
       │
       ▼
    ETL / ELT
       │
       ▼
Data warehouse
       │
       ▼
      BI
```

It typically:

- integrates data from one or more operational systems;
- separates analytical workloads from production;
- may reshape data into models better suited for analytics;
- is designed specifically for analytical querying.

The distinction matters because saying _"analytics runs somewhere else"_ does not imply that the system is a data warehouse.

## Internal analytics vs product analytics

Another distinction I find useful is between analytics used internally and analytics exposed as part of the product.

### Internal analytics

Used by:

- operations;
- product teams;
- finance;
- business analysts.

Typical tools are BI platforms and dashboards.

### Product analytics

Analytical capabilities delivered directly to users or customers.

These systems may require:

- continuous ingestion;
- low query latency;
- fresh data;
- predictable performance under customer traffic.

This often pushes the architecture towards dedicated real-time analytical systems rather than traditional BI infrastructure.

## Mental model

The model I keep in mind is:

```text
                    Business activity
                           │
               ┌───────────┴───────────┐
               │                       │
               ▼                       ▼
       Operational state          Derived data
               │                       │
               │             ┌─────────┴─────────┐
               │             │                   │
               ▼             ▼                   ▼
             OLTP       Internal analytics   Product analytics
```

The useful questions are:

1. What workload is this system serving?
2. Which system contains the authoritative data?
3. Which representations are derived?
4. Can analytical work interfere with production traffic?
5. How fresh does the analytical data need to be?
6. Can derived data be rebuilt if it becomes inconsistent?

Those questions are usually more useful than starting from the names of the technologies involved.

## Related concepts

This topic connects naturally to:

- read replicas;
- replication lag;
- ETL and ELT;
- data warehouses;
- materialized views;
- event pipelines;
- eventual consistency;
- workload isolation;
- real-time analytics.

---

_Source: Martin Kleppmann, Chris Riccomini, **Designing Data-Intensive Applications**, 2nd edition — sections on operational and analytical systems._
