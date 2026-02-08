# Timbal HTTP Protocol Layer (Timbal HTTP/1.0)

**tl;dr** — RESTful HTTP API for creating and managing agent threads with Timbal streaming. Clients connect via WebSocket to receive NDJSON frames, then POST to create threads and send messages. Thread IDs are client-determined UUIDs.

## 1. Overview

The Timbal HTTP Protocol Layer defines a **RESTful API** for managing agent conversation threads on top of the Timbal framing layer. It provides:

* **Resource-oriented REST endpoints** for threads and messages
* **WebSocket transport** for receiving Timbal NDJSON frames
* **Client-determined thread IDs** using UUIDs for distributed coordination
* **Flexible connection order** — clients can connect to WebSocket before creating the thread
* **Capability negotiation** — threads declare agent capabilities and credentials on creation

This protocol layer sits **above** the Timbal framing layer and **below** application logic, defining how clients create threads, send messages, and receive streaming responses over HTTP/WebSocket.

## 2. Architecture

```
┌─────────────────────────────────────────┐
│         Application Logic               │
│  (Agent behavior, tool execution, etc)  │
└─────────────────────────────────────────┘
                   ↕
┌─────────────────────────────────────────┐
│    Timbal HTTP Protocol Layer           │
│  (Thread lifecycle, message routing)    │
│  • POST /threads/:threadId              │
│  • POST /threads/:threadId/messages     │
│  • POST /threads/:threadId/cancel       │
│  • WebSocket /threads/:threadId/stream  │
└─────────────────────────────────────────┘
                   ↕
┌─────────────────────────────────────────┐
│      Timbal Framing Layer               │
│   (NDJSON streaming, sync, ordering)    │
└─────────────────────────────────────────┘
                   ↕
┌─────────────────────────────────────────┐
│         Transport Layer                 │
│    (WebSocket, HTTP chunked, SSE)       │
└─────────────────────────────────────────┘
```

## 3. Base URL and Versioning

All endpoints are prefixed with a base URL:

```
https://api.example.com/v1
```

Version is included in the URL path (`/v1`) to enable breaking changes in future versions.

## 4. Thread Resource Model

A **thread** represents a conversation session with an agent. Each thread:

* Has a unique **UUID identifier** (client-determined)
* Contains **application-specific configuration** (capabilities, authentication, ownership)
* Maintains an **append-only message transcript** (via Timbal framing)
* Has a **lifecycle** (created → active → completed/cancelled)

### 4.1 Thread ID

Thread IDs MUST be valid UUIDs (Universally Unique Identifiers), preferably version 4 (random):

* **Format**: Standard UUID format (e.g., `550e8400-e29b-41d4-a716-446655440000`)
* **Client-determined**: Clients generate thread IDs before making requests
* **Collision resistance**: UUID randomness makes collisions extremely unlikely

**Rationale for client-determined IDs:**
* Enables idempotent thread creation
* Allows WebSocket connection before thread creation
* Simplifies distributed systems (no server round-trip for ID generation)
* Wide ecosystem support (standard format)

## 5. WebSocket Connection

### 5.1 Endpoint

```
WebSocket /threads/{threadId}/stream
```

**Purpose**: Receive Timbal NDJSON frames for a specific thread.

**Connection behavior**:
* Client MAY connect **before** the thread is created (server accepts connection)
* Client MAY connect **after** the thread is created
* Multiple clients MAY connect to the same thread (broadcast)
* Connection remains open until client disconnects or server closes

### 5.2 Protocol

Once connected, the WebSocket transport carries **Timbal NDJSON frames** as defined in the framing layer specification.

**Client → Server messages** (optional):
```json
{"request": "sync"}
{"request": "sync", "since": "2025-01-15T14:30:00.000Z"}
```

**Server → Client messages**:
* Start frames: `{"i": "01J...", "m": {"type": "agent"}}`
* Append frames: `{"i": "01J...", "a": "Hello"}`
* Set frames: `{"i": "01J...", "t": "2025-01-15T14:30:00.000Z", "v": {"type": "agent", "content": "Hello world!"}}`
* Error frames: `{"error": "rate_limited", "message": "Too many requests"}`

### 5.3 Connection Lifecycle

**Scenario 1: Connect → Create → Send**
```
1. Client generates threadId: 550e8400-e29b-41d4-a716-446655440000
2. Client connects WebSocket to /threads/550e8400-e29b-41d4-a716-446655440000/stream
3. Server accepts (thread doesn't exist yet, queues connection)
4. Client POSTs to /threads/550e8400-e29b-41d4-a716-446655440000 (create thread)
5. Client POSTs to /threads/550e8400-e29b-41d4-a716-446655440000/messages
6. Server streams frames via WebSocket
```

**Scenario 2: Create → Connect → Send**
```
1. Client generates threadId
2. Client POSTs to /threads/{threadId} (create thread)
3. Client connects WebSocket to /threads/{threadId}/stream
4. Client sends sync request: {"request": "sync"}
5. Server sends history (if any)
6. Client POSTs to /threads/{threadId}/messages
7. Server streams frames via WebSocket
```

### 5.4 Reconnection

On reconnection, clients SHOULD send a sync request with the last seen timestamp:

```json
{"request": "sync", "since": "2025-01-15T14:30:00.000Z"}
```

Server responds with all frames where `t >= since` (see Timbal framing spec §8.5).

### 5.5 WebSocket Close Codes

| Code | Meaning |
|------|---------|
| 1000 | Normal closure |
| 1003 | Unsupported data (invalid frame format) |
| 1008 | Policy violation (e.g., rate limit) |
| 1011 | Internal server error |
| 4004 | Thread not found (after grace period) |
| 4009 | Thread completed/cancelled |

**Grace period**: If a client connects before thread creation, servers SHOULD wait at least 30 seconds before sending 4004.

## 6. REST Endpoints

### 6.1 Create Thread

**Endpoint**: `POST /threads/{threadId}`

**Purpose**: Initialize a new agent thread.

**Path Parameters**:
* `threadId` (required): Client-generated UUID

**Request Body**:

The request body is **application-specific** and undefined by this specification. Implementations MAY use any JSON structure.

**Semantic Requirements**:

The payload SHOULD semantically define:
* **Capabilities**: What the agent can do (models, tools, limits, etc.)
* **Authentication**: Credentials for the agent's operations
* **Ownership**: Who owns/controls this thread

**Example** (informative, not normative):
```json
{
  "capabilities": {
    "model": "claude-opus-4-5-20251101",
    "tools": ["web_search", "code_execution"]
  },
  "credentials": {
    "apiKey": "sk-...",
    "userId": "user_123"
  }
}
```

**Note**: The specific structure, required fields, and validation rules are entirely implementation-defined.

**Response** (201 Created):
```json
{
  "threadId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "created",
  "createdAt": "2025-01-15T14:30:00.000Z",
  "streamUrl": "wss://api.example.com/v1/threads/550e8400-e29b-41d4-a716-446655440000/stream"
}
```

**Response** (200 OK — Idempotent):
If thread with this ID already exists and the request body matches the original creation request:
```json
{
  "threadId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "exists",
  "createdAt": "2025-01-15T14:25:00.000Z",
  "streamUrl": "wss://api.example.com/v1/threads/550e8400-e29b-41d4-a716-446655440000/stream"
}
```

**Error Responses**:

* `400 Bad Request`: Invalid threadId format or invalid request body
  ```json
  {
    "error": "invalid_request",
    "message": "threadId must be a valid UUID"
  }
  ```

* `401 Unauthorized`: Authentication failed (implementation-specific)
  ```json
  {
    "error": "unauthorized",
    "message": "Authentication failed"
  }
  ```

* `409 Conflict`: Thread exists with different configuration
  ```json
  {
    "error": "conflict",
    "message": "Thread already exists with different configuration"
  }
  ```

* `429 Too Many Requests`: Rate limit exceeded
  ```json
  {
    "error": "rate_limited",
    "message": "Too many threads created",
    "retryAfter": 60
  }
  ```

**Idempotency**: Creating a thread with the same ID and identical request body is idempotent (returns 200). This enables safe retries.

### 6.2 Send User Message

**Endpoint**: `POST /threads/{threadId}/messages`

**Purpose**: Send a user message to the agent thread, triggering agent processing.

**Path Parameters**:
* `threadId` (required): Thread UUID

**Request Body**:
```json
{
  "content": "What's the weather in San Francisco?",
  "metadata": {
    "role": "user",
    "timestamp": "2025-01-15T14:30:00.000Z",
    "attachments": []
  }
}
```

**Request Fields**:
* `content` (required): Message text
* `metadata` (optional): Additional message metadata
  * `role` (optional): Message role (default: "user")
  * `timestamp` (optional): Client timestamp (server may override)
  * Additional fields are implementation-specific

**Response** (202 Accepted):
```json
{
  "messageId": "01J4KZ3Y9ABCDEFGHIJKLMNOPQ",
  "threadId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "receivedAt": "2025-01-15T14:30:00.123Z"
}
```

**Response Fields**:
* `messageId`: Server-generated ULID for the user message
* `threadId`: Echo of thread ID
* `status`: Processing state (always "processing" for async)
* `receivedAt`: Server timestamp when message was received

