# Timbal Framing Layer (Timbal/1.0)

**tl;dr** — Timbal defines the framing layer for streaming agent chats over WebSocket/SSE/HTTP. Each message is a sortable ULID and its contents stream progressively as NDJSON frames. Start frames declare metadata upfront, enabling immediate UI rendering. Clients can request partial sync on reconnect. A single connection can multiplex multiple streams via the `s` field.

## 1. Overview

Timbal defines the **framing layer** for streaming chat transcripts as an **append-only sequence of NDJSON frames**. Each frame updates a **message**, identified by a **sortable ULID**, where message order is implied by the ULIDs.

This framing layer sits above the transport layer (WebSocket, SSE, HTTP, stdio) and defines how messages are encoded, streamed, and synchronized.

Each message's **value is always a JSON object**. Messages can be streamed in two modes:

1. **Text streaming with metadata**: Start frame includes metadata (e.g., `{"type": "agent"}`), and appends contain plain text that becomes the `content` field. No JSON escaping needed.
2. **Object streaming**: Start frame has no metadata, and appends contain JSON fragments requiring partial parsing.

The framing layer is transport-agnostic and works over:

* HTTP streaming (chunked responses)
* Server-Sent Events (SSE)
* WebSockets
* stdio pipes

**Stream multiplexing:** A single connection can carry frames for multiple logical streams (channels, threads, event feeds) using the optional `s` field. See §9.

## 2. Key Properties

* **Message ordering is implied by IDs**: ULIDs are lexicographically sortable by creation time, so sorting by `i` yields message order.
* **NDJSON frames**: The wire format is newline-delimited JSON objects.
* **Streaming in groups**: Transports may deliver frames in arbitrary "chunks" (e.g., batches over WS/SSE). Chunk boundaries are not semantically meaningful.
* **Message value is always a JSON object**: Not string, not array—object only.
* **Metadata-first rendering**: Start frames include metadata, enabling immediate UI rendering before content arrives.
* **Incremental rendering**: For object streaming, clients use partial JSON parsing. For text streaming, content renders directly.
* **Partial sync**: Clients can request history from a specific point on reconnect.
* **Stream multiplexing**: A single connection can carry multiple independent message streams, identified by the `s` field.

## 3. Terminology

* **Frame**: A single NDJSON record (one JSON object on one line).
* **Stream**: A logical grouping of messages, identified by stream ID `s`. What a stream represents (channel, thread, event feed) is application-defined.
* **Stream ID (`s`)**: An opaque string identifying which stream a frame belongs to. Optional on single-stream connections.
* **Message**: A logical item keyed by message ID `i`, belonging to a stream.
* **Message ID (`i`)**: A ULID string; sortable order defines transcript order within a stream.
* **Timestamp (`t`)**: ISO 8601 UTC timestamp with millisecond precision (e.g., `"2025-01-15T14:30:00.000Z"`). Timestamps MUST include milliseconds (`.000` through `.999`). Indicates when a frame was emitted.
* **Metadata (`m`)**: Optional JSON object in start frame describing the message (e.g., `{"type": "agent"}`).
* **Buffer**: The receiver's accumulated string for a message, built from append frames.
* **Value**: The final JSON object (complete message value).

## 4. Frame Encoding

### 4.1 NDJSON Format (required)

Each frame MUST be a single JSON object encoded as UTF-8, terminated by a newline character.

Example (text streaming with metadata — 3 frames, where each line ends with a newline):

```
{"i":"01J...","m":{"type":"agent"}}
{"i":"01J...","a":"Hello"}
{"i":"01J...","a":" world!"}
```

Example (object streaming — 3 frames):

```
{"i":"01J..."}
{"i":"01J...","a":"{\"status\":\"processing\",\"progress\":"}
{"i":"01J...","a":"50}"}
```

### 4.2 Chunking (informative)

Transports (SSE, WS, chunked HTTP) may group multiple NDJSON lines into a single delivered chunk. Receivers MUST treat the stream as a continuous byte sequence and split frames by newlines.

Chunk boundaries MUST NOT affect parsing or semantics.

### 4.3 UTF-8

All frames MUST be UTF-8.

## 5. Frame Schema

A frame MUST be one of:

1. **Start**: declares message existence, optionally with metadata
   `{"i":"<ulid>"}` or `{"i":"<ulid>","m":<metadata>}`

2. **Append**: appends a string fragment to the message buffer
   `{"i":"<ulid>","a":"<string fragment>"}`

3. **Set/Reset**: replaces the entire message value, or deletes message
   `{"i":"<ulid>","t":"<timestamp>","v":<json object>}` or `{"i":"<ulid>","v":null}`

4. **Sync**: requests stream history from server
   `{"request":"sync"}` or `{"request":"sync","s":"<stream>","since":"<timestamp>"}`

5. **Unsub**: unsubscribes from a stream (see §9)
   `{"request":"unsub","s":"<stream>"}`

6. **Error**: communicates protocol-level errors (see §11.1)
   `{"error":"<code>","message":"<description>"}`

**Stream ID on all frames:** Any message frame (start, append, set) MAY include an `s` field identifying the stream. See §9 for multiplexing rules.

### 5.1 Start Frame

**Meaning:** The message begins "now", optionally with metadata describing it.

**Fields:**

* `i` (required): message ULID
* `s` (optional): stream ID (see §9)
* `m` (optional): metadata JSON object

**Metadata constraints:**

* `m` MUST be a JSON object if present
* The key `"content"` is **reserved and MUST NOT appear** in `m`
* Common metadata: `{"type": "agent"}`, `{"type": "user"}`, `{"type": "error"}`

**Receiver behavior:**

* Create message state if it doesn't exist, or reset existing message state.
* Store metadata (`m`) if provided.
* Initialize buffer to the empty string.
* If metadata is present, immediately construct value as `{...m, "content": ""}`.
* If metadata is absent, this is object streaming mode (buffer will contain JSON).
* Mark message as streaming (not complete).

**Note:** A start frame for an existing message resets it—this allows re-streaming a message with new content.

### 5.2 Append Frame

**Meaning:** Append `a` to the end of the message's buffer string.

**Constraints:**

* `a` MUST be a string.
* Append frames received **before** a start frame for the same `i` MUST be ignored.
* Append frames received **after** a set frame for the same `i` MUST be ignored (message is already complete).

**Receiver behavior:**

* If message `i` doesn't exist yet: ignore (late-join safety).
* Else: `buffer[i] += a`
* Update the message value based on mode:
  * **Text mode** (metadata present): Set `value = {...metadata, "content": buffer}`
  * **Object mode** (no metadata): Attempt partial JSON parse of buffer

### 5.3 Set/Reset Frame

**Meaning:** Replace the entire message value (or delete it).

**Fields:**

* `i` (required): message ULID
* `t` (required for set, omit for delete): ISO 8601 UTC timestamp when this frame was emitted (e.g., `"2025-01-15T14:30:00.000Z"`)
* `v` (required): JSON object or `null`

**Important:** The `v` field contains a **JSON object directly**, not a JSON-encoded string.

**Receiver behavior:**

* If `v` is `null`: delete all state for `i` (buffer, parsed value, streaming state). Deletion removes the message from the transcript entirely. A deleted message ID MAY be reused by subsequent set frames (effectively recreating it).
* If `v` is an object:
  * Create message state if missing (remember ULID ordering—messages may arrive out of order).
  * Set `value = v` directly.
  * Store `t` as the message's last-updated timestamp.
  * Mark message as complete.

**Timestamp usage:** Clients should track the highest `t` value seen. On reconnect, this can be sent as the `since` parameter to sync only newer frames.

### 5.4 Sync Frame

**Meaning:** Client requests message history from server, optionally subscribing to a stream.

**Fields:**

* `request` (required): Must be `"sync"`
* `s` (optional): Stream ID to sync. On multiplexed connections, this subscribes the client to the stream and requests its history. See §9.
* `since` (optional): ISO 8601 UTC timestamp. If provided, server sends only messages with `t` greater than or equal to this value.

**Note:** Sync frames do not have an `i` field—they are control messages, not message updates.

**Receiver behavior (server):**

* If `s` is provided, subscribe the client to that stream (if not already subscribed).
* Query messages for the stream (or default stream), filtered by `since` if provided.
* Send a set frame (with `t` timestamp and `s` if multiplexed) for each completed message.
* If a message is currently streaming, client will receive it via append frames followed by a final set frame.

**Retention policy:** Servers are not required to retain message history indefinitely. If `since` refers to a time beyond the server's retention window, the server MUST return all available messages instead. A server with zero retention (always returning all messages) is compliant.

### 5.5 Unsub Frame

**Meaning:** Client unsubscribes from a stream. Server MUST stop sending frames for that stream.

**Fields:**

* `request` (required): Must be `"unsub"`
* `s` (required): Stream ID to unsubscribe from.

**Receiver behavior (server):**

* Remove the client from the stream's subscriber list.
* Stop sending frames for that stream to this client.
* If the client is not subscribed to the stream, ignore silently.

## 6. Message Value Rules (critical)

### 6.1 Value is Always a JSON Object

A message's semantic value MUST always be a JSON object.

For **text streaming** (start frame has `m`):
* Value is constructed as `{...m, "content": buffer}`
* Value is always valid and renderable, even with empty buffer
* Example: metadata `{"type": "agent"}` + buffer `"Hello"` → value `{"type": "agent", "content": "Hello"}`

For **object streaming** (start frame has no `m`):
* Value is constructed by parsing the buffer as JSON
* During streaming, use a partial JSON parser for intermediate values

### 6.2 Completeness

A message is considered **complete** when a set frame (`v`) is received.

A message is considered **streaming** when:
* A start frame was received, but no set frame yet
* For text mode: value is always available as `{...m, "content": buffer}`
* For object mode: value may be null or partial until buffer parses

### 6.3 Parsing Strategy (recommended)

For **text mode**: No parsing needed. Merge metadata with content directly.

For **object mode**: Receivers SHOULD:
* Attempt partial JSON parsing after each append
* Use a library like `partial-json` for robust incomplete JSON handling
* If parsing fails, keep last known-good value or null

## 7. Ordering and Transcript Construction

### 7.1 Message Order is Implied by ULID

To construct a transcript, clients SHOULD sort messages lexicographically by `i` (ULID). This yields creation-time order without requiring explicit sequence numbers.

### 7.2 Frame Order vs Message Order

Frames may arrive interleaved across messages. Receivers MUST apply frames in arrival order to each message's buffer, but transcript display order is determined by sorting message IDs.

## 8. Sync and Reconnection

### 8.1 Sync Request

Clients request history by sending a sync message:

```json
{"request": "sync"}
```

Or with a timestamp cursor for partial sync:

```json
{"request": "sync", "since": "2025-01-15T14:30:00.000Z"}
```

**Fields:**

* `request` (required): Must be `"sync"`
* `since` (optional): ISO 8601 UTC timestamp. Server sends only messages with `t` **greater than or equal to** this value. Using `>=` ensures messages aren't missed when multiple messages share the same timestamp (common at millisecond granularity during batch operations). This may result in duplicates, which are safe—just overwrite by ULID.

**Use cases:**

* **Initial connect**: Send `{"request": "sync"}` to get full history
* **Reconnect**: Send `{"request": "sync", "since": "<last-seen-timestamp>"}` to get only newer frames

### 8.2 Server Response to Sync

On receiving a sync request, servers MUST:

1. Query messages for the thread, optionally filtered by `since`
2. Send a set frame `{"i": <ulid>, "t": <timestamp>, "v": <value>}` for each completed message
3. If a message is currently streaming, the client will receive append frames as they arrive, then a final set frame

**Note:** If `since` is beyond the server's retention window, the server MUST return all available messages.

### 8.3 Server Requirements for Late-Join Support

To support clients connecting mid-stream, servers MUST:

1. **Send history on sync**: Respond to sync requests with set frames (including `t`) for all (or filtered) messages.
2. **Always finalize with a set frame**: When streaming completes for any message, ALWAYS send a final `{"i": <ulid>, "t": <timestamp>, "v": <value>}` frame. This ensures late-joiners receive the complete value.

### 8.4 Client Behavior for Late-Join

Clients connecting to an in-progress stream MUST:

1. **Send sync**: After connection is established, send a sync request.
2. **Process history**: Accept all `{"v": ...}` frames as the current transcript state.
3. **Track timestamps**: Store the highest `t` value seen for use in future `since` parameters.
4. **Ignore orphaned appends**: If an append frame arrives for a message ID the client has not seen a start frame for, ignore it. (Required by §5.2.)
5. **Wait for completion**: Ignored in-progress messages will arrive as complete set frames when streaming finishes.

### 8.5 Example: Reconnection with Partial Sync

Client disconnects at `2025-01-15T14:30:00.000Z`. On reconnect:

```
Client → {"request":"sync","since":"2025-01-15T14:30:00.000Z"}
Server → {"i":"01J...006","t":"2025-01-15T14:30:05.000Z","v":{"type":"user","content":"What's the weather?"}}
Server → {"i":"01J...007","t":"2025-01-15T14:30:10.000Z","v":{"type":"agent","content":"It's sunny today!"}}
Server → {"i":"01J...008","m":{"type":"agent"}}  ← currently streaming
Server → {"i":"01J...008","a":"Let me check"}
Server → {"i":"01J...008","a":" the forecast..."}
```

The client receives frames with `t >= since`, which may include some duplicates but ensures nothing is missed. Duplicates are harmless—just overwrite by ULID. This also catches updates to older messages if they were modified at or after `since`.

On a multiplexed connection, include `s` to sync a specific stream:

```
Client → {"request":"sync","s":"chat-general","since":"2025-01-15T14:30:00.000Z"}
```

---

## 9. Stream Multiplexing

A single Timbal connection can carry frames for multiple independent **streams**. What a stream represents is application-defined — it could be a chat channel, a conversation thread, a system event feed, or any other logical grouping of messages.

### 9.1 The `s` Field

All message frames (start, append, set) MAY include an `s` field:

```
{"s":"stream-abc","i":"01J...","m":{"type":"agent"}}
{"s":"stream-abc","i":"01J...","a":"Hello from stream abc"}
{"s":"stream-xyz","i":"01J...","t":"...","v":{"type":"user","content":"Hello from stream xyz"}}
```

**Rules:**

* `s` MUST be a string if present.
* `s` is **optional** — connections carrying a single stream do not need it.
* When a connection is multiplexed, servers MUST include `s` on **every** message frame (start, append, set). Clients MUST use `s` for routing — not connection-level state.
* Message IDs (`i`) are unique within a stream. The same ULID MUST NOT appear in different streams on the same connection.
* Each stream maintains its own independent message state (buffers, values, timestamps).

### 9.2 Subscribing to Streams

Clients subscribe to streams by sending a sync request with `s`:

```json
{"request": "sync", "s": "stream-abc"}
{"request": "sync", "s": "stream-abc", "since": "2025-01-15T14:30:00.000Z"}
```

**Behavior:**

* The server subscribes the client to the stream and sends its history (per §8).
* A client MAY subscribe to multiple streams on the same connection.
* Subscribing to an already-subscribed stream is idempotent — it re-syncs without duplicating the subscription.
* History frames in the response MUST include `s` so the client can route them.

### 9.3 Unsubscribing from Streams

Clients unsubscribe by sending an unsub request:

```json
{"request": "unsub", "s": "stream-abc"}
```

**Behavior:**

* The server stops sending frames for that stream to this client.
* The client SHOULD discard local state for the stream (buffers, values).
* Unsubscribing from a stream the client is not subscribed to is a no-op.

### 9.4 Stream Isolation

Streams are fully independent:

* Each stream has its own message ordering (by ULID within the stream).
* Each stream has its own sync cursor (`since` timestamp).
* Subscribing/unsubscribing from one stream does not affect others.
* A message belongs to exactly one stream.

### 9.5 Single-Stream Connections

When `s` is absent from all frames, the connection carries a single implicit "default stream." This is the common case for simple setups (e.g., one WebSocket per thread).

Single-stream and multiplexed modes are **not mixed** on the same connection. If any frame includes `s`, all message frames on that connection SHOULD include `s`.

### 9.6 Example: Multiplexed Connection

Client subscribes to two streams and receives interleaved frames:

```
Client → {"request":"sync","s":"chat-general"}
Client → {"request":"sync","s":"announcements"}

Server → {"s":"chat-general","i":"01J...001","t":"...","v":{"type":"user","content":"Hey everyone"}}
Server → {"s":"announcements","i":"01J...002","t":"...","v":{"type":"agent","content":"System update: v2.1 deployed"}}
Server → {"s":"chat-general","i":"01J...003","m":{"type":"agent"}}
Server → {"s":"chat-general","i":"01J...003","a":"Hi there!"}
Server → {"s":"chat-general","i":"01J...003","t":"...","v":{"type":"agent","content":"Hi there!"}}
```

Client switches focus — unsubscribes from one stream, subscribes to another:

```
Client → {"request":"unsub","s":"chat-general"}
Client → {"request":"sync","s":"chat-random"}

Server → {"s":"chat-random","i":"01J...010","t":"...","v":{"type":"user","content":"Anyone seen the new release?"}}
Server → {"s":"announcements","i":"01J...011","t":"...","v":{"type":"agent","content":"Maintenance window at 2am UTC"}}
```

No race condition — frames are self-describing. The client never needs to track "which stream am I on."

### 9.7 Application-Level Stream Types (informative)

The protocol treats all streams identically. Applications assign meaning:

| Application concept | Stream ID pattern (example) |
|--------------------|-----------------------------|
| Chat channel | `channel:general` |
| Conversation thread | `thread:550e8400-...` |
| System announcements | `system:announcements` |
| Agent status feed | `agent:weather-bot:status` |
| User notifications | `user:alice:notifications` |

Stream ID format is application-defined. The protocol imposes no structure beyond "it's a string."

---

## 10. UI Guidance (non-normative but recommended)

### 10.1 Metadata-First Rendering

With text streaming, UIs can render immediately upon receiving a start frame:

1. **Start frame arrives** (`{"i":"...","m":{"type":"agent"}}`): Show "Agent" label, empty content area
2. **Append frames arrive**: Update content progressively, show streaming indicator
3. **Set frame arrives**: Mark message complete, remove streaming indicator

This provides a better user experience than waiting for parseable JSON.

### 10.2 Streaming Indicators

Common patterns:
* Blinking cursor at end of streaming content
* "Typing..." indicator
* Pulsing background on the message bubble

### 10.3 When to Use Object Streaming

Use object streaming (start frame without `m`) for complex structured data that doesn't fit the `{...metadata, content}` pattern:

* Progress updates: `{"status": "processing", "progress": 50}`
* Structured tool results
* Custom message types with multiple fields

For most chat messages (user input, agent responses), text streaming with metadata is simpler and more efficient.

## 11. Error Handling

### 11.1 Error Frame

Servers MAY send error frames to communicate protocol-level errors to clients:

```
{"error":"<error_code>","message":"<human readable message>"}
```

**Fields:**

* `error` (required): A short error code string (e.g., `"rate_limited"`, `"invalid_thread"`, `"server_error"`)
* `message` (optional): Human-readable description of the error

**Note:** Error frames do not have an `i` field—they are control messages, not message updates.

**Receiver behavior:**

* Display error to user or log for debugging
* Error frames do not affect message state

**Common error codes:**

* `rate_limited` — Too many requests
* `invalid_thread` — Thread does not exist or access denied
* `server_error` — Internal server error

### 11.2 Invalid Frame JSON

If a line is not valid JSON, receivers MUST discard that line.

### 11.3 Invalid Frame Shape

Receivers MUST ignore **message frames** (start, append, set) that:

* lack `i`, or `i` is not a string
* contain both `a` and `v`
* have `a` present but not a string
* have `v` present but neither object nor null
* have `m` present but not an object
* have `m` containing the reserved key `"content"`

