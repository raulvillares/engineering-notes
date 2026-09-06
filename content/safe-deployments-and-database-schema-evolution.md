---
title: Safe Deployments and Database Schema Evolution
tags:
  - deployments
  - databases
  - reliability
  - kubernetes
---

A deployment is not an atomic event.

For some period of time, production may contain a mixture of:

- old application code;
- new application code;
- the old database schema;
- an already migrated schema;
- a migration that is still running.

This means that safe deployments are largely about making those intermediate states valid.

> A deploy is not an instant. For a while, the system may exist in a hybrid state.

## The incident that made this concrete for me

In a system I worked on, a deployment introduced a new database column together with application code that depended on it.

The database migration took longer than expected.

Meanwhile, the rollout of the new application version had already started.

Conceptually, the sequence looked like this:

```text
migration ─────────────────────►
         rollout ──────────────►
                new instance
                     │
                     └── tries to use the new column
                         before it exists
```

Some application instances therefore started failing while the migration was still running.

The important lesson was not simply:

> The migration was slow.

The deeper problem was:

> The release process allowed code that depended on a schema change to run before that schema change was guaranteed to be available.

That distinction matters because making the migration faster would reduce the probability of failure, but would not remove the unsafe state.

## Release vs deployment

I find it useful to separate these two ideas.

### Release

A release is the logical set of changes we want to put into production.

It may include:

- application code;
- configuration changes;
- database migrations;
- data migrations;
- tasks that must run before or after application startup.

### Deployment

A deployment is the process by which application instances are replaced or started.

A single logical release may therefore involve several independent operations.

This is important because those operations may not happen simultaneously.

## Rolling deployments

A rolling deployment replaces old application instances progressively.

For example:

```text
Start:

v1  v1  v1  v1


During rollout:

v1  v1  v2  v2


End:

v2  v2  v2  v2
```

This makes it possible to keep the service available while deploying.

But it introduces an important constraint:

> Old and new application versions may use the same database at the same time.

Database changes therefore have to be designed with that overlap in mind.

## Schema evolution

A database migration being syntactically valid does not mean it is safe to deploy.

The more useful question is:

> Which versions of the application may be running before, during and after this schema change?

A schema change should generally preserve compatibility with the application versions that may coexist during deployment and rollback.

This is the broader problem of **schema evolution**.

## Expand and contract

A common way to evolve a schema safely is to split an incompatible change into several compatible steps.

Suppose we want to replace:

```text
full_name
```

with:

```text
first_name
last_name
```

Changing everything at once creates a dependency between the new code and the new schema.

Instead, the change can be split into phases.

### 1. Expand

Add the new schema while keeping the old one:

```text
full_name
first_name
last_name
```

The existing application continues to work.

### 2. Transition

Deploy application code that understands the new schema.

Depending on the change, this phase may also include:

- backfilling data;
- reading from both representations;
- writing to both representations;
- gradually switching readers and writers.

### 3. Contract

Once no running or rollback version depends on the old representation, remove it:

```text
first_name
last_name
```

The important property is not the exact number of releases.

It is this:

> Do not introduce an incompatible schema transition as one indivisible step.

## Ordering and compatibility are different problems

This distinction became especially useful to me.

Suppose the deployment pipeline guarantees:

```text
migration
    │
    ▼
new application
```

That provides an **ordering guarantee**.

The new version cannot start before the migration finishes.

But while the migration is executing, the old application may still be running:

```text
old application + migration
```

That combination must also be safe.

And if the migration succeeds but the new deployment fails:

```text
migration ✅
new application ❌
rollback → old application
```

the old version may need to work against the migrated schema.

So:

```text
Migration gate
      │
      └── solves ordering

Expand / contract
      │
      └── solves compatibility
```

They are complementary mechanisms, not alternatives.

## Release phases

Some platforms provide an explicit release phase: a task runs before the new application release is started.

A typical example is:

```text
db:migrate
    │
    ▼
success
    │
    ▼
start new release
```

This creates a useful guarantee:

> Application code that depends on the migration will not start before the migration task has completed successfully.

A higher-level PaaS can provide this lifecycle as part of the platform abstraction.

## Kubernetes and migration ordering

Kubernetes provides primitives such as:

- Deployments;
- Pods;
- Jobs;
- rolling updates;
- readiness checks.

A Kubernetes Job can execute a migration:

```text
Job: db:migrate
      │
      ▼
   success
```

But a Job by itself does not express:

> Do not begin the application rollout until this Job has succeeded.

That dependency must be established by the deployment pipeline or another orchestration layer.

Conceptually:

```text
Migration Job
      │
      ▼
wait for success
      │
      ▼
Deployment rollout
```

This taught me a broader lesson about moving from a higher-level platform to lower-level infrastructure:

> When changing platforms, identify which guarantees the old platform provided implicitly and decide where those guarantees now live.

More control usually means owning more lifecycle decisions.

## Separating schema changes from dependent code

One conservative strategy is to ensure that a schema change and the code that depends on it are deployed separately.

### Release A

Introduce the compatible schema change:

```sql
ADD COLUMN new_field;
```

The existing application must continue to work.

### Release B

Deploy the code that uses:

```text
new_field
```

Now the application can depend on the column because its existence is no longer part of the same deployment transition.

This makes a whole class of ordering failures much harder to produce.

It does not, however, remove the need for compatibility thinking.

A destructive migration, data transformation or rollback can still create unsafe intermediate states.

## The mechanisms solve different problems

I keep these four concepts separate:

| Mechanism | Main problem it addresses |
| --- | --- |
| Expand / contract | Compatibility during schema evolution |
| Separate schema and dependent code | Prevent dependent code from appearing too early |
| Release phase / migration gate | Ordering between migration and rollout |
| Rolling deployment | Progressive instance replacement without full downtime |

A robust deployment process may use several of them together.

## Zero-downtime deployments

A zero-downtime deployment is not simply:

> Kubernetes keeps some Pods alive.

It requires thinking about the whole transition:

- old and new application versions;
- schema compatibility;
- long-running migrations;
- rollout ordering;
- readiness;
- rollback;
- intermediate states.

The system must remain valid throughout the transition, not merely before and after it.

## Mental model

The model I use is:

```text
SAFE SCHEMA EVOLUTION
        │
        └── expand / contract
            compatibility


RELEASE PIPELINE
        │
        └── migration → rollout
            ordering


DEPLOYMENT STRATEGY
        │
        └── rolling deployment
            progressive replacement


PLATFORM
        │
        └── PaaS / Kubernetes / pipeline
            who provides each guarantee
```

This turns the question from:

> How does Kubernetes handle database migrations?

into:

> What guarantees does the release lifecycle need so that different application and schema versions can safely coexist?

That is the more general problem.

## Questions I find useful

When reviewing a schema-changing release:

1. Can the current application run while this migration is executing?
2. Can old and new application versions use the migrated schema simultaneously?
3. What happens if the migration succeeds but the application rollout fails?
4. Can we safely roll back?
5. Is any application code depending on a change that may not exist yet?
6. Which system guarantees migration-before-rollout ordering?
7. Are we relying on a guarantee provided implicitly by the deployment platform?
8. Would splitting the schema change and the dependent code reduce risk?

## Related concepts

This connects naturally to:

- [[Rolling Deployments]]
- [[Backward Compatibility]]
- [[Database Migrations]]
- [[Expand and Contract]]
- [[Readiness Checks]]
- [[Rollback]]
- [[Zero-Downtime Deployments]]

---

This note came from investigating a production deployment failure after moving an application from a higher-level PaaS deployment model to Kubernetes-based infrastructure.