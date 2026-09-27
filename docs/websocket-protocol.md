# WebSocket Protocol

## Overview

The frontend connects to a WebSocket endpoint defined by `NEXT_PUBLIC_WS_URL`. The generic
`useWebSocket<T>` hook (`src/hooks/useWebSocket.ts`) manages the connection, auto-reconnect, and
JSON parsing. Each frame is expected to be a JSON object matched to the generic type `T`.

**Connection behavior:**

- Reconnect delay: 3 seconds
- Malformed frames are silently ignored
- Passing `null` as the URL tears down the socket and stays idle

## Feeds

Both feeds subscribe to the same `NEXT_PUBLIC_WS_URL` and expect identical JSON shapes.

### Live Intents feed

**Hook:** `src/hooks/useLiveIntents.ts`
**Type:** `FeedItem` from `src/lib/types.ts`
**Max items:** 200

### Intent feed (homepage preview)

**Hook:** `src/hooks/useIntentFeed.ts`
**Type:** `FeedItem` from `src/lib/types.ts`
**Max items:** 8
Seeds from REST `/intents/feed` (`src/hooks/useActivityFeed.ts`) and layers live updates on top.

## `FeedItem` Shape (Canonical)

```json
{
  "id": "string",
  "srcChain": "string",
  "srcToken": "string",
  "srcAmount": "string",
  "dstToken": "string",
  "solver": "string",
  "status": "pending | accepted | filled | failed",
  "createdAt": "ISO-8601 timestamp"
}
```

| Field       | Type     | Notes                                           |
| ----------- | -------- | ----------------------------------------------- |
| `id`        | `string` | Unique intent identifier                        |
| `srcChain`  | `string` | Source chain identifier                         |
| `srcToken`  | `string` | Source asset symbol                             |
| `srcAmount` | `string` | Human-readable amount                           |
| `dstToken`  | `string` | Destination asset symbol                        |
| `solver`    | `string` | Solver name or address                          |
| `status`    | `string` | Enum: `pending`, `accepted`, `filled`, `failed` |
| `createdAt` | `string` | ISO-8601 UTC timestamp                          |

## Accepted Message Shapes & Validation

Every frame is validated by `parseFeedItemFrame`
([`src/lib/realtime/quarantine.ts`](../src/lib/realtime/quarantine.ts)) before it
reaches state. Two shapes are accepted:

```json
{ "id": "i1", "srcChain": "ethereum", "srcToken": "USDC", "srcAmount": "10.5",
  "dstToken": "USDC", "solver": "Alpha", "status": "pending",
  "createdAt": "2026-07-14T00:00:00Z", "version": 3, "updatedAt": "2026-07-14T00:01:00Z" }
```

```json
{ "type": "intent", "data": { "...": "FeedItem as above" } }
```

Rules:

- Frames larger than 64 KB are dropped.
- Envelopes with any `type` other than `"intent"` are ignored silently.
- Keys `__proto__`, `constructor` and `prototype` reject the frame.
- `status` must be one of `pending | accepted | filled | failed`; `srcAmount`
  must be a decimal string (not a number); dates must be valid ISO strings;
  `version`, when present, must be a number.
- Unknown fields are stripped and all string fields pass through
  `sanitizeDisplayText`.
- Rejected frames are counted and the last 20 kept (truncated preview) for
  dev diagnostics (`getQuarantineDiagnostics()`); a rate-limited
  `secureLogger.warn` fires in development only. End users see nothing.

## Reconnect & Backfill (Assumed Relay Contract)

The relay does not replay missed frames. The client assumes:

- A reconnect may have missed any number of frames; after every transition to
  `open` following a disconnect, the REST snapshot for the active view is
  revalidated (deduplicated, at most once per 5 s, ignored after unmount).
- `version` (preferred) or `updatedAt`, when present, increase monotonically per
  intent; older frames are discarded. Without either, arrival order wins but
  status may never regress (`filled → pending` is rejected), which is
  clock-skew safe.
- If the relay later supports `?since=` / cursors, only the delta needs to be
  fetched; the store already merges partial snapshots without clearing rows.

## App-Wide Status-Change Alerts

`useIntentStatusWatcher` (`src/hooks/useIntentStatusWatcher.ts`), mounted via
`IntentStatusWatcher` in `src/app/layout.tsx`, subscribes to the same feed as
`useMyLiveIntents` but skips the REST snapshot fetch — it only diffs
`status` per intent `id` across incoming WebSocket messages, so it stays
cheap to run on every page. On a transition it pushes a toast (batched into
a single "N intents updated" toast when several land within the same
1-second window) linking to the intent, unless the user is already on
`/my-intents` where the transition is visible directly. State resets when
the connected wallet address changes. Browser `Notification` support was
scoped out of the initial pass — see issue #231 — since it requires an
explicit settings toggle to request permission.

## Backend Reference

If the canonical schema lives in [vortex-backend](https://github.com/vortex-protocol/vortex-backend),
link it here and keep the frontend types in sync. If the schema diverges, update the
`FeedItem` type in `src/lib/types.ts` and the corresponding tests in
`src/hooks/useLiveIntents.test.ts` and `src/hooks/useIntentFeed.test.ts`.
