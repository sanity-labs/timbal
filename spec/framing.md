# Timbal Framing Layer (Timbal/1.0)

**tl;dr** — Timbal defines the framing layer for streaming agent conversations. Each message is a sortable ULID and its contents stream progressively as NDJSON frames. Start frames declare metadata upfront, enabling immediate UI rendering. A single connection can multiplex multiple streams via the `s` field. Control frames (`c`) provide an extension point for application-specific operations like sync, subscribe, and filtering.

## 1. Overview

Timbal defines the **framing layer** for streaming message transcripts as an **append-only sequence of NDJSON frames**. Each frame updates a **message**, identified by a **sortable ULID**, where message order is implied by the ULIDs.

This framing layer sits above the transport layer (WebSocket, SSE, HTTP, stdio) and defines how messages are encoded, streamed, and routed.

Each message's **value is always a JSON object**. Messages can be streamed in two modes:

1. **Text streaming with metadata**: Start frame includes metadata (e.g., `{"type": "agent"}`), and appends contain plain text that becomes the `content` field. No JSON escaping needed.
2. **Object streaming**: Start frame has no metadata, and appends contain JSON fragments requiring partial parsing.

The framing layer is transport-agnostic and works over:

* HTTP streaming (chunked responses)
* Server-Sent Events (SSE)
* WebSockets
* stdio pipes

**Stream multiplexing:** A single connection can carry frames for multiple logical streams (channels, threads, event feeds) using the optional `s` field. See §8.

**Extensibility:** Control frames (`c`) provide a typed extension point for application-specific operations. The core spec defines the control frame shape but only mandates `error` — everything else (sync, subscribe, filters) is defined by extensions. See §7.

## 2. Key Properties

* **Message ordering is implied by IDs**: ULIDs are lexicographically sortable by creation time, so sorting by `i` yields message order.
* **NDJSON frames**: The wire format is newline-delimited JSON objects.
* **Streaming in groups**: Transports may deliver frames in arbitrary "chunks" (e.g., batches over WS/SSE). Chunk boundaries are not semantically meaningful.
* **Message value is always a JSON object**: Not string, not array—object only.
* **Metadata-first rendering**: Start frames include metadata, enabling immediate UI rendering before content arrives.
* **Incremental rendering**: For object streaming, clients use partial JSON parsing. For text streaming, content renders directly.
* **Stream multiplexing**: A single connection can carry multiple independent message streams, identified by the `s` field.
* **Extensible control plane**: Control frames provide a typed extension point. Applications define their own control types for sync, subscribe, filtering, etc.

## 3. Terminology

* **Frame**: A single NDJSON record (one JSON object on one line).
* **Message frame**: A frame that creates or updates a message (has `i` field).
* **Control frame**: A frame that carries a control operation (has `c` field). Not a message — not part of the transcript.
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

## 5. Frame Types

Every frame is exactly one of: **message frame** or **control frame**.

* **Message frames** have an `i` field (message ID). They create or update messages in the transcript.
* **Control frames** have a `c` field (control type). They carry operations that are not messages — commands, responses, errors.

A frame MUST NOT have both `i` and `c`. A frame with neither MUST be ignored.

### 5.1 Message Frames

#### 5.1.1 Start Frame

**Meaning:** The message begins "now", optionally with metadata describing it.

`{"i":"<ulid>"}` or `{"i":"<ulid>","m":<metadata>}`

**Fields:**

* `i` (required): message ULID
* `s` (optional): stream ID (see §8)
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

#### 5.1.2 Append Frame

**Meaning:** Append `a` to the end of the message's buffer string.

`{"i":"<ulid>","a":"<string fragment>"}`

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

#### 5.1.3 Set/Reset Frame

**Meaning:** Replace the entire message value (or delete it).

`{"i":"<ulid>","t":"<timestamp>","v":<json object>}` or `{"i":"<ulid>","v":null}`

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

### 5.2 Control Frames

Control frames carry operations that are **not messages** — they don't appear in the transcript and don't have ULIDs. They provide a typed extension point for application-specific operations.

**Shape:** `{"c":"<type>", ...params}`

**Rules:**

* `c` MUST be a string (the control type).
* A control frame MUST NOT have an `i` field.
* Receivers MUST ignore control frames with unknown `c` types (forward compatibility).
* Control frames MAY include an `s` field to scope the operation to a specific stream.

**Core control type — `error`:**

The only control type defined by this spec. All others are defined by extensions.

