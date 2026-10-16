---
title: "One HTTP Exchange, One Log Event"
description: "Designing structured HTTP client logs without breaking existing integrations."
tags:
  - observability
  - system-design
  - ruby
  - http
  - incremental-rollouts
---

# One HTTP Exchange, One Log Event

An outgoing HTTP call looks like one operation in application code. In traditional HTTP client logs, however, it can look like several unrelated events: request headers on one line, request body on another, response status somewhere else, and response body after that.

That representation is convenient for reading a local console, but less useful when a log pipeline stores **each line as an independent document**. Answering “What happened to this request?” becomes an exercise in reconstructing the exchange.

A better unit of observation is the **HTTP exchange itself**: one event containing its request, response, elapsed time, and any error.

## The event is the unit of correlation

A structured event might look like this (synthetic example):

```json
{
  "request_id": "req-example",
  "http_client": {
    "name": "external-service",
    "method": "POST",
    "url": "https://example.invalid/api/check",
    "status": 200,
    "duration_ms": 84,
    "error_class": null
  }
}
```

The precise schema is less important than its properties: the fields are stable, the event is emitted once, and it represents the whole exchange. Failed connections should not disappear simply because there was no HTTP response. A timeout can be expressed with a null status and an error class; an HTTP 500 is still an HTTP response with a status.

When logs are shipped as line-delimited JSON, **one complete JSON object per line** also makes ingestion and field extraction much more predictable.

## Middleware placement is part of the contract

In Ruby applications using Faraday, it is tempting to replace every `response :logger` with a custom middleware and call the job done. But middleware ordering changes what the logger can see.

Requests travel through the stack towards the adapter; responses travel back through it. A logger outside a JSON parser may observe a parsed response body, while one inside it may observe raw response data. Placement relative to error-raising middleware can also determine whether a response is recorded before an exception propagates.

Those differences matter even if every HTTP request still succeeds.

A safe migration therefore preserves each integration's middleware position rather than forcing every connection builder into a single template. It also preserves options such as header/body logging, logger destination, filters, and conditional registration.

This is an important kind of compatibility: not just **the same HTTP behavior**, but **the same logging semantics until a deliberate change is enabled**.

## Separate collection, representation, and selection

Three responsibilities are useful to keep apart:

**Collection.** A middleware captures request and response information, measures elapsed time with a monotonic clock, and handles exchanges that end in network exceptions. Logging failures should not unexpectedly turn a successful HTTP operation into a failed one.

**Representation.** A formatter builds a consistent structured object. It respects the existing choices about which headers or bodies may be represented and how configured filters are applied.

**Selection.** A small wrapper chooses between the existing logger and the structured logger, using a feature flag evaluated **for each request**.

That last detail matters for long-lived or memoized connections. If the decision is made only when the connection is built, turning a feature flag off may not change anything until processes restart. Per-request selection allows a rollback without rebuilding the client or sending requests twice.

## Extending libraries without coupling them to an application

A related problem appears when an application uses HTTP client gems that build their own Faraday stacks. Copying an application's logging middleware into each gem creates duplicate implementations and forces the libraries to know about application-specific infrastructure.

A smaller interface can be enough:

```ruby
Client.new(
  logger_middleware: CustomLoggingMiddleware,
  logger_options: { name: "external-service" }
)
```

The library still owns its HTTP connection and supplies a backward-compatible default logger. The application provides a middleware implementation and its configuration. Existing users of the library need not change anything.

There is a subtle requirement here: if a library passes a filter block to the middleware, the replacement must actually execute that block. Accepting a block is not the same as preserving the behavior it configures. This belongs in integration tests.

## Roll out vertically, not everywhere at once

Before migrating multiple clients, choose one integration that can be exercised in a controlled environment. Follow the request all the way through the system:

- Did it reach the destination without a behavior change?
- Was exactly one structured event emitted?
- Were the expected fields extracted by the log platform?
- Were the previous log lines absent when the new mode was enabled?
- Did disabling the feature flag restore the legacy logger without a restart?

Once this vertical slice works, other integrations can be migrated in small groups. Straightforward clients can share a pull request; clients with unusual middleware ordering or connection management deserve separate reviews.

Avoid adding a shared configuration helper merely because several builders repeat two or three options. Sometimes that repetition is clearer than an abstraction that hides important ordering differences.

## Structured does not automatically mean safe

A single searchable document can also make sensitive data easier to find. Capturing full request or response bodies should be a deliberate policy choice, not an accidental side effect of structured logging.

Preserve existing filters during a compatibility-focused migration, but do not confuse that with comprehensive redaction. Credential removal, personal-data handling, size limits, and retention deserve explicit requirements and tests of their own.

The goal is not to log everything. It is to make **the information that is appropriate to log** easier to relate and investigate.

## What I would keep from this design

The main lesson is that observability changes are system changes. They cross the HTTP client, middleware stack, application processes, log shipping, indexing, and queries. A clean JSON formatter is only one part of the solution.

A good migration protects the old behavior, proves the new path end to end, and makes rollback cheap. With those constraints in place, moving from many log lines to one event per HTTP exchange becomes a manageable evolution rather than a risky rewrite.
