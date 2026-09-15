---
title: Domain events vs commands in asynchronous workflows
tags:
  - asynchrony
  - queues
  - publish-subscriber
---

A queue answers **when** work runs.

It does not answer **what the message means**.

That distinction matters because two pieces of asynchronous work can look almost identical operationally while representing very different ideas:

```text
PaymentPending
```

means:

> A fact has occurred.

while:

```text
ProcessPayment
```

means:

> Perform this operation.

The first is an **event**. The second is a **command**.

This note explores when each model is useful, how they affect coupling and operability, and why asynchronous workflows also force us to think about delivery guarantees, duplicate processing, and the dual-write problem.

---

## Starting point

Consider a system with an internal event publisher.

A business operation produces a domain event:

```text
PaymentPending
```

The publisher persists that event, finds subscribers interested in it, and schedules asynchronous work for each one:

```text
Business operation
       │
       ▼
 PaymentPending
       │
       ▼
 Event publisher
       │
   ┌───┴────┐
   ▼        ▼
Consumer A Consumer B
   │        │
   ▼        ▼
 async     async
 work      work
```

One possible consumer might eventually execute the real operation:

```text
PaymentPending
       │
       ▼
ProcessPaymentSubscriber
       │
       ▼
process payment
```

This creates an architectural question that is independent of any job library or framework:

> Is `PaymentPending` genuinely a domain fact that independent consumers should react to, or are we using an event as an indirect way to say `ProcessPayment`?

---

# Events and commands have different semantics

## Event: something happened

An event describes a fact that has already happened.

It is naturally named in the past tense:

```text
PaymentCreated
PaymentPending
PaymentSucceeded
TransferRejected
CustomerVerified
```

Its meaning is:

> This happened.

The producer should not need to know every consumer interested in that fact.

```text
              PaymentSucceeded
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
      Analytics   Receipt   Accounting
```

The event remains meaningful even if one of those consumers disappears or a new one is added.

---

## Command: do something

A command expresses an intention.

It is naturally named as an imperative:

```text
ProcessPayment
SendTransfer
GenerateInvoice
RetryWebhook
VerifyCustomer
```

Its meaning is:

> Do this.

A command normally has one logical owner:

```text
ProcessPayment
      │
      ▼
Payment processor
```

It may succeed, fail, be rejected, or be retried.

That is a different relationship from publishing a fact for zero or more interested observers.

---

# Jobs are an execution mechanism, not a semantic model

It is easy to collapse these concepts into framework terminology:

> events vs jobs

But that mixes two separate decisions.

The semantic question is:

```text
EVENT
"PaymentSucceeded"
```

versus:

```text
COMMAND
"SendReceipt"
```

The execution question is:

```text
synchronous call
background job
message queue
broker
HTTP
RPC
```

A command can be executed by a background job.

An event handler can also be executed by a background job.

So the useful distinction is:

> **event vs command**, not **event vs job**.

---

# When events are valuable

Events are particularly useful when the producer should publish a fact without owning every consequence.

## Natural fan-out

Suppose a successful payment has several independent consequences:

```text
PaymentSucceeded
      │
      ├──► notify customer
      ├──► update analytics
      ├──► trigger reconciliation
      └──► update rewards
```

The payment workflow should not necessarily know about all of them.

An event allows those consumers to evolve independently.

---

## Extensibility without changing the producer

A new reaction can be added later:

```text
PaymentSucceeded
      │
      └──► FraudLearningConsumer
```

without modifying the code that completes the payment.

That is useful when new consumers are a realistic part of the system's evolution.

---

## Domain vocabulary

Events can make important business transitions explicit:

```text
TransferSubmitted
TransferSettled
TransferRejected
```

This is more than an implementation detail if those facts matter to the domain.

They can also become useful boundaries for:

- audit trails;
- external integrations;
- analytics;
- notifications;
- cross-context communication.

---

# The cost of events

Event-driven code often improves decoupling, but it also introduces **indirection**.

Consider:

```text
publish(PaymentPending)
```

Reading that line does not tell us what happens next.

The actual flow may be:

```text
request
  ↓
domain operation
  ↓
event
  ↓
publisher
  ↓
subscriber registry
  ↓
job
  ↓
subscriber
  ↓
service
```

Every layer may make sense locally, while the whole path becomes harder to follow.

That cost is justified only if the abstraction gives us something valuable in return.

---

## Hidden mandatory dependencies

Suppose this is always true:

```text
PaymentPending
       │
       ▼
ProcessPaymentSubscriber
```

and processing the payment is mandatory for the workflow to continue.

The code is syntactically decoupled, but semantically the dependency is still there.

The system cannot really say:

> "A payment became pending; maybe somebody will react."

What it means is:

> "The next required operation is to process this payment."

In that case the event bus may be hiding a workflow dependency rather than removing one.

---

# When commands are a better fit

Commands work well when the system knows exactly which operation must happen next.

```text
CreatePayment
     │
     ▼
ValidatePayment
     │
     ▼
ProcessPayment
     │
     ▼
SubmitTransfer
```

This makes the workflow explicit.

Benefits include:

- easier code navigation;
- clearer ownership;
- simpler debugging;
- more obvious retry semantics;
- fewer layers of indirection.

If `ProcessPayment` has one logical owner, modelling it directly is usually easier to understand than introducing an event whose only purpose is to reach that owner.

---

# Orchestration vs choreography

This distinction is closely related to two ways of coordinating distributed workflows.

## Orchestration

A component explicitly coordinates the steps:

```text
          Orchestrator
               │
      ┌────────┼────────┐
      ▼        ▼        ▼
 Command A  Command B  Command C
```

The benefit is visibility.

The cost is that the orchestrator knows more about the workflow.

---

## Choreography

Components react to events:

```text
Event A
   │
   ├──► Consumer B
   │         │
   │         ▼
   │      Event B
   │         │
   │         ▼
   │      Consumer D
   │
   └──► Consumer C
```

The benefit is looser coupling between participants.

The cost is that the global workflow becomes emergent.

There may be no single place where an engineer can see the whole process.

A useful rule is:

> The more important it is to understand who must do what and in which order, the more attractive explicit orchestration becomes.

> The more important it is for independent capabilities to react to a fact without central coordination, the more attractive choreography becomes.

---

# A practical hybrid

Events and commands do not need to compete for control of the whole architecture.

A useful model is:

```text
              BUSINESS WORKFLOW

CreatePayment
     │
     ▼
ValidatePayment
     │
     ▼
ProcessPayment
     │
     ▼
Payment succeeds
     │
     ▼
PaymentSucceeded
   /      |       \
  ▼       ▼        ▼
Receipt Analytics Accounting
```

Commands drive the workflow:

```text
CreatePayment
ValidatePayment
ProcessPayment
```

The event announces the fact:

```text
PaymentSucceeded
```

Independent capabilities then react to it.

This keeps mandatory workflow steps explicit while preserving event-driven fan-out where it provides real value.

---

# Failure modes matter more than the syntax

Once work becomes asynchronous, the architecture must assume partial failure.

The interesting questions are no longer only:

> Who calls whom?

They become:

> What happens when the process dies between two steps?

> Can the same message be delivered twice?

> Can messages arrive out of order?

> How do we know what happened?

---

## Lost work

Consider:

```text
update business state   ✓
enqueue work            ✗
```

The database says the entity moved forward, but the required asynchronous work will never run.

The opposite order also has problems:

```text
enqueue work            ✓
commit business state   ✗
```

A worker may observe state that never became authoritative.

---

# The dual-write problem

A common publisher looks conceptually like this:

```text
store_event(event)
enqueue_job(event)
```

Those are two writes.

If they are not atomic, the process can fail between them:

```text
event persisted   ✓
job enqueued      ✗
```

The same problem appears one level earlier:

```text
update payment state
publish event
```

If the state change commits but event publication fails:

```text
payment = pending   ✓
PaymentPending      ✗
```

the system has two pieces of information that should agree but no longer do.

This is the classic **dual-write problem**.

---

# Why a normal transaction is not enough

A local database transaction can safely combine operations that happen in the same transactional database:

```text
BEGIN

UPDATE payments ...
INSERT domain_events ...

COMMIT
```

Either both commit or neither does.

But the database normally cannot atomically commit together with a separate queue or broker:

```text
COMMIT database
AND
publish message
```

They are separate systems with separate failure boundaries.

This is where the **transactional outbox** becomes useful.

---

# Transactional outbox

Instead of trying to atomically perform:

```text
business change
+
message publication
```

we atomically perform:

```text
business change
+
durable intent to publish
```

inside the same database transaction.

```text
BEGIN

UPDATE payments
SET state = 'pending'
WHERE id = 123;

INSERT INTO outbox
  (message_id, message_type, payload)
VALUES
  (...);

COMMIT
```

Now the business state and the publication intent either both exist or neither does.

A separate dispatcher publishes pending outbox records:

```text
┌──────────────────────────┐
│ Database                 │
│                          │
│ business state           │
│ outbox                   │
└────────────┬─────────────┘
             │
             ▼
      Outbox dispatcher
             │
             ▼
        Queue / broker
             │
             ▼
          Consumer
```

If the application crashes after the commit, the outbox record remains available for later publication.

That closes the dangerous gap between committing the business change and remembering that a message must be sent.

---

# The outbox does not give exactly-once processing

Suppose the dispatcher does this:

```text
publish message          ✓
mark outbox as published ✗
                         ↑
                       crash
```

After restarting, the dispatcher still sees the record as unpublished.

So it publishes it again.

```text
message 42
message 42
```

This is why realistic messaging systems often aim for:

```text
at-least-once delivery
+
idempotent processing
```

rather than assuming that duplicate delivery can never happen.

---

# Idempotency

An idempotent consumer can safely process the same logical message more than once.

One approach is deduplication:

```text
message_id = abc-123

if already_processed?(message_id)
  ignore
else
  process
  mark_as_processed
end
```

A more domain-oriented approach is to make the state transition itself safe:

```text
pending → processing
```

may be accepted only once.

The correct strategy depends on the side effect.

Repeating:

```text
set status = processing
```

is very different from repeating:

```text
charge customer
```

External side effects often require explicit idempotency keys or equivalent guarantees from the downstream system.

---

# Fan-out creates independent delivery state

For an event with several consumers:

```text
PaymentSucceeded
   │
   ├── Consumer A ✓
   ├── Consumer B ✗
   └── Consumer C ✓
```

there is no single meaningful answer to:

> Has the event been processed?

Each consumer has its own delivery state.

That means retries and observability should usually be consumer-specific.

One failed consumer should not require replaying successful consumers unless replay is explicitly safe.

---

# Ordering

Asynchronous systems should not assume global ordering unless they deliberately provide it.

The producer may emit:

```text
PaymentCreated
PaymentCancelled
```

but consumers can observe:

```text
PaymentCancelled
PaymentCreated
```

because of:

- retries;
- multiple workers;
- independent queues;
- network delays;
- partitioning.

The first question should be:

> What ordering do we actually need?

Often the requirement is not global ordering but ordering **per aggregate**:

```text
payment-123:
  version 10
  version 11
  version 12
```

Possible techniques include:

- partitioning by aggregate ID;
- sequence numbers;
- aggregate versions;
- state validation;
- optimistic locking.

Ordering is a business invariant, not something to assume because a queue happens to look FIFO in the happy path.

---

# Retries and poison messages

Retries are necessary for transient failures, but they are not free.

A permanently invalid message can become a poison message:

```text
attempt 1 ✗
attempt 2 ✗
attempt 3 ✗
...
```

Systems need an explicit policy:

- retry limits;
- exponential backoff;
- jitter;
- dead-letter handling;
- manual recovery;
- alerting.

A dependency outage can also produce a retry storm:

```text
dependency fails
      │
      ▼
50,000 jobs fail
      │
      ▼
50,000 jobs retry together
```

Backoff, jitter, concurrency limits, and circuit breaking can prevent recovery traffic from becoming a second incident.

---

# Observability and causality

Asynchronous systems need enough metadata to reconstruct why work exists.

Useful identifiers include:

```text
message_id
command_id / event_id
aggregate_id
correlation_id
causation_id
job_id
consumer
attempt
created_at
processed_at
```

A useful causal chain might look like:

```text
HTTP request
correlation_id = C1
       │
       ▼
Payment created
       │
       ▼
PaymentPending
event_id = E1
correlation_id = C1
       │
       ▼
ProcessPayment
command_id = CMD1
causation_id = E1
       │
       ▼
worker attempt #2
```

This lets an engineer answer:

> Why did this job run?

> Which user operation eventually caused it?

> Which attempt failed?

Without that context, asynchronous debugging often becomes archaeology.

---

# A decision checklist

For each asynchronous message, ask:

### Semantics

Does the message mean:

```text
something happened
```

or:

```text
do something
```

Would its name naturally be past tense or imperative?

---

### Workflow

Is this reaction mandatory for the current operation to complete?

If yes, a command often makes the dependency clearer.

---

### Consumers

Are there **real** independent consumers today?

Not hypothetical future consumers.

Would adding a new consumer without changing the producer be genuinely valuable?

---

### Ownership

Is there one clear component responsible for performing the operation?

If yes, that points towards a command.

Are several independent capabilities interested in the same fact?

That points towards an event.

---

### Operability

Can an engineer trace the flow from the original action to the final effect?

Can production tooling answer:

```text
what happened?
what message was created?
who consumed it?
how many attempts were made?
what will retry?
```

---

### Reliability

What happens if the process dies between:

```text
database commit
```

and:

```text
enqueue / publish
```

Are consumers safe under duplicate delivery?

Does ordering matter?

If so, at what scope?

---

# Do not pay for hypothetical extensibility forever

One common justification for event-driven abstractions is:

> We may add more subscribers later.

That is possible, but flexibility has a cost.

If a system spends years with:

```text
1 producer
1 event
1 subscriber
1 required operation
```

then it may simply have built a very indirect function call.

A useful design principle is:

> **Do not permanently pay the complexity cost of extensibility that exists only in theory.**

Architecture should respond to plausible evolution, not every imaginable one.

---

# A useful mental model

Separate three questions:

```text
SEMANTICS
event or command?
      │
      ▼
What does this message mean?


RELIABILITY
outbox, retries, idempotency, ordering
      │
      ▼
What happens when things fail?


TRANSPORT
queue, broker, background jobs, HTTP
      │
      ▼
How does the message move or execute?
```

Mixing these layers makes architectural discussions much harder.

Changing a queue implementation should not force the system to change what its messages mean.

Likewise, choosing events does not remove the need to design delivery guarantees.

---

# Key takeaways

**Events describe facts. Commands express intentions.**

Use events when independent consumers genuinely care about a meaningful domain fact.

Use commands when a specific operation must be performed by a clear owner.

**Decoupling and indirection are not the same thing.**

An event bus can remove useful dependencies, but it can also hide mandatory workflow dependencies behind extra layers.

**Commands are often better for explicit workflows; events are often better for independent reactions.**

A hybrid design is usually more useful than forcing the entire system into one paradigm.

**Two related writes create a failure boundary.**

Whenever the design contains:

```text
write A
write B
```

ask what happens if the process dies between them.

**Transactional outbox makes publication intent durable with the business transaction.**

It does not make the whole system exactly-once.

**At-least-once delivery implies duplicate-aware consumers.**

Idempotency is not an optimisation. In many asynchronous systems it is part of correctness.

**Ordering must be designed at the scope where the business actually needs it.**

Global ordering is expensive and often unnecessary.

**Every abstraction should pay its rent.**

If:

```text
A → event → publisher → registry → job → subscriber → B
```

does not provide meaningful fan-out, independence, auditability, or domain clarity, then:

```text
A → command → B
```

may be the better design.