```
{"c":"error","code":"rate_limited","message":"Too many requests"}
{"c":"error","code":"invalid_stream","message":"Stream does not exist","s":"chat-general"}
```

**Fields for `error`:**

* `c` (required): Must be `"error"`
* `code` (required): A short error code string
* `message` (optional): Human-readable description
* `s` (optional): Stream the error relates to

**Common error codes:**

* `rate_limited` — Too many requests
* `invalid_stream` — Stream does not exist or access denied
* `server_error` — Internal server error

**Extension control types** are defined outside this spec. See §7 for the extension model.

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

## 7. Extension Model

Timbal's core defines message frames (the data format) and the control frame shape (the extension point). Application-specific behavior is layered on top through two mechanisms:

### 7.1 Control Frame Extensions

Extensions define new control types by specifying a `c` value and its associated fields. For example, a sync extension might define:

```
{"c":"sync","s":"chat-general","since":"2025-01-15T14:30:00.000Z"}
{"c":"unsub","s":"chat-general"}
```

A library that doesn't know about the sync extension will ignore these frames (per §5.2 — unknown control types are ignored). A library that does can register a handler for `c:"sync"`.

**Conventions for extension authors:**

* Document the `c` value, required/optional fields, and expected behavior.
* Use descriptive type names. Avoid single-character or ambiguous names.
* Extensions SHOULD be self-contained — a receiver that ignores unknown control types should still function correctly for the parts it does understand.

### 7.2 Application-Defined Message Types

The `m.type` field in message metadata is application-defined. The core spec does not mandate any specific types — `"agent"`, `"user"`, etc. are conventions, not requirements.

Applications MAY define custom message types. The `x-` prefix convention signals application-specific types that generic tooling should not attempt to interpret:

```
{"i":"01J...","t":"...","v":{"type":"x-roster","agents":[...]}}
{"i":"01J...","m":{"type":"x-cost","model":"claude-3"}}
```

Receivers MUST ignore messages with unrecognized types (or render them generically). The `x-` prefix makes it explicit which types are application-specific vs. conventional.

### 7.3 Unknown Fields

For forward compatibility, receivers MUST ignore unknown fields on any frame type. This allows both the core spec and extensions to add fields in future versions without breaking existing implementations.

## 8. Stream Multiplexing

A single Timbal connection can carry frames for multiple independent **streams**. What a stream represents is application-defined — it could be a chat channel, a conversation thread, a system event feed, or any other logical grouping of messages.

### 8.1 The `s` Field

All frames (message and control) MAY include an `s` field:

```
{"s":"stream-abc","i":"01J...","m":{"type":"agent"}}
{"s":"stream-abc","i":"01J...","a":"Hello from stream abc"}
{"s":"stream-xyz","i":"01J...","t":"...","v":{"type":"user","content":"Hello from stream xyz"}}
{"s":"stream-abc","c":"error","code":"rate_limited","message":"Slow down"}
```

**Rules:**

* `s` MUST be a string if present.
* `s` is **optional** — connections carrying a single stream do not need it.
* When a connection is multiplexed, servers MUST include `s` on **every** message frame (start, append, set). Clients MUST use `s` for routing — not connection-level state.
* Message IDs (`i`) are unique within a stream. The same ULID MUST NOT appear in different streams on the same connection.
* Each stream maintains its own independent message state (buffers, values, timestamps).

### 8.2 Stream Isolation

Streams are fully independent:

* Each stream has its own message ordering (by ULID within the stream).
* Each stream has its own state (buffers, values, timestamps).
* A message belongs to exactly one stream.

### 8.3 Single-Stream Connections

When `s` is absent from all frames, the connection carries a single implicit "default stream." This is the common case for simple setups (e.g., one WebSocket per conversation).

Single-stream and multiplexed modes are **not mixed** on the same connection. If any frame includes `s`, all message frames on that connection SHOULD include `s`.

### 8.4 Application-Level Stream Types (informative)

The protocol treats all streams identically. Applications assign meaning:

| Application concept | Stream ID pattern (example) |
|--------------------|----------------------------|
| Chat channel | `channel:general` |
| Conversation thread | `thread:550e8400-...` |
| System announcements | `system:announcements` |
| Agent status feed | `agent:weather-bot:status` |
| User notifications | `user:alice:notifications` |

Stream ID format is application-defined. The protocol imposes no structure beyond "it's a string."

## 9. Ordering and Transcript Construction

### 9.1 Message Order is Implied by ULID

To construct a transcript, clients SHOULD sort messages lexicographically by `i` (ULID). This yields creation-time order without requiring explicit sequence numbers.

### 9.2 Frame Order vs Message Order

Frames may arrive interleaved across messages. Receivers MUST apply frames in arrival order to each message's buffer, but transcript display order is determined by sorting message IDs.

## 10. Error Handling

### 10.1 Invalid Frame JSON

If a line is not valid JSON, receivers MUST discard that line.

### 10.2 Invalid Frame Shape

Receivers MUST ignore **message frames** (start, append, set) that:

* lack `i`, or `i` is not a string
* contain both `a` and `v`
* have `a` present but not a string
* have `v` present but neither object nor null
* have `m` present but not an object
* have `m` containing the reserved key `"content"`

Receivers MUST ignore **control frames** that:

* lack `c`, or `c` is not a string

Frames with neither `i` nor `c` MUST be ignored.

Unknown additional fields MUST be ignored for forward compatibility.

### 10.3 Invalid Message JSON

If the buffer parses but is not a JSON object, receivers MUST treat the message as invalid (and MAY surface an error state in UI).

## 11. UI Guidance (non-normative)

### 11.1 Metadata-First Rendering

With text streaming, UIs can render immediately upon receiving a start frame:

1. **Start frame arrives** (`{"i":"...","m":{"type":"agent"}}`): Show "Agent" label, empty content area
2. **Append frames arrive**: Update content progressively, show streaming indicator
3. **Set frame arrives**: Mark message complete, remove streaming indicator

This provides a better user experience than waiting for parseable JSON.

### 11.2 Streaming Indicators

Common patterns:
* Blinking cursor at end of streaming content
* "Typing..." indicator
* Pulsing background on the message bubble

### 11.3 When to Use Object Streaming

Use object streaming (start frame without `m`) for complex structured data that doesn't fit the `{...metadata, content}` pattern:

* Progress updates: `{"status": "processing", "progress": 50}`
* Structured tool results
* Custom message types with multiple fields

For most chat messages (user input, agent responses), text streaming with metadata is simpler and more efficient.

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

### 12.6 Rich Metadata

Metadata can contain additional fields beyond `type`:

```
{"i":"01J...","m":{"type":"agent","model":"claude-3","toolUse":true}}
{"i":"01J...","a":"Let me search for that..."}
```

Value during streaming: `{"type":"agent","model":"claude-3","toolUse":true,"content":"Let me search for that..."}`

### 12.7 Multiplexed Streams

Frames from different streams interleaved on one connection:

```
{"s":"chat-general","i":"01J...001","t":"...","v":{"type":"user","content":"Hey everyone"}}
{"s":"announcements","i":"01J...002","t":"...","v":{"type":"agent","content":"System update: v2.1 deployed"}}
{"s":"chat-general","i":"01J...003","m":{"type":"agent"}}
{"s":"chat-general","i":"01J...003","a":"Hi there!"}
{"s":"chat-general","i":"01J...003","t":"...","v":{"type":"agent","content":"Hi there!"}}
```

No race condition — frames are self-describing. The client routes by `s`, not connection state.

### 12.8 Control Frames

Error scoped to a stream:

```
{"c":"error","code":"rate_limited","message":"Too many requests","s":"chat-general"}
```

Extension control frame (from a sync extension — not part of core):

```
{"c":"sync","s":"chat-general","since":"2025-01-15T14:30:00.000Z"}
{"c":"unsub","s":"chat-general"}
```

## 13. Conformance Checklist

An implementation is conformant with Timbal/1.0 if it:

* Accepts and emits NDJSON frames (§4)
* Distinguishes message frames (`i`) from control frames (`c`) (§5)
* Supports start, append, and set message frames (§5.1)
* Supports the `m` (metadata) field in start frames (§5.1.1)
* Treats `"content"` as reserved in metadata (§5.1.1)
* Includes `t` (timestamp) in set frames (§5.1.3)
* Constructs values correctly for both text and object streaming (§6)
* Ignores append-before-start (§5.1.2)
* Handles the `error` control type (§5.2)
* Ignores unknown control types (§5.2)
* Builds transcript order by ULID sort (§9)
* Supports the `s` field for stream multiplexing when present (§8)
* Includes `s` on all message frames when connection is multiplexed (§8.1)
* Ignores unknown fields (§7.3)

---

**Version**: Timbal/1.0
**Status**: Draft
