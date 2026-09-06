---
title: Schema Changes, Rails Caches, and Long-Lived Processes
tags:
  - rails
  - postgresql
  - databases
  - deployments
  - reliability
---

A database migration does not affect only the database.

Long-lived application processes can keep state that was created against an earlier version of the schema.

In a Rails application, that may include things such as:

- Active Record's knowledge about model columns and types;
- database connections kept in a connection pool;
- prepared statements associated with those connections.

This creates another kind of hybrid state during a deployment:

```text
PostgreSQL = schema B
Rails process = state derived from schema A
```

This is one reason schema changes need to be considered together with the lifecycle of application processes.

See also [[Safe Deployments and Database Schema Evolution]].

## The incidents that made this concrete for me

I encountered two production failures after an application moved from a higher-level PaaS deployment model to Kubernetes.

They had different immediate causes, but both exposed the same underlying idea:

> A process that survives a schema change may continue carrying assumptions created before that change.

### A process starting too early

In one deployment, new application code introduced an attribute backed by a new database column.

The schema migration and application rollout started at roughly the same time.

Some new processes started before the migration had created the column.

Conceptually:

```text
new process starts
        │
        ▼
observes schema A
        │
migration finishes
        │
        ▼
PostgreSQL now has schema B
        │
        ▼
process remains alive
with state derived from schema A
```

The database was eventually correct, but some processes continued failing until they were restarted.

That was an important distinction:

> Finishing the migration corrected PostgreSQL, but it did not necessarily reset the state of application processes that were already running.

## Schema metadata and process lifetime

Rails processes are long-lived.

They do not rediscover every property of the database from scratch for every request or job.

Conceptually:

```text
PostgreSQL
    │
    └── current schema

Rails process
    │
    ├── model/schema knowledge
    │
    └── connection pool
            │
            └── connection-specific state
```

If a schema changes underneath a running process, there can therefore be a temporary mismatch between:

- what PostgreSQL currently contains;
- what the application process previously learned or prepared.

A restart removes that process-local state:

```text
restart
   │
   ▼
new process
   │
   ├── new application state
   └── new database connections
            │
            ▼
       schema B
```

This explains why restarting a pod can fix some deployment-related database errors even though Kubernetes has not changed anything in PostgreSQL itself.

But:

> Restarting processes is a recovery mechanism, not a substitute for a safe schema evolution strategy.

## Prepared statements add another layer

Prepared statements make this more subtle.

Suppose a connection prepares a query such as:

```sql
SELECT payments.*
FROM payments
WHERE id = $1;
```

At that point, PostgreSQL knows the structure of the query result.

Now imagine a migration changes that structure:

```sql
ALTER TABLE payments
ADD COLUMN new_field text;
```

The meaning of:

```sql
SELECT payments.*
```

has changed.

A previously prepared statement can therefore become incompatible with the new result shape.

PostgreSQL may report:

```text
cached plan must not change result type
```

and Rails surfaces this kind of failure as:

```text
ActiveRecord::PreparedStatementCacheExpired
```

This illustrates an important distinction:

```text
database schema
      │
      ▼
prepared statement result shape
      │
      ▼
state attached to a long-lived connection
```

A connection pool means those connections may survive across many requests or background jobs.

## Why explicit column enumeration helps

Rails has an option:

```ruby
config.active_record.enumerate_columns_in_select_statements = true
```

which changes generated queries away from wildcard selections such as:

```sql
SELECT payments.*
```

towards explicit columns:

```sql
SELECT id, amount, status
FROM payments
```

For an old process that does not know about a newly added column, adding that column no longer changes the result shape of its existing query.

That removes an important source of prepared statement cache failures during additive schema changes.

It is not a universal solution.

For example, changing the type of a column that is already part of the result can still alter the prepared statement's expected result type.

The broader lesson is:

> Explicitly enumerating columns reduces coupling between additive schema changes and prepared query result shapes.

## Transactions can make recovery harder

There is another complication when a prepared statement fails inside a database transaction.

After certain PostgreSQL errors, the transaction is considered failed until it is rolled back.

