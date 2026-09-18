---
title: "Where Should Validation Live?"
tags:
  - system-design
  - backend
  - rails
  - domain-design
  - api-design
---

"Where should we validate this?" sounds like a simple application-design question.

It usually is not.

A modern backend may need to validate several fundamentally different things:

- JSON shape;
- external types;
- required parameters;
- API-version-specific rules;
- mutually exclusive representations;
- business invariants;
- entity state;
- persistent integrity;
- concurrency-sensitive constraints.

Treating all of those as "validation" encourages us to put unrelated rules in the same place.

A more useful question is:

> **Which layer understands the meaning of this failure?**

---

## A layered model

Consider a Rails-like backend:

```text
HTTP API
    │
    ▼
controller / request adapter
    │
    ▼
use case / domain
    │
    ▼
model / persistence
    │
    ▼
database
```

There may also be other entry points:

```text
jobs
rake tasks
scripts
internal calls
other adapters
```

That matters.

A controller is not necessarily the boundary through which every operation enters the system.

So a useful default is:

| Concern | Natural boundary |
|---|---|
| Payload shape | Request / adapter |
| External types | Request / adapter |
| Required request fields | Request / adapter |
| API versioning | Request / adapter |
| Representation-specific XOR rules | Request / adapter |
| Business invariants | Use case / domain |
| Valid persisted entity state | Domain / model |
| Hard persistent integrity | Database |
| Concurrency-sensitive uniqueness | Database |

This is not about finding one perfect validation layer.

It is about assigning different responsibilities to the layers that have the right knowledge.

---

## The request boundary understands representation

Suppose an API expects:

```json
{
  "recipient": "Alice"
}
```

but receives:

```json
{
  "recipient": ["Alice"]
}
```

This is not yet a business error.

It is a representation error.

The request boundary knows:

- that the payload is JSON;
- that `recipient` is an external field;
- that this version of the API expects a string;
- how malformed input should be reported to the caller.

The domain should not need to discover the problem accidentally because some later operation assumes:

```ruby
recipient.length
```

or because an array happens to trigger a business validation.

A malformed request should remain a malformed request.

---

## Missing, null, wrong type and blank are different failures

These inputs are not equivalent:

### Missing

```json
{}
```

### Null

```json
{
  "recipient": null
}
```

### Wrong type

```json
{
  "recipient": ["Alice"]
}
```

### Blank

```json
{
  "recipient": ""
}
```

The first three are primarily about the external contract.

The last one may be a domain concern.

For example:

```text
Array instead of String
        │
        ▼
request boundary
        │
        ▼
invalid request
```

while:

```text
""
 │
 ▼
domain
 │
 ▼
recipient cannot be empty
```

if "a payment must have a non-empty recipient" is a genuine business invariant.

That distinction improves both architecture and error semantics.

---

## Parameter filtering is not necessarily type validation

Rails gives us useful boundary tools such as:

```ruby
params.permit(...)
```

and newer APIs such as:

```ruby
params.expect(...)
```

They help control which structure is allowed into the application.

But accepting a key does not automatically mean that the value has been fully proven to satisfy the application's external type contract.

This distinction is easy to miss:

> **Filtering parameters and validating an input schema are related, but different responsibilities.**

For simple APIs, built-in Rails mechanisms plus a small amount of explicit validation may be enough.

For richer contracts, a request object, schema library or OpenAPI-driven validation may provide a clearer boundary.

The implementation is secondary.

The boundary is the important part.

---

## Domain invariants must survive other entry points

Now consider a use case:

```ruby
CreatePayment.call(...)
```

Perhaps today it is called from an HTTP controller.

Tomorrow it might also be called from:

```text
background job
rake task
console
internal workflow
another adapter
```

If the rule:

> a payment must have a valid destination

exists only in the HTTP controller, those other entry points can bypass it.

This is the core test for domain validation:

> **Would this still need to be true if there were no HTTP request at all?**

If yes, it probably belongs at the use-case/domain boundary or below.

---

## Why "validate everything in the domain" is also problematic

The opposite extreme is to move every rule into the domain because it is the common entry point.

That causes a different kind of coupling.

