# Event Flow

## Status

This document defines the approved v1 processing semantics. It does not define endpoint paths, HTTP status codes, or wire-format details.

## Happy path

```text
GitHub Actions or another registered Source
        |
        | generic Event input
        v
1. HTTP request decoding
        |
        v
2. Source identification and authentication
        |
        v
3. Event validation
        |
        v
4. Internal Event construction
        |
        v
5. Event persistence in PostgreSQL
        |                           ACCEPTANCE POINT
        v
6. Telegram message formatting
        |
        v
7. One synchronous Telegram attempt
        |
        v
8. Delivery construction and persistence
        |
        v
9. HTTP response: Event accepted
```

The response communicates only whether the Hub accepted the Event. It does not expose Telegram results to the Source.

## Step-by-step semantics

| Step | Hub responsibility | Result | Stop or continue on failure |
|---|---|---|---|
| HTTP decoding | Read authentication material and decode the input | Transport-neutral input | Stop; nothing is persisted |
| Source authentication | Resolve and verify one registered Source | Authenticated Source | Stop; no Event and no Telegram call |
| Event validation | Check required fields and timestamp structure | Validated input | Stop; no Event and no Telegram call |
| Event construction | Combine input, authenticated Source, Event identity, and `received_at` | Internal Event | Stop; Event is not accepted |
| Event persistence | Store Event and its Source relationship | Persisted Event | Stop; report not accepted; do not call Telegram |
| Telegram formatting | Build a human-readable message | Telegram representation | Event remains accepted; delivery fails |
| Telegram attempt | Perform one synchronous best-effort call | Confirmed success or known failure | Event remains accepted; no retry |
| Delivery persistence | Store `success` or `failed`, attempt time, and optional error | Delivery record | Event remains accepted; History may show `not recorded` |
| HTTP response | Report Event acceptance | Success with Event identity, or an acceptance error | Delivery details are not returned |

## Input contract

The Source supplies:

- `title`;
- `message`;
- `occurred_at`.

All are required. `title` and `message` must be non-empty, and `occurred_at` must use an accepted timestamp representation.

The Source does not supply:

- Source identity as trusted payload data;
- Event identity;
- `received_at`;
- delivery state;
- Telegram data.

Source identity comes from authentication. Event identity and `received_at` are created by the Hub. Delivery belongs to downstream processing.

## The acceptance point

The central v1 contract is:

```text
valid request != accepted Event
persisted Event = accepted Event
```

Authentication success and validation success are necessary but not sufficient. The Hub reports acceptance only after Event persistence succeeds.

Once persistence succeeds:

- Telegram failure cannot revoke acceptance;
- failure to persist Delivery cannot revoke acceptance;
- the Event remains visible in history;
- a missing Delivery is represented as `not recorded`.

## Delivery semantics

v1 performs exactly one Telegram attempt for each normally processed accepted Event. The attempt is synchronous and best-effort:

- confirmed Telegram success becomes `Delivery(success)`;
- an explicit Telegram error, timeout, or other unconfirmed outcome becomes `Delivery(failed)` when that result can be persisted;
- the Hub does not retry automatically;
- the Hub does not promise eventual delivery.

A timeout or lost response is treated conservatively: the Hub cannot confirm success, even though Telegram may have received the message.

## Response boundary

The Source is responsible for delivering an Event to the Hub. It is not responsible for the Hub-to-Telegram operation.

Therefore:

```text
Event was not persisted
-> response means Event not accepted

Event was persisted
-> response means Event accepted
```

Telegram success, Telegram failure, and Delivery-recording failure do not change that response meaning. They are owner-facing diagnostic information exposed through History.

Because delivery is synchronous in v1, the Source still waits for the Telegram attempt even though it does not receive the attempt result. Removing that latency would require a later asynchronous processing design.

## History view

History is ordered by the Hub's `received_at` time and can show:

- `success` — a successful Delivery is stored;
- `failed` — a failed Delivery is stored;
- `not recorded` — the Event exists but no Delivery record exists.

Event details combine the immutable Event data with the known Delivery information, if any.

## Explicit non-guarantees

v1 does not guarantee:

- safe replay of the same submission;
- deduplication;
- recovery after a crash between Event and Delivery processing;
- retry after Telegram failure;
- eventual Telegram delivery;
- strict ordering across Events or notifications.

See [Failure Model](failure-model.md) for the resulting scenarios.