Control frames (sync, unsub, error) are identified by the presence of `request` or `error` fields and do not require `i`.

Unknown additional fields MUST be ignored for forward compatibility.

### 11.4 Invalid Message JSON

If the buffer parses but is not a JSON object, receivers MUST treat the message as invalid (and MAY surface an error state in UI).

## 12. Examples

### 12.1 Text Streaming with Metadata (recommended)

Streaming an agent message:

```
{"i":"01JEV5WQ7R1P0S6YB5T2JH9B3X","m":{"type":"agent"}}
{"i":"01JEV5WQ7R1P0S6YB5T2JH9B3X","a":"Hello"}
{"i":"01JEV5WQ7R1P0S6YB5T2JH9B3X","a":" world!"}
{"i":"01JEV5WQ7R1P0S6YB5T2JH9B3X","t":"2025-01-15T14:30:00.000Z","v":{"type":"agent","content":"Hello world!"}}
```

Client value progression:
1. After start: `{"type":"agent","content":""}`
2. After first append: `{"type":"agent","content":"Hello"}`
3. After second append: `{"type":"agent","content":"Hello world!"}`
4. After set: `{"type":"agent","content":"Hello world!"}` (complete, timestamp recorded)

### 12.2 Object Streaming (for complex structures)

Streaming a progress update:

```
{"i":"01J..."}
{"i":"01J...","a":"{\"status\":\"processing\",\"progress\":"}
{"i":"01J...","a":"50}"}
{"i":"01J...","t":"2025-01-15T14:30:00.000Z","v":{"status":"complete","progress":100}}
```

Requires partial JSON parser for intermediate values. Note that `v` contains a JSON object directly.

### 12.3 Complete Message via Set (no streaming)

For messages that don't need streaming (e.g., user input, historical messages):

```
{"i":"01J...","t":"2025-01-15T14:30:00.000Z","v":{"type":"user","content":"Hello!"}}
```

### 12.4 Deleting a Message

```
{"i":"01J...","v":null}
```

### 12.5 Interleaved Messages (transcript order by ULID)

```
{"i":"01J...001","m":{"type":"agent"}}
{"i":"01J...002","m":{"type":"agent"}}
{"i":"01J...002","a":"Second message"}
{"i":"01J...001","a":"First message"}
```

Display order (sorted by `i`): `...001` then `...002`, regardless of append order.

### 12.6 Sync with Timestamp Cursor

```
Client → {"request":"sync","since":"2025-01-15T14:00:00.000Z"}
Server → {"i":"01J...004","t":"2025-01-15T14:30:00.000Z","v":{"type":"user","content":"Next question"}}
Server → {"i":"01J...005","t":"2025-01-15T14:30:05.000Z","v":{"type":"agent","content":"Here's my answer"}}
```

### 12.7 Rich Metadata

Metadata can contain additional fields beyond `type`:

```
{"i":"01J...","m":{"type":"agent","model":"claude-3","toolUse":true}}
{"i":"01J...","a":"Let me search for that..."}
```

Value during streaming: `{"type":"agent","model":"claude-3","toolUse":true,"content":"Let me search for that..."}`

## 13. Conformance Checklist

An implementation is conformant with Timbal/1.0 if it:

* Accepts and emits NDJSON frames with schemas in §5
* Supports the `m` (metadata) field in start frames (§5.1)
* Treats `"content"` as reserved in metadata (§5.1)
* Includes `t` (timestamp) in set frames (§5.3)
* Constructs values correctly for both text and object streaming (§6)
* Ignores append-before-start (§5.2)
* Builds transcript order by ULID sort (§7)
* Supports sync requests with optional `since` cursor (§8)
* Supports the `s` field for stream multiplexing when present (§9)
* Includes `s` on all message frames when connection is multiplexed (§9.1)
* Supports `unsub` requests to unsubscribe from streams (§9.3)
* Ignores unknown fields (§11.3)

---

**Version**: Timbal/1.0
**Status**: Draft
