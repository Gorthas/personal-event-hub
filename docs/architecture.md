# Architecture

## Status

This document describes the approved v1 architecture. The design is complete; the implementation has not started.

## System context

Personal Event Hub sits between external event producers and a single personal notification channel.

```text
External Source                    Owner
      |                              |
      | Submit Event                 | Manage Sources / inspect History
      v                              v
                HTTP boundary
                      |
                      v
                 Coordinator
                  /        \
                 v          v
              Storage    Telegram boundary
                 |          |
                 v          v
             PostgreSQL  Telegram Bot API
```

GitHub Actions is the first intended Source client. It sends the Hub's generic event contract rather than an unmodified GitHub-specific payload.

## Architectural goals

The v1 design prioritizes:

- an explicit acceptance boundary;
- durable Event history before external delivery;
- separation of Hub rules from HTTP, PostgreSQL, and Telegram details;
- small, testable dependencies;
- honest failure reporting;
- a deliberately narrow first release.

The design does not attempt to establish a general event platform, multi-channel router, or distributed messaging system.

## Runtime responsibilities

### HTTP boundary

The HTTP boundary knows how to:

- receive and decode HTTP requests;
- extract authentication material;
- reject malformed transport input;
- pass transport-neutral input to application logic;
- map application results and errors back to HTTP responses.

It does not own operation ordering, PostgreSQL access, Telegram calls, or the rule that persistence defines acceptance.

### Coordinator

The Coordinator owns the rules of Personal Event Hub operations. For Event submission it coordinates:

```text
authenticate
-> validate
-> construct Event
-> persist Event
-> attempt Telegram delivery
-> construct Delivery
-> persist Delivery if possible
```

It decides when the flow stops and which downstream failures do or do not affect Event acceptance. It depends on storage and Telegram capabilities without owning their technical details.

### Storage boundary

Storage knows how to read and persist Source, Event, and Delivery data in PostgreSQL. It provides the data operations required by Management, Ingestion, and History flows.

Storage does not authenticate a caller, decide whether a payload is a valid Event, call Telegram, or define the meaning of a user operation. The Coordinator owns those application decisions; Storage owns their persistent execution.

### Telegram boundary

The Telegram boundary:

- receives an accepted Event;
- formats a human-readable Telegram message from Source name, title, message, and occurrence time;
- uses the configured bot credential and single target;
- makes one delivery attempt;
- translates Telegram or network outcomes into a confirmed success or a known error.

It does not persist Delivery records, access Source credentials, return HTTP responses, or decide whether an Event is accepted.

### Startup / Composition

Startup / Composition is allowed to know the concrete application components. It:

1. reads and validates external configuration;
2. prepares the PostgreSQL connection and Storage component;
3. prepares the Telegram component;
4. creates the Coordinator with its dependencies;
5. creates the HTTP boundary with the Coordinator;
6. registers transport routes and starts the server.

The exact Go package layout and dependency abstraction mechanism are implementation decisions, not part of the B00 design.

## Startup flow and runtime flow

These are two different flows.

### Startup flow

```text
configuration
    -> PostgreSQL / Storage
    -> Telegram boundary
    -> Coordinator
    -> HTTP server
    -> ready
```

Required configuration includes PostgreSQL connection settings, Telegram bot credential, Telegram target, HTTP/server settings, and the Owner credential. The application validates required configuration before declaring itself ready. A mandatory network probe to Telegram at every startup is not part of the approved design.

### Runtime flow

```text
request
   -> HTTP
   -> Coordinator
      -> Storage
      -> Telegram
```

Storage and Telegram do not call each other. HTTP does not coordinate the business flow. Startup connects the components but does not process Events.

## Data ownership

The Hub owns three domain records:

```text
Source 1 -> N Event
Event  1 -> 0..1 Delivery
```

PostgreSQL is the durable system of record for these entities. Telegram is an external delivery dependency, not a source of truth for Event acceptance.

## Key architectural decisions

### Persist before delivery

Telegram is called only after Event persistence succeeds. This ensures the Hub does not send a notification for an Event it failed to accept.

### Acceptance is independent of delivery

After the Event is persisted, Telegram failure or Delivery-persistence failure does not revoke acceptance.

### One synchronous best-effort attempt

v1 performs one Telegram attempt in the request flow. The Source therefore waits for that attempt to finish, even though the response communicates only Event acceptance. This is an acknowledged v1 limitation rather than an eventual-delivery guarantee.

### One global Telegram target

Telegram connection details are application configuration. There is no Destination entity, Route entity, routing policy, or runtime Telegram-configuration API in v1.

### Hard deletion

Deleting a Source removes the Source, all of its Events, and their Deliveries. The design does not retain a hidden archive or soft-deleted copy. Export is a separate read operation available before deletion.

## Rationale for the boundary split

The split follows distinct kinds of knowledge already required by the flow:

| Boundary | Knowledge it owns |
|---|---|
| HTTP | HTTP requests, responses, and transport decoding |
| Coordinator | Hub rules and operation order |
| Storage | Persistent data and PostgreSQL interaction |
| Telegram | Telegram formatting and API interaction |
| Startup / Composition | Concrete construction and wiring |

No named architecture framework is required to justify these boundaries. They follow directly from the responsibilities of the approved v1 flow.

## Deliberate limitations

v1 has no queue, retry worker, recovery process, idempotency key, deduplication mechanism, strict ordering guarantee, or multiple delivery targets. See [Failure Model](failure-model.md) for the consequences.