This matters because recovery operations may themselves need a usable connection.

A failure can conceptually look like:

```text
prepared statement fails
        │
        ▼
transaction becomes invalid
        │
        ▼
connection needs cleanup / rollback
```

In one incident I worked on, application code rescued a broad database-related exception inside the transactional flow and prevented it from reaching the layer responsible for completing the normal transaction failure lifecycle.

The result was not merely one failed operation.

The problematic connection remained reusable by subsequent jobs, which could encounter the same invalid prepared statement again.

Restarting the process destroyed the old connections and removed the persistent failure.

The lesson I take from this is not:

> Never rescue `StandardError`.

It is:

> Be very careful about swallowing database exceptions inside a transaction when the surrounding transaction or adapter needs to observe the failure in order to recover correctly.

Error handling is part of transactional correctness, not only application control flow.

## Migration, rollout and restart are different operations

I find it useful to keep these separate.

### Migration

Changes database schema or data.

```text
schema A → schema B
```

### Rollout

Progressively replaces application processes.

```text
v1 v1 v1 v1
      ↓
v1 v1 v2 v2
      ↓
v2 v2 v2 v2
```

### Restart

Destroys a particular process and starts a fresh one.

```text
old process
     ↓
destroy
     ↓
new process
```

They affect different kinds of state.

A migration changes PostgreSQL.

A rollout changes which application version is running.

A restart also removes process-local and connection-local state.

This is why saying _"the migration completed successfully"_ does not necessarily imply that every running application process is now healthy.

## Migration before rollout

One useful deployment guarantee is:

```text
migration
    │
    ▼
wait for success
    │
    ▼
rollout
```

This prevents new application processes from starting against a schema that has not yet been migrated.

It would prevent the class of failure where new code observes the old schema during startup.

But this is still not sufficient by itself.

During the migration, the old application may still be running:

```text
old application + new migration
```

And if deployment of the new version fails:

```text
migration ✅
new version ❌
rollback → old version
```

the old application may need to work against the migrated schema.

So, as described in [[Safe Deployments and Database Schema Evolution]]:

```text
migration → rollout
        │
        └── ordering

expand / contract
        │
        └── compatibility
```

Both properties matter.

## PaaS vs Kubernetes

A higher-level platform may hide much of this lifecycle behind its release abstraction.

Moving to Kubernetes exposes more individual primitives:

- Pods;
- Deployments;
- Jobs;
- rolling updates;
- readiness;
- external deployment pipelines.

The useful architectural question is therefore not:

> How does Kubernetes handle Rails migrations?

but:

> Which part of the platform guarantees each property that the application requires during a release?

For example:

```text
schema compatibility
        → application/release design

migration-before-rollout ordering
        → deployment orchestration

process replacement
        → Kubernetes Deployment

process-local state reset
        → process/pod lifecycle
```

Moving to lower-level infrastructure does not necessarily create these problems.

It often makes responsibilities visible that a higher-level platform previously handled implicitly.

## Mental model

The model I keep is:

```text
                    DATABASE
                       │
                       └── schema
                            │
                            ▼
                  APPLICATION PROCESS
                       │
             ┌─────────┴──────────┐
             │                    │
             ▼                    ▼
       schema knowledge      connection pool
                                  │
                                  ▼
                          prepared statements
```

A schema migration can therefore affect several layers of a running system.

When debugging deployment failures, I want to ask:

1. What changed in PostgreSQL?
2. Which processes existed before that change?
3. What database-related state can those processes retain?
4. Which connections survived the migration?
5. Does restarting help because it replaces application state, connection state, or both?
6. Could the release have avoided the incompatible intermediate state entirely?

## Related concepts

- [[Safe Deployments and Database Schema Evolution]]
- [[Prepared Statements]]
- [[Connection Pools]]
- [[Database Migrations]]
- [[Rolling Deployments]]
- [[Expand and Contract]]
- [[Transactions]]
- [[Rollback]]
- [[Long-Lived Processes]]

---

This note came from investigating production failures where database migrations had completed successfully but some long-lived application processes continued behaving as if part of the previous database state still existed.