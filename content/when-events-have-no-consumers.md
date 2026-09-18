---
title: "When Events Have No Consumers"
tags:
  - system-design
  - event-driven-architecture
  - domain-events
  - backend
---

I recently reviewed a system that persisted an entity and then published a `*.created` event for the same operation.

The event had no subscribers.

It was not used for analytics.

Its payload added no information.

The useful data already lived in the row that had just been written.

That raised a simple question:

> What capability exists because this event exists?

If the answer is "none", the event may not be an architectural asset anymore. It may just be another write.

This note is about how I think about that situation.

---

## A durable event should have a responsibility

Publishing an event can enable useful properties:

- decoupling;
- fan-out;
- asynchronous processing;
- integrations;
- projections;
- analytics;
- audit trails;
- replay;
- event-sourced state reconstruction.

But the event itself is not the value.

The value is the capability it enables.

A healthy flow usually looks like this:

```text
fact
 │
 ▼
event
 │
 ├──► consumer A
 ├──► consumer B
 └──► projection / analytics / integration
```

The suspicious version looks like this:

```text
fact
 │
 ▼
event
 │
 └──► nobody
```

That does not automatically make the event wrong.

It does mean the event needs another explicit reason to exist.

---

## The duplicated representation problem

Imagine a request recorder:

```text
HTTP request
    │
    ▼
store request
    │
    ├──► INSERT requests
    │
    └──► INSERT events
           request.created
```

The `requests` row already contains the useful representation of the request.

If `request.created`:

- has no consumers;
- contains no additional information;
- is not part of an audit requirement;
- is not replayed;
- is not the source of truth;
- cannot reconstruct the row;

then the second write is not creating a new capability.

It is another durable representation of the same fact.

That can be fine when intentional.

It becomes questionable when nobody can explain why it is there.

---

## Write amplification

This is a small example of **write amplification**.

One logical operation produces more persistent writes than are strictly necessary for the operation itself.

```text
1 logical operation
        │
        ├──► business / operational row
        └──► durable event
```

Write amplification is not inherently bad.

Indexes, replicas, audit records, caches and event logs all add writes in exchange for useful properties.

The important part is the exchange.

What do we get for the extra write?

If the event is produced on every request or every webhook delivery, the cost scales with normal traffic:

- more rows;
- more index updates;
- more WAL;
- more I/O;
- more vacuum work;
- larger backups;
- more retention or cleanup work;
- more noise when inspecting the event store.

I would not claim this is a performance problem without measurements.

The architectural observation is narrower:

> The system is paying an ongoing write cost for a capability that may not exist.

---

## Not every `created` fact needs a domain event

One reason these events appear is that event infrastructure is already available.

Once there is an `EventPublisher`, this can feel natural:

```text
record created
      │
      ▼
publish RecordCreated
```

But "something happened in the application" is not enough to make it a useful domain event.

A domain event should usually express something meaningful in the domain:

```text
PaymentConfirmed
TransferRejected
AccountClosed
OrderShipped
```

A technical fact is different:

```text
HttpRequestStored
WebhookRequestCreated
JobStarted
CacheRefreshed
```

Technical events can absolutely be useful.

They may support:

- observability;
- tracing;
- metrics;
- technical auditing;
- operational automation.

But that is a different responsibility.

The important distinction is not whether the event is "domain" or "technical" in name.

It is whether its semantics and consumers are clear.

---

## Persisting a row already records a fact

Sometimes the database row is already the right representation.

```text
INSERT requests (...)
```

That operation means:

> this request existed, and here is the information we decided to retain about it.

Adding:

```text
INSERT events (
  name = "request.created",
  payload = {}
)
```

only adds value if the event serves a different purpose.

For example:

```text
                    ┌────────────────┐
incoming request ──►│ requests table │
                    └────────────────┘
                         operational
                           record

incoming request ──► RequestReceived
                          │
                          ├──► security pipeline
                          └──► analytics pipeline
```

Now the two representations have different responsibilities.

Without that difference, the event is mostly duplication.

---

## An event with no subscribers can still be valid

"No subscribers" is a useful warning sign, not a complete decision rule.

A durable event may still matter if it supports:

### Audit

The event log may be the historical record required to answer:

> What happened, and in what order?

### Replay

A process may intentionally reread past events later.

### Analytics

The event may feed a downstream analytical pipeline rather than an application subscriber.

### External consumers

The consumer may live outside the repository being inspected.

### Event sourcing

The event may be part of the authoritative history from which state is rebuilt.

So the right question is not:

> Does this event have a local subscriber?

It is:

> What system responsibility depends on this event?

---

## Event store does not mean event sourcing

This distinction matters.

A system may have a table called `events` or an "event store" without being event-sourced.

In event sourcing:

```text
events
   │
   ▼
replay / fold
   │
   ▼
current state
```

The event history is authoritative.

The current state can be derived from it.

Now compare that with:

```text
requests row
    │
    └──► contains the useful data

request.created
    │
    └──► payload: {}
```

The event cannot reconstruct the request.

The event is not the source of truth.

Removing it would not remove event sourcing, because event sourcing was never happening for that entity.

An event store is infrastructure.

Event sourcing is a data model.

They are not the same thing.

---

## Event-driven architecture does not mean publishing everything

An event-driven system does not become better by maximizing the number of events.

The useful question is:

> Where does asynchronous communication or decoupling improve the system?

For example:

```text
PaymentConfirmed
       │
       ├──► notify customer
       ├──► update reporting
       └──► start reconciliation
```

The event clearly has a role.

By contrast:

```text
RequestStored
       │
       └──► nobody
```

may simply be a record of an implementation detail.

The existence of event infrastructure can make publication cheap in code.

That does not make it free in architecture.

---

## A heuristic: dead events

I find it useful to think in terms of **dead events**.

This is not a formal architectural category.

It is a practical smell.

An event is a good candidate when several of these are true:

- no subscribers exist;
- no external consumers are known;
- it does not feed analytics;
- it does not update a projection;
- it is not replayed;
- it is not required for auditing;
- it is not part of event-sourced state;
- its payload is empty or redundant;
- the same fact already exists elsewhere;
- deleting it would not change observable behaviour.

One signal is weak.

Several together are much stronger.

---

## How dead events appear

They do not necessarily come from bad design.

### A consumer disappeared

Originally:

```text
publisher ──► consumer
```

Later:

```text
publisher ──► ∅
```

The consumer was removed, but the publisher survived.

### The event was added "for later"

Someone expected a future consumer.

That future never arrived.

### A convention became mechanical

The codebase established a pattern:

```text
EntityCreated
EntityUpdated
EntityDeleted
```

and new events started appearing because the pattern existed, not because a consumer needed them.

### The architecture changed

Perhaps the event once carried information that later became available from a dedicated table, projection or service.

The original reason may have been valid.

The current reason may no longer exist.

---

## Future optionality has a present cost

A common argument for keeping an unused event is:

> We may want to consume it later.

That is real optionality.

But optionality is not free.

```text
future convenience
      vs
present cost + complexity
```

Every production operation continues paying for the event today.

If a concrete consumer appears later, adding the event at that point may be cheap and explicit.

Until then, the system is maintaining infrastructure for a hypothetical use case.

This is essentially YAGNI applied to event infrastructure.

---

## Complexity is not only runtime cost

Even if the extra write is cheap, unused events still have a cognitive cost.

When someone sees:

```text
RequestCreated
```

they reasonably assume that it matters.

They may ask:

- Who consumes this?
- Is it part of an integration contract?
- Can it be removed?
- Does replay depend on it?
- Is the event store authoritative?
- Will changing the payload break something?

An event creates expectations.

If there is no responsibility behind it, those expectations become accidental complexity.

---

## Observability is not the same as domain modelling

Another smell appears when durable domain-event infrastructure becomes a generic logging mechanism.

Suppose the real need is:

> I want to know that requests happened.

That might belong in:

```text
request
   │
   ├──► logs
   ├──► metrics
   └──► traces
```

rather than:

```text
request
   │
   └──► domain event store
```

Those mechanisms have different purposes.

Logs answer detailed diagnostic questions.

Metrics answer aggregate operational questions.

Traces connect work across components.

Domain events represent meaningful facts that other parts of the system may need to react to.

Blurring those responsibilities makes both the architecture and the event vocabulary harder to reason about.

---

## Removing an event is also a system design decision

System design is often presented as adding things:

- queues;
- caches;
- services;
- databases;
- streams;
- replicas.

But simplification is also design.

If the current system is:

```text
operation
   │
   ├──► row
   └──► event ──► ∅
```

and careful investigation shows the event has no responsibility, the better system may simply be:

```text
operation
   │
   └──► row
```

Fewer components.

Fewer writes.

Fewer concepts.

Same behaviour.

That is not "just cleanup".

It is reducing accidental architecture.

---

## What I would verify before deleting one

A source-code search for subscribers is necessary, but not sufficient.

I would check:

### Internal consumers

- subscribers;
- handlers;
- jobs;
- callbacks;
- projections.

### Analytics

- event whitelists;
- pipelines;
- BI ingestion;
- product analytics.

### External consumers

- streams;
- webhooks;
- connectors;
- other services or repositories.

### Audit and compliance

Is the event log intentionally used as historical evidence?

### Replay

Does any process reread old events?

### Operational tooling

Are dashboards, scripts or maintenance jobs querying the event?

### Source of truth

Does the row already contain the authoritative information?

### Retention assumptions

Does removing the event affect any operational or compliance retention model?

Only after that can "nobody consumes it" become "nothing depends on it".

---

## A decision model

Before publishing or retaining a durable event, I find this sequence useful:

```text
Has a meaningful fact occurred?
            │
            ▼
Who needs to know?
            │
     ┌──────┴──────┐
     │             │
   nobody      real consumer
     │             │
     ▼             ▼
Is audit, replay   Does async
or source-of-truth communication
the purpose?       help here?
     │             │
  ┌──┴──┐       ┌──┴──┐
  │     │       │     │
 no    yes     no    yes
  │     │       │     │
  ▼     ▼       ▼     ▼
don't  persist  direct publish
emit   with a   flow   event
       clear
       reason
```

The key is that the event should exist because of a responsibility, not because the infrastructure already exists.

---

## A more useful question than "is this event used?"

The question I want to ask in future reviews is:

> If I remove this event, which system capability disappears?

Possible good answers:

- customer notifications stop;
- a projection stops updating;
- reconciliation loses its input;
- analytics loses an important fact;
- audit history becomes incomplete;
- state can no longer be rebuilt;
- another service loses a contract it depends on.

A suspicious answer is:

> Nothing, but we have always published it.

That is usually enough to investigate further.

---

## Reusable principles

### Events are not free abstractions

They may be cheap to publish in code while still adding persistence, infrastructure and conceptual cost.

### Durable events need durable reasons

A long-lived event should usually correspond to a long-lived system responsibility.

### "No subscribers" is a smell, not proof

Audit, replay, analytics, external consumers and event sourcing can all justify subscriber-less events.

### The row and the event should not accidentally do the same job

Multiple representations are useful when their responsibilities differ.

### Event stores and event sourcing are different concepts

Persisting events does not make them authoritative.

### Optionality should be priced

Keeping infrastructure "for later" means paying for it now.

### Simplification is architecture

Removing a component whose responsibility disappeared is a design improvement, not merely cleanup.

---

## Questions I should be able to answer

1. What concrete capabilities can justify a durable event?
2. When can an event with no local subscribers still be valid?
3. What is write amplification in an event-driven system?
4. Why can persisting a row and publishing an event be two valid, distinct responsibilities?
5. What is the difference between an event store and event sourcing?
6. Why is a technical event not automatically a domain event?
7. What dependencies should be checked before deleting an apparently dead event?
8. How does future optionality create present operational and cognitive cost?
9. When is observability a better fit than a durable domain event?
10. Why can removing an unused event be a system design decision?

---

## Related concepts

- [[Domain Events vs Commands]]
- [[Event-driven architecture]]
- [[Write amplification]]
- [[Event sourcing]]
- [[Observability]]
- [[Accidental complexity]]