The domain should not normally need to know:

```text
API version 2026-05
API version 2026-06
JSON null vs missing
BFF field names
partner-specific payload shape
```

Those are properties of external representation.

Imagine that one API version accepts:

```text
recipient + destination
```

while a later one accepts:

```text
counterparty_id
XOR
recipient + destination
```

At the adapter boundary, that XOR may be essential.

But both representations could resolve to the same internal concept:

```text
counterparty_id ─────────┐
                         ├──► ResolvedDestination
recipient + destination ─┘
```

The domain can then enforce the stable rule:

> A payment requires a valid destination.

That is much less coupled to transport history.

---

## API contracts change more often than domain meaning

Versioning is a strong clue.

If a rule exists because:

- API v1 works one way;
- API v2 works another way;
- the BFF uses a different payload;
- a partner has another representation;

then the rule probably belongs to an adapter.

A useful architecture is:

```text
API v1 ──┐
API v2 ──┼──► map / validate ──► internal input ──► domain
BFF ─────┘
```

The adapters absorb representational differences.

The domain operates on more stable concepts.

This does not mean external contracts are less important.

It means they are important for a different reason.

---

## XOR rules: contract or invariant?

Rules such as:

```text
A
XOR
B + C
```

are especially easy to misclassify.

Ask:

> Is this exclusion fundamental to the business, or is it just how this interface lets callers identify the same thing?

For example:

```text
counterparty_id
```

and:

```text
recipient + destination
```

may simply be two ways to identify a destination.

If so:

```text
external representation A ──┐
                            ├──► destination
external representation B ──┘
```

The XOR belongs to the adapter.

After mapping, the domain only cares that a valid destination exists.

---

## Models are a safety net, not necessarily the public contract

ActiveRecord validations can still be useful:

```ruby
validates :name, presence: true
```

They protect persisted state across multiple code paths.

That is valuable.

But pushing an entire external API contract into the model creates problems:

- version-specific rules leak into persistence;
- HTTP semantics become model semantics;
- public error behaviour depends on persistence details;
- entity validity and request validity become indistinguishable.

A useful way to think about it is:

> **The model protects the persisted entity. The request boundary protects the external contract.**

Sometimes the same rule appears in both places for different reasons.

That is not automatically bad duplication.

---

## Some guarantees belong in the database

Application validation cannot guarantee everything.

Consider uniqueness:

```ruby
validates :external_id, uniqueness: true
```

Two concurrent processes can still both pass the Ruby validation before either commits.

The real guarantee is:

```sql
UNIQUE (external_id)
```

Likewise, the database is often the right final boundary for:

- `NOT NULL`;
- foreign keys;
- unique constraints;
- simple `CHECK` constraints.

The application layer provides meaning and useful errors.

The database provides hard integrity.

```text
application validation
        │
        ▼
clear semantics
        │
        ▼
database constraint
        │
        ▼
hard guarantee
```

These are complementary responsibilities.

---

## Multiple entry points are the architectural test

The easiest way to expose bad validation placement is to draw all entry points.

```text
                 HTTP
                   │
            request contract
                   │
                   ▼
                 adapter
                   │
        ┌──────────┼──────────┐
        │          │          │
       job        rake      internal
        │          │          │
        └──────────┼──────────┘
                   ▼
                use case
                   │
                   ▼
                 domain
```

Then ask:

> Which rules must still hold for every path?

Those belong at or below the common boundary.

And:

> Which rules only make sense for one path?

Those belong in that adapter.

This is more reliable than debating "fat controller" versus "rich domain" in the abstract.

---

## Error semantics improve when boundaries are explicit

A malformed request should not masquerade as a business failure.

Bad flow:

```text
HTTP sends Array instead of String
          │
          ▼
controller accepts it
          │
          ▼
domain sees unexpected type
          │
          ▼
business error
```

The caller receives the wrong explanation.

Better:

```text
wrong external type
        │
        ▼
request boundary
        │
        ▼
invalid_request
```

while:

```text
valid String, invalid meaning
        │
        ▼
domain
        │
        ▼
domain error
```

This is not cosmetic.

Error categories are part of the contract between layers.

