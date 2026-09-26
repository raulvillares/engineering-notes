---
title: Reproducing Production Data Volume Safely
tags:
  - infra
  - infra-fluency
  - system-design
  - performance
  - postgres
---

# Reproducing Production Data Volume Safely

When a performance problem only appears for customers with large datasets, reproducing it outside production is not always straightforward.

The goal should not necessarily be to **copy production**, but to reproduce the characteristics that actually trigger the problem:

- row count;
- data distribution;
- query shape;
- indexes and execution plans;
- concurrency;
- latency;
- application behavior under large datasets.

Different strategies offer different levels of fidelity, safety, and freedom to experiment.

## 1. Test directly in production

The highest-fidelity option is to run the real flow against real production data.

This can be useful when:

- the issue depends heavily on the real dataset size;
- we need to validate a very specific behavior;
- there is no reliable reproduction outside production.

### Advantages

- Maximum fidelity.
- Real data, indexes, statistics, and database configuration.
- Real infrastructure.
- Lets us observe what users actually experience.

### Trade-offs

- Risk of affecting production.
- A read-only query can still consume significant resources.
- `EXPLAIN ANALYZE`, for example, actually executes the query.
- Destructive or highly exploratory testing is out of the question.
- Queries must be well understood and tightly controlled.

Production can be a good place to **observe and measure**, but not a general-purpose sandbox.

## 2. Analyze the real queries in production

Often we do not need to reproduce the whole system.

A better approach can be to reduce the problem until we identify the SQL queries behind the problematic flow.

A typical process is:

1. identify the endpoint or use case involved;
2. follow the execution path down to the relevant SQL;
3. measure timings;
4. compare different data volumes or time windows;
5. inspect query plans with `EXPLAIN`;
6. identify which operation actually scales with volume.

This can answer questions such as:

- Which query is consuming the time?
- Is the cost in the listing, a `COUNT`, a `SUM`, a join, or application-side processing?
- Is the expected index being used?
- Is a sequential scan involved?
- Does the execution plan change as the dataset grows?
- Does the original performance problem still exist?

### Advantages

- Uses the real dataset without copying it.
- Helps isolate the actual bottleneck.
- Risk can remain low if queries are known and controlled.
- Often enough to determine whether a historical workaround is still necessary.

### Limitations

- Does not reproduce the complete user flow.
- Provides little freedom for experimentation.
- A heavy read query is still heavy.
- Results can vary depending on cache state, concurrent load, statistics, vacuum state, and other database conditions.

### Example

In one investigation, a feature flag reduced the default transaction history window from 30 days to 1 day because the larger range had historically caused timeouts for high-volume accounts.

Instead of cloning production data, the execution path was followed from the API request down to the database operations, and the expensive parts were measured independently.

The results were roughly:

| Operation | 1 day | 30 days |
| --- | ---: | ---: |
| First page + balance calculation | ~100 ms | ~100 ms |
| `total_count` | 15 ms | 564 ms |

The wider time window made the `COUNT` substantially more expensive, but the full request still remained comfortably below timeout territory.

The query plan also showed why: the expensive `COUNT` performed a parallel sequential scan across transactions in the requested time range and then matched the relevant account through an index.

The important lesson was not the specific query, but the investigation method:

> Before reproducing all of production, reduce the problem until you know which operation actually scales with data volume.

## 3. Generate synthetic data with production-like volume

Another strategy is to generate an artificial dataset with characteristics similar to the production dataset causing the problem.

For example:

- create a tenant or company;
- create accounts;
- generate hundreds of thousands or millions of transactions;
- preserve a realistic temporal distribution;
- reproduce relevant entity relationships;
- run the same application flow against that dataset.

The goal is not to copy the real data.

The goal is to reproduce **the properties of the data that matter**.

### Advantages

- No production data exposure.
- No production risk.
- Full freedom to modify or destroy data.
- Easy to test volumes larger than current production.
- Useful for finding degradation thresholds.
- Repeatable.
- Can become a reusable performance-testing tool.

### Limitations

Volume alone does not guarantee a realistic reproduction.

Synthetic data can differ from production in:

- temporal distribution;
- cardinality;
- relationships between tables;
- status distribution;
- value distribution;
- checkpoints or aggregates;
- planner statistics;
- table and index bloat;
- concurrency patterns.

A good generator should therefore do more than create `N` rows.

It should model the **shape of the data** that affects the query.

## 4. Use scalable synthetic datasets

A useful variation is to make the synthetic dataset configurable:

```text
small   → 10k records
medium  → 100k records
large   → 1M records
xlarge  → 10M records
```

This changes the question from:

> Is this query slow?

to:

> How does this query behave when the dataset grows by 10x?

This makes it possible to detect:

- roughly linear growth;
- superlinear degradation;
- query planner changes;
- sequential scans appearing at certain sizes;
- indexes no longer being selected;
- latency thresholds where the behavior becomes unacceptable.

This is particularly useful for studying **scalability**, not only for reproducing incidents.

## 5. Restore a production dump locally

Another option is to restore a production database dump into a local or isolated environment.

### Advantages

- Highly realistic dataset.
- Full freedom to experiment.
- Safe to modify records, feature flags, and indexes after restoration.
- Experiments do not affect the live system.

### Trade-offs

Creating the dump itself can place meaningful load on the primary database.

It also introduces the operational and security cost of copying production data into another environment:

- privacy concerns;
- security concerns;
- compliance requirements;
- credential management;
- accidental persistence of sensitive data.

Even when technically possible, this should not be treated as the default option.

## 6. Restore a dump from a read replica

If a read replica exists, a dump can be taken from the replica instead of the primary.

This reduces direct pressure on the primary database.

### Advantages

- Realistic dataset.
- Lower operational risk for the primary.
- Full freedom to experiment after restoring it elsewhere.

### Limitations

- Production data still leaves the production environment.
- The replica still has finite capacity.
- Replication lag may matter.
- Access may be restricted.

This addresses part of the operational risk, but not the risks associated with copying production data.

## 7. Connect a local application to a production database

It may be technically possible to run a local application against a production database or read replica.

This gives local code access to the real dataset.

It is also a particularly risky approach.

### Risks

- accidental writes;
- callbacks or background jobs running unexpectedly;
- local code differing from deployed code;
- migrations;
- scripts or commands targeting the wrong environment;
- expensive experimental queries.

Even with read-only credentials, there is still a risk of creating unwanted load.

For this reason, this approach is often better avoided entirely.

## 8. Use a read replica for analysis

A read replica can be useful for:

- `EXPLAIN`;
- exploratory analysis;
- volume queries;
- index validation;
- measuring expensive reads.

It provides some isolation from the primary database.

However, a replica should not be treated as an unlimited sandbox. Heavy queries can still affect replication, reporting workloads, analytics, or other consumers.

## Comparison

| Strategy | Fidelity | Risk to production | Freedom to experiment | Real data |
| --- | --- | --- | --- | --- |
| Full flow in production | Very high | High | Very low | Yes |
| Controlled SQL in production | High | Low–medium | Low | Yes |
| SQL on a read replica | High | Low | Medium | Yes |
| Local app → production DB | High | High | Medium | Yes |
| Production dump → local | High | Medium during dump | Very high | Yes |
| Replica dump → local | High | Low–medium | Very high | Yes |
| Synthetic data | Medium–high | None | Very high | No |

There is no universally best strategy.

The right choice depends on **which property of production we actually need to reproduce**.

## A practical investigation sequence

### 1. Reduce the problem

Before moving large amounts of data around, determine:

- which endpoint is slow;
- which use case it triggers;
- which SQL queries it generates;
- which operations depend on data volume.

### 2. Measure against real data in a controlled way

If safe, collect timings for the relevant queries using production or a read replica.

The goal is to identify the bottleneck, not to perform arbitrary experiments.

### 3. Understand the execution plan

`EXPLAIN` can reveal:

- sequential scans;
- expensive joins;
- incorrect cardinality estimates;
- which indexes are being used;
- expected row counts;
- relative cost of different plan nodes.

### 4. Build a synthetic dataset

Once the relevant data shape is understood, reproduce it locally:

```text
synthetic data
      ↓
production-like or larger volume
      ↓
same backend flow
      ↓
same application behavior
```

### 5. Find the breaking point

Do not stop at the current production volume.

Test larger datasets too:

```text
1x current production volume
2x
5x
10x
```

This helps distinguish between a fix for today's incident and a design that scales.

## Key idea

Reproducing production does not necessarily mean **copying production**.

For performance investigations, the important question is which dimensions of the production environment actually matter, and how to reproduce those dimensions safely.

Sometimes analyzing the real queries is enough.

Sometimes we need production-like synthetic data.

Only in the most constrained cases do we need to run the full flow against live production infrastructure.

The useful question is not:

> How can I get production locally?

It is:

> Which property of production do I need to reproduce for the same problem to appear?