**Error Responses**:

* `400 Bad Request`: Invalid message format
* `404 Not Found`: Thread does not exist
* `409 Conflict`: Thread is in a non-active state
* `429 Too Many Requests`: Rate limit exceeded

**Behavior**:
* Server immediately returns 202 (does not wait for agent response)
* Agent processes message asynchronously
* Frames are streamed to WebSocket clients as they're generated
* User message and agent response appear as Timbal frames on the WebSocket

**Expected WebSocket Frames**:
```
← {"i":"01J4KZ3Y9ABCDEFGHIJKLMNOPQ","t":"2025-01-15T14:30:00.123Z","v":{"type":"user","content":"What's the weather in San Francisco?"}}
← {"i":"01J4KZ3Z1XYZABCDEFGHIJKLMNO","m":{"type":"agent"}}
← {"i":"01J4KZ3Z1XYZABCDEFGHIJKLMNO","a":"Let me check"}
← {"i":"01J4KZ3Z1XYZABCDEFGHIJKLMNO","a":" the weather..."}
← {"i":"01J4KZ3Z1XYZABCDEFGHIJKLMNO","t":"2025-01-15T14:30:05.000Z","v":{"type":"agent","content":"Let me check the weather for San Francisco..."}}
```

### 6.3 Cancel Thread Processing

**Endpoint**: `POST /threads/{threadId}/cancel`

**Purpose**: Request cancellation of in-progress agent processing.

**Path Parameters**:
* `threadId` (required): Thread UUID

**Request Body** (optional):
```json
{
  "reason": "user_requested"
}
```

**Response** (200 OK):
```json
{
  "threadId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "cancelling",
  "cancelledAt": "2025-01-15T14:30:10.000Z"
}
```

**Behavior**:
* Server sends cancellation signal to agent process
* Agent SHOULD stop processing and clean up resources
* Streaming stops (current message may be incomplete)
* Final frame MAY indicate cancellation
* WebSocket MAY close with code 4009

**Note**: Cancellation is **best-effort**. Some agents may not support mid-processing cancellation.

### 6.4 Get Thread Status (optional)

**Endpoint**: `GET /threads/{threadId}`

**Purpose**: Retrieve thread metadata and status.

**Note**: This endpoint is **optional** for implementations. Clients can derive status from WebSocket frames.

### 6.5 Delete Thread (optional)

**Endpoint**: `DELETE /threads/{threadId}`

**Purpose**: Delete thread and all associated messages.

**Response** (204 No Content)

**Behavior**:
* All message data is deleted
* WebSocket connections are closed with code 4009
* Thread ID may be reused for a new thread

**Note**: This endpoint is **optional**. Many implementations use automatic retention policies instead.

## 7. Headers

### 7.1 Request Headers

All requests SHOULD include:

* `Content-Type: application/json` (for POST requests with body)
* `Accept: application/json`
* `User-Agent: <client identifier>` (for analytics)

Authentication headers are implementation-specific:
* `Authorization: Bearer <token>` (OAuth/JWT)
* `X-API-Key: <key>` (API key)

### 7.2 Response Headers

All responses include:

* `Content-Type: application/json; charset=utf-8`
* `X-Request-Id: <unique request id>` (for debugging)

Rate-limited responses (429) include:
* `Retry-After: <seconds>`
* `X-RateLimit-Limit: <max requests>`
* `X-RateLimit-Remaining: <remaining requests>`
* `X-RateLimit-Reset: <unix timestamp>`

## 8. Error Response Format

All error responses follow a consistent schema:

```json
{
  "error": "<error_code>",
  "message": "<human-readable description>",
  "details": {
    "field": "threadId",
    "reason": "must be a valid UUID"
  }
}
```

**Fields**:
* `error` (required): Machine-readable error code
* `message` (required): Human-readable description
* `details` (optional): Additional context (field validation errors, etc.)

**Common Error Codes**:
* `invalid_request` — Malformed request (400)
* `unauthorized` — Authentication failed (401)
* `forbidden` — Authorization failed (403)
* `thread_not_found` — Thread does not exist (404)
* `conflict` — Resource conflict (409)
* `rate_limited` — Too many requests (429)
* `server_error` — Internal server error (500)
* `service_unavailable` — Service temporarily unavailable (503)

## 9. Security Considerations

### 9.1 Authentication

The thread creation payload typically includes authentication information for the agent's operations. The HTTP API itself MAY require separate authentication via headers.

