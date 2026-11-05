# Spec — event-inbox (milestone 1)

**Repo:** event-inbox
**One event type:** `device.reading.uploaded`
**Contract:** `POST` the same `event_id` twice → **200 both times, one side effect.**

## Problem

A client POSTs a device reading. Networks fail. The client retries. The inbox must accept the retry and must not apply the reading again.

This is **idempotency**: one `event_id` means one outcome, however many times the HTTP request arrives.

## Endpoint

`POST /events`

No auth in this milestone. Signing, outbound retries, GraphQL, and a UI wait for later repos.

## Allow-list (only these fields)

```json
{
  "event_id": "evt_01JEXAMPLE",
  "type": "device.reading.uploaded",
  "occurred_at": "2026-09-21T14:02:11.000Z",
  "device_id": "dev_thermostat_garage",
  "reading": {
    "value": 21.5,
    "unit": "celsius"
  }
}
```

| Field | Rules |
|---|---|
| `event_id` | required string, non-empty, caller-supplied. This is the idempotency key. You do not mint it. |
| `type` | required. Must be exactly `device.reading.uploaded`. Anything else → 400. |
| `occurred_at` | required ISO-8601 UTC timestamp. |
| `device_id` | required string, non-empty. |
| `reading.value` | required finite number. |
| `reading.unit` | required non-empty string. |

Unknown keys: **ignore**. Do not persist them. Do not spread the body into storage.

`event_id` is **not** `device_id`. Two readings from the same device are two events. The same `event_id` twice is one event, retried.

## Side effect (exactly one)

On first accept, in **one database transaction**:

1. Insert the inbox row keyed by `event_id` (unique).
2. Insert **one** row into `readings` (`event_id`, `device_id`, `value`, `unit`, `occurred_at`).

That insert is the side effect. A test can `COUNT(*)` it.

On a duplicate `event_id`: do not insert a second inbox row, do not insert a second reading, still return 200. Response body matches the first success — no `"duplicate": true`. A retrying client should not have to interpret a special shape.

If the second POST sends a **different** payload with the same `event_id`, do not update the stored reading. First write wins.

## Responses

| Case | Status | Body |
|---|---|---|
| First accept | **200** | `{ "event_id": "...", "status": "accepted" }` |
| Same `event_id` again | **200** | same shape, same `event_id` |
| Missing/invalid allow-listed field, or `type` mismatch | **400** | `{ "error": "invalid_event" }` |
| Crash / DB down | **500** | `{ "error": "internal" }` |

First accept is 200 too (not 201). Match the GitHub description.

## Persistence

SQLite file on disk. Not an in-memory `Map`.

- `inbox_events(event_id PRIMARY KEY, type, received_at, payload_json)`
- `readings(event_id PRIMARY KEY REFERENCES inbox_events, device_id, value, unit, occurred_at)`

The race is closed by the unique constraint, not by check-then-insert. Two concurrent POSTs with the same id: one insert wins, the other hits the unique violation, treat that as already accepted, return 200. Claim + reading insert in one transaction.

## The test that names the failure

**`duplicate delivery does not double-apply a reading`**

1. POST a valid event.
2. POST the same `event_id` again.
3. Both responses are 200.
4. `SELECT COUNT(*) FROM readings WHERE event_id = ?` is **1**.

Also worth a 400 test for a missing `event_id`.

## Out of scope

- HMAC / webhook signatures
- Outbound notify / backoff / dead-letter
- GraphQL
- Status page
- Auth, users, API keys
- Multiple event types
- PHI, print, Stripe field names
