# Sync Extension (Timbal Extension)

**tl;dr** — Defines control frames for synchronizing message history and managing stream subscriptions. Clients send `sync` to request history (with optional timestamp cursor for partial sync), and `unsub` to stop receiving frames for a stream. This is the standard pattern for reconnection and stream lifecycle management.

## Overview

This extension defines two control frame types:

| Control type | Direction | Purpose |
|-------------|-----------|---------|
| `sync` | Client → Server | Request message history, subscribe to a stream |
| `unsub` | Client → Server | Unsubscribe from a stream |

These build on the core Timbal framing spec (§5.2 Control Frames, §8 Stream Multiplexing).

## Control Frames

### `sync` — Request History / Subscribe

Clients send a `sync` control frame to request message history. On multiplexed connections, this also subscribes the client to the stream.

```
{"c":"sync"}
{"c":"sync","since":"2025-01-15T14:30:00.000Z"}
{"c":"sync","s":"chat-general"}
{"c":"sync","s":"chat-general","since":"2025-01-15T14:30:00.000Z"}
```

**Fields:**

* `c` (required): Must be `"sync"`
* `s` (optional): Stream ID to sync. On multiplexed connections, this subscribes the client to the stream.
* `since` (optional): ISO 8601 UTC timestamp. Server sends only messages with `t` greater than or equal to this value.

**Server behavior:**

1. If `s` is provided, subscribe the client to that stream (if not already subscribed).
2. Query messages for the stream (or default stream), filtered by `since` if provided.
3. Send a set frame (`{"i": <ulid>, "t": <timestamp>, "v": <value>}`) for each completed message. On multiplexed connections, include `s` on each frame.
4. If a message is currently streaming, the client will receive it via live append frames followed by a final set frame.

**Partial sync with `since`:**

Using `>=` comparison ensures messages aren't missed when multiple messages share the same timestamp (common at millisecond granularity during batch operations). This may result in duplicates, which are safe — receivers overwrite by ULID.

**Idempotency:** Subscribing to an already-subscribed stream re-syncs without duplicating the subscription.

**Retention:** Servers are not required to retain message history indefinitely. If `since` refers to a time beyond the server's retention window, the server MUST return all available messages. A server with zero retention (always returning all messages) is compliant.

### `unsub` — Unsubscribe from Stream

Clients send an `unsub` control frame to stop receiving frames for a stream.

```
{"c":"unsub","s":"chat-general"}
```

**Fields:**

* `c` (required): Must be `"unsub"`
* `s` (required): Stream ID to unsubscribe from.

**Server behavior:**

* Stop sending frames for that stream to this client.
* If the client is not subscribed to the stream, ignore silently.

**Client behavior:**

* The client SHOULD discard local state for the stream (buffers, values).

## Usage Patterns

### Initial Connection

Client connects and requests full history:

```
Client → {"c":"sync"}
Server → {"i":"01J...001","t":"...","v":{"type":"user","content":"Hello"}}
Server → {"i":"01J...002","t":"...","v":{"type":"agent","content":"Hi there!"}}
Server → {"i":"01J...003","m":{"type":"agent"}}  ← currently streaming
Server → {"i":"01J...003","a":"Let me check"}
Server → {"i":"01J...003","a":" the forecast..."}
```

### Reconnection with Partial Sync

Client disconnects at `2025-01-15T14:30:00.000Z`. On reconnect, it sends the highest `t` value it saw:

```
Client → {"c":"sync","since":"2025-01-15T14:30:00.000Z"}
Server → {"i":"01J...006","t":"2025-01-15T14:30:05.000Z","v":{"type":"user","content":"What's the weather?"}}
Server → {"i":"01J...007","t":"2025-01-15T14:30:10.000Z","v":{"type":"agent","content":"It's sunny today!"}}
```

Duplicates are harmless — just overwrite by ULID.

### Multiplexed Stream Management

Client subscribes to multiple streams, then switches focus:

```
Client → {"c":"sync","s":"chat-general"}
Client → {"c":"sync","s":"announcements"}

Server → {"s":"chat-general","i":"01J...001","t":"...","v":{"type":"user","content":"Hey everyone"}}
Server → {"s":"announcements","i":"01J...002","t":"...","v":{"type":"agent","content":"System update: v2.1"}}
Server → {"s":"chat-general","i":"01J...003","m":{"type":"agent"}}
Server → {"s":"chat-general","i":"01J...003","a":"Hi there!"}

Client → {"c":"unsub","s":"chat-general"}
Client → {"c":"sync","s":"chat-random"}

Server → {"s":"chat-random","i":"01J...010","t":"...","v":{"type":"user","content":"Anyone seen the new release?"}}
Server → {"s":"announcements","i":"01J...011","t":"...","v":{"type":"agent","content":"Maintenance at 2am UTC"}}
```

No race condition — frames are self-describing via `s`. The client never needs to track "which stream am I on."

### Late-Join Support

For servers implementing this extension, late-join requires:

1. **Always finalize with a set frame**: When streaming completes for any message, send a final `{"i": <ulid>, "t": <timestamp>, "v": <value>}` frame. This ensures late-joiners receive the complete value.
2. **Track timestamps**: Clients should store the highest `t` value seen for use in future `since` parameters.
3. **Ignore orphaned appends**: Append frames for unknown message IDs are safely ignored (per core spec §5.1.2).

## Implementation Notes

This extension is designed for the common case of WebSocket-based real-time applications. Other sync patterns are equally valid:

* **REST-based sync**: Fetch history via HTTP GET, use WebSocket only for live frames.
* **Cursor-based pagination**: Use message IDs instead of timestamps for pagination.
* **Server-push only**: Server decides what to send; no client-initiated sync.
* **GraphQL subscriptions**: Use GraphQL for stream management, Timbal for the wire format.

The core Timbal framing spec works with any of these patterns. This extension documents one well-tested approach.

---

**Status**: Draft