**Layered auth model** (implementation-specific):
1. **HTTP layer auth**: Authenticates the client making API requests
2. **Thread-level auth**: Authenticates the agent for tool use, API calls, etc.

This separation allows clients to create threads on behalf of different users.

### 9.2 Authorization

Servers MUST verify:
* Client has permission to create threads
* Client has permission to send messages to a thread
* Thread configuration is valid and authorized

### 9.3 Rate Limiting

Servers SHOULD implement rate limiting:
* Thread creation rate (e.g., 10 threads/minute per API key)
* Message send rate (e.g., 60 messages/minute per thread)
* WebSocket connection rate (e.g., 100 connections/minute per IP)

### 9.4 Input Validation

Servers MUST validate:
* `threadId` is a valid UUID format
* Request body JSON is well-formed
* Application-specific validation of thread configuration

### 9.5 Sensitive Data Handling

* Sensitive data in the thread creation payload (credentials, tokens, etc.) MUST NOT appear in logs or error messages
* Servers MUST transmit all data over TLS (HTTPS/WSS only)
* Servers SHOULD support credential rotation (implementation-specific)

### 9.6 WebSocket Security

* WebSocket connections MUST use WSS (TLS)
* Servers SHOULD validate `Origin` header to prevent CSRF
* Servers MAY require authentication via query parameters, headers, or subprotocols

## 10. CORS (Cross-Origin Resource Sharing)

For browser-based clients, servers SHOULD support CORS:

**Preflight response** (`OPTIONS /threads/{threadId}`):
```
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-API-Key
Access-Control-Max-Age: 86400
```

WebSocket CORS is handled via `Origin` header validation.

## 11. Example

**Basic flow**:
```javascript
// 1. Generate thread ID and connect WebSocket
const threadId = crypto.randomUUID();
const ws = new WebSocket(`wss://api.example.com/v1/threads/${threadId}/stream`);

// 2. Create thread
fetch(`https://api.example.com/v1/threads/${threadId}`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ capabilities: {...}, credentials: {...} })
});

// 3. Send message
fetch(`https://api.example.com/v1/threads/${threadId}/messages`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ content: "Hello" })
});

// 4. Receive frames via WebSocket
ws.onmessage = (event) => {
  const frame = JSON.parse(event.data);
  // Frames: {"i":"01J...","m":{"type":"agent"}}
  //         {"i":"01J...","a":"Hello"}
  //         {"i":"01J...","t":"...","v":{"type":"agent","content":"Hello world"}}
};
```

## 12. Implementation Checklist

A conformant Timbal HTTP/1.0 implementation MUST:

* Support client-determined thread IDs (UUIDs)
* Accept WebSocket connections to `/threads/{threadId}/stream`
* Allow WebSocket connection before thread creation
* Implement `POST /threads/{threadId}` for thread creation
* Implement `POST /threads/{threadId}/messages` for sending messages
* Implement `POST /threads/{threadId}/cancel` for cancellation
* Return proper HTTP status codes (201, 202, 400, 401, 404, 409, 429, etc.)
* Stream Timbal NDJSON frames via WebSocket
* Support sync requests over WebSocket
* Implement idempotent thread creation
* Validate thread IDs as UUIDs
* Use TLS for all connections (HTTPS/WSS)

A conformant implementation SHOULD:

* Support CORS for browser clients
* Implement rate limiting
* Return standardized error response format
* Include rate limit headers in 429 responses
* Support WebSocket reconnection with sync
* Validate credentials and enforce authorization
* Log requests with unique request IDs

## 13. Relationship to Timbal Framing Layer

The HTTP protocol layer is **complementary** to the framing layer:

| Layer | Responsibility |
|-------|---------------|
| **HTTP Protocol** | Thread lifecycle, message routing, authentication, REST API |
| **Framing** | Message ordering, streaming format, sync protocol, NDJSON encoding |

Both layers are **required** for a complete Timbal HTTP implementation. The HTTP layer uses WebSocket as transport, and the framing layer runs on top of that WebSocket connection.

## 14. Future Extensions (non-normative)

Possible future additions to the HTTP protocol layer:

* **Batch message sending**: `POST /threads/{threadId}/messages/batch`
* **Thread search/listing**: `GET /threads?status=active&limit=20`
* **Message history endpoint**: `GET /threads/{threadId}/messages?since=<timestamp>`
* **Thread templates**: Predefined capability sets
* **Streaming via SSE**: Alternative to WebSocket for unidirectional streaming
* **Multimodal content**: Image/file attachments in messages
* **Thread forking**: Create a new thread from a message in existing thread
* **Thread sharing**: Share read-only access to threads

---

**Version**: Timbal HTTP/1.0
**Status**: Draft
