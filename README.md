# Personal Event Hub

[Русская версия](README.ru.md)

> **Status: In Progress** — the architecture and v1 design are complete. Implementation has not started yet.

Personal Event Hub is a planned Go backend service for accepting generic events from registered external sources, preserving their history in PostgreSQL, and sending human-readable notifications to a single Telegram target.

The initial real-world producer is GitHub Actions, but the Hub owns a generic event contract rather than a GitHub-specific payload.

## Why this project exists

Automation systems produce useful events, but those events are often tied to individual tools and have no shared history. Personal Event Hub is designed as a small, explicit boundary between event producers and a personal notification channel:

```text
GitHub Actions
      |
      v
HTTP Ingestion API
      |
      v
Personal Event Hub -----> PostgreSQL
      |
      v
Telegram
```

The design deliberately starts with one producer pattern, one delivery channel, and clear failure semantics. It does not present a message broker or a distributed system as a requirement for v1.

## v1 scope

- Register and list event Sources.
- Give each Source its own credential.
- Accept a generic Event containing `title`, `message`, and `occurred_at`.
- Persist an Event before acknowledging it as accepted.
- Make one synchronous, best-effort Telegram delivery attempt.
- Record the delivery result when possible.
- Expose owner-facing history with `success`, `failed`, and `not recorded` views.
- Rotate a Source credential without losing the Source or its history.
- Export the history of a Source.
- Hard-delete a Source together with all of its Events and Deliveries.

## Core model

```text
Source 1 ---- N Event
                  |
                  | 1
                  |
                  0..1 Delivery
```

- **Source** — a registered external producer with a unique name and its own authentication credential.
- **Event** — an immutable fact accepted from an authenticated Source and stored by the Hub.
- **Delivery** — the recorded outcome of the single Telegram attempt for an Event.

`Destination` and `Route` are intentionally absent from v1. All Events use one Telegram target supplied through application configuration.

## Acceptance and delivery contract

```text
authenticate Source
        -> validate input
        -> construct Event
        -> persist Event
        -> attempt Telegram delivery once
        -> persist Delivery if possible
        -> return Event acceptance result
```

An Event is **accepted only after Event persistence succeeds**. A later Telegram or Delivery-recording failure does not revoke that acceptance.

The Source receives the result of its own contract with the Hub: whether the Event was accepted. Telegram details remain internal and are available through owner-facing history.

History derives delivery state as follows:

| Stored state | History state |
|---|---|
| Delivery with `success` | `success` |
| Delivery with `failed` | `failed` |
| No Delivery record for a persisted Event | `not recorded` |

`not recorded` is not a third Delivery result. It means the Hub has an Event but no persisted Delivery outcome.

## Architecture

The design separates responsibilities without committing prematurely to a framework or package layout:

- **HTTP** translates requests and responses at the transport boundary.
- **Coordinator** owns authentication, validation, and operation ordering.
- **Storage** reads and writes persistent data in PostgreSQL.
- **Telegram boundary** formats an Event for Telegram and performs one attempt.
- **Startup / Composition** validates configuration, creates dependencies, connects them, and starts the service.

See [Architecture](docs/architecture.md) and [Event Flow](docs/event-flow.md) for the full rationale.

## Security model

- Every Source has a separate credential used only for event ingestion.
- Management and History operations use a separate Owner credential.
- A new Source credential is disclosed only when it is created or rotated.
- Rotation invalidates the old Source credential while preserving the Source and its history.
- The Telegram bot credential, Telegram target, PostgreSQL connection settings, and Owner credential belong to external application configuration.
- Secrets must not be committed or emitted into logs; public deployment requires HTTPS.

The exact credential generation, verification, and storage mechanisms are intentionally deferred to the Security implementation work rather than claimed as already implemented.

## v1 guarantees and limitations

The v1 design guarantees that Telegram is not called before an Event is persisted, and that a confirmed Event-persistence failure is never reported as accepted.

It deliberately does **not** provide:

- automatic retries;
- recovery of an interrupted delivery flow;
- idempotency or deduplication;
- eventual delivery;
- strict event or notification ordering;
- multiple Telegram targets or routing;
- manual Delivery retry;
- a history of multiple delivery attempts.

These constraints create known ambiguity around lost responses, timeouts, and process crashes. They are documented explicitly in [Failure Model](docs/failure-model.md).

## Planned technology boundaries

- Go
- HTTP/REST
- PostgreSQL
- Database migrations
- Telegram Bot API

This list describes the approved v1 direction, not completed implementation.

## Documentation

- [Architecture](docs/architecture.md) — system boundaries and responsibility split
- [Domain Model](docs/domain-model.md) — Source, Event, Delivery, relationships, and lifecycles
- [Event Flow](docs/event-flow.md) — acceptance point and delivery sequence
- [API Boundaries](docs/api-boundaries.md) — semantic operations without premature endpoint design
- [Security](docs/security.md) — credentials, trust boundaries, rotation, and secret handling
- [Failure Model](docs/failure-model.md) — guarantees, ambiguous outcomes, and known gaps
- [Implementation Plan](docs/implementation-plan.md) — ordered verticals and learning dependencies

## Roadmap

- [x] Architecture and v1 design
- [ ] Foundation
- [ ] Source onboarding
- [ ] Event acceptance
- [ ] Telegram delivery
- [ ] History
- [ ] Credential lifecycle
- [ ] Export and hard delete
- [ ] Reliability pass
- [ ] Testing and v1 completion