---

## Implementation options

There are several reasonable ways to build the request boundary in Rails.

### Strong Parameters / `expect`

Good for:

- allowed keys;
- nested shape;
- simple contracts.

Cheap and idiomatic, but not necessarily a complete schema/type system.

### Manual controller validation

Good for a small number of simple rules.

Very explicit, but scales poorly when the contract grows.

### Request / Form objects

Useful when controllers start accumulating:

- type checks;
- required fields;
- combinations;
- normalization;
- contract-specific errors.

They create a clear testable adapter boundary.

### Schema libraries

Tools such as `dry-schema` can provide:

- explicit types;
- coercion;
- structured errors;
- reusable contracts.

Useful when the API contract is rich enough to justify another abstraction.

### OpenAPI / JSON Schema

Useful when the external contract is already formally described and the team wants documentation and validation to align.

Still not a substitute for domain invariants.

### Use-case/domain validation

Right place for rules that must survive every adapter.

Wrong place for JSON shape and API history.

### ActiveRecord validation

Useful as persisted-entity protection.

Not necessarily the right source for public API semantics.

### Database constraints

The final integrity boundary.

Essential for guarantees that must survive concurrency and every application code path.

---

## A practical decision heuristic

When adding a validation, ask these questions in order.

### 1. Does the rule depend on how the input is represented?

Examples:

```text
JSON structure
String vs Array
missing vs null
parameter names
API version
```

Put it at the adapter/request boundary.

### 2. Must it remain true if the operation is called from a job or rake task?

Put it in the use case/domain.

### 3. Does it define a valid persisted entity?

Consider the model/domain layer as a safety net.

### 4. Must it remain true under concurrency or regardless of application code?

Enforce it in the database.

This turns "where should validation live?" into several smaller questions.

---

## A compact model

```text
              EXTERNAL INPUT
                    │
                    ▼
        ┌─────────────────────┐
        │ Request / Adapter   │
        │                     │
        │ shape               │
        │ external types      │
        │ requiredness        │
        │ API versioning      │
        │ representation XOR  │
        └──────────┬──────────┘
                   │
             mapped input
                   │
                   ▼
        ┌─────────────────────┐
        │ Use case / Domain   │
        │                     │
        │ invariants          │
        │ business meaning    │
        │ valid operations    │
        └──────────┬──────────┘
                   │
                   ▼
        ┌─────────────────────┐
        │ Model / Persistence │
        │                     │
        │ entity safety net   │
        └──────────┬──────────┘
                   │
                   ▼
        ┌─────────────────────┐
        │ Database            │
        │                     │
        │ hard integrity      │
        │ concurrency safety  │
        └─────────────────────┘
```

It is not a rigid architecture.

It is a model for ownership of validation knowledge.

---

## Reusable principles

### Validation is not one responsibility

Input contracts, business invariants and persistent integrity solve different problems.

### Validate where the meaning is known

A layer should reject what it can understand precisely.

### The domain should survive multiple adapters

If a rule must hold for HTTP, jobs, rake tasks and internal calls, it cannot depend only on the controller.

### Do not leak transport history into the domain

API versions and payload representations should normally disappear during mapping.

### Models are useful safety nets

But persisted-state validation and public-request validation are not the same contract.

### Databases provide guarantees applications cannot

Especially under concurrency.

### Error categories are architectural

A malformed request should not become a fake business error.

---

## Questions I should be able to answer

1. What is the difference between request validation and a domain invariant?
2. Why is parameter filtering not necessarily type validation?
3. Why do `missing`, `null`, `wrong type` and `blank` matter differently?
4. Why do background jobs and rake tasks affect validation placement?
5. Why should API versioning usually stay outside the domain?
6. How do I decide whether an XOR rule is contract-specific or domain-specific?
7. What role should model validations play?
8. Which rules require database constraints?
9. When is duplicating a validation across layers justified?
10. How do explicit validation boundaries improve error semantics?

---

## Related concepts

- [[Hexagonal architecture]]
- [[Domain invariants]]
- [[API contracts]]
- [[Defense in depth]]
- [[Database constraints]]
- [[Error semantics]]
