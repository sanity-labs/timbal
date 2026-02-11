# Chorus — Channel Protocol

**Version:** 0.1 (draft)

A protocol for connecting AI agents to shared channels. Any service — a collaboration platform, a Slack adapter, a Discord bridge — can use Chorus to deliver messages to agents and let them participate in channels.

## How It Works

The service POSTs a message to the agent. The payload includes everything the agent needs to participate:

1. **Channel** — Where this is happening (ID, name, context)
2. **Message** — What was said
3. **Callback** — URL to post output back (messages, tool calls, status)
4. **MCP** — Tools the agent can use (read history, create artifacts, etc.)

```
┌─────────┐                          ┌─────────┐
│         │  POST message + callback  │         │
│ Service │ ───────────────────────→  │  Agent  │
│         │                           │         │
│         │  ← POST to callback URL   │         │
│         │    (messages, status)      │         │
│         │                           │         │
│         │  ← MCP tool calls         │         │
│         │    (read history, etc.)    │         │
└─────────┘                          └─────────┘
```

## 1. Connection

To connect an agent to a channel, the service needs one thing: the agent's **connection string**.

```
https://agent.example.com/inbox/sk_a1b2c3d4e5f6
```

A connection string is a URL — it contains the agent's hostname and an embedded secret key. Paste it into the service, and you're done.

**The connection string is a secret.** Anyone with the URL can send messages to the agent. This is the same pattern as Slack incoming webhooks and Stripe webhook endpoints.

### Setup

1. Agent owner generates a connection string from their agent service
2. Agent owner provides it to the channel service (paste into UI, API call, etc.)
3. Channel service stores it and starts POSTing messages
4. Done

### Rotation

If compromised, the agent owner generates a new connection string. The old one stops working immediately.

## 2. Message Delivery

The service delivers messages by POSTing JSON to the agent's connection string.

### Request

```
POST {connection_string}
Content-Type: application/json
```

```json
{
  "channel": {
    "id": "ch_abc123",
    "name": "ops-team",
    "service": "Miriad — multi-agent collaboration studio",
    "context": "A channel in @alice's Miriad space. The team uses this channel for ops coordination."
  },
  "message": {
    "id": "msg_01HX...",
    "sender": "alice",
    "content": "What's the weather in Oslo?"
  },
  "callback": "https://service.example.com/channels/ch_abc123/inbox/tok_xyz789",
  "mcp": {
    "url": "https://service.example.com/mcp/ch_abc123",
    "headers": {
      "Authorization": "Bearer eyJ..."
    }
  }
}
```

### Fields

#### `channel` (required)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Immutable unique identifier. Stable across messages. The agent uses this to maintain per-channel state. |
| `name` | string | | Human-readable channel name. May change over time. |
| `service` | string | | Name of the hosting service. Helps the agent understand what platform it's on. |
| `context` | string | | Freeform context about the channel — who owns the space, what the channel is for, relevant background. |

#### `message` (required)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique message identifier. |
| `sender` | string | ✅ | Display name or identifier of the sender. |
| `content` | string | ✅ | Message content (plain text or markdown). |

#### `callback` (required)

URL where the agent POSTs its output (§3). The URL is opaque — authentication is embedded in the URL itself. The agent POSTs to it as-is, no auth headers needed.

The service generates a fresh callback URL per message delivery.

#### `mcp` (optional)

MCP server configuration. The agent passes this directly to its MCP client library.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | ✅ | MCP HTTP transport endpoint URL. |
| `headers` | object | | HTTP headers for MCP requests (auth, etc.). |

The MCP server exposes whatever tools the service provides — message history, artifacts, roster, knowledge bases, file uploads. This is service-specific and not prescribed by Chorus.

The agent SHOULD treat the `mcp` object as opaque configuration — pass it to an MCP client library as-is.

### Response

The agent responds `200 OK` to acknowledge receipt. The response body is ignored. All output goes through the callback.

If the agent cannot process the message, it responds with an appropriate HTTP error. The service MAY retry on 5xx.

## 3. Callback

The agent posts output to the callback URL. Each POST is a single JSON object. No auth headers — the URL is self-authenticating.

```
POST {callback_url}
Content-Type: application/json
```

### Event Types

#### `message` — Say something in the channel

```json
{
  "type": "message",
  "content": "The weather in Oslo is 3°C and cloudy."
}
```

#### `tool_call` — Report a tool invocation

```json
{
  "type": "tool_call",
  "name": "web_search",
  "args": { "query": "oslo weather" },
  "id": "tc_001"
}
```

#### `tool_result` — Report a tool result

```json
{
  "type": "tool_result",
  "id": "tc_001",
  "content": "Oslo: 3°C, cloudy, wind 5m/s NW"
}
```

#### `status` — Update agent status

```json
{
  "type": "status",
  "status": "searching weather data"
}
```

#### `error` — Report an error

```json
{
  "type": "error",
  "message": "Failed to connect to weather API",
  "code": "tool_error"
}
```

### Response

The service responds `200 OK` to acknowledge. On non-2xx, the agent MAY retry or drop the event.

### Ordering

The agent MAY send multiple POSTs per inbound message. They SHOULD be sent in order. The service SHOULD process them in receipt order.

## 4. Authentication

### Service → Agent

The connection string IS the credential. The embedded secret authenticates the service. No additional auth headers are required, though the agent owner MAY configure additional verification (signed JWTs, IP allowlists, etc.).

### Agent → Service

**Callback:** Self-authenticating URL. The service embeds credentials in the URL. The agent POSTs as-is. Same pattern as Slack webhooks and AWS pre-signed URLs — with HTTPS, the full URL is encrypted in transit.

**MCP:** The `mcp` config contains whatever auth the service requires, typically in `headers`. The agent passes it to its MCP client library, which handles auth transparently.

No registration dance. No key exchange. Credentials arrive with every message.

## 5. Design Principles

1. **Stateless delivery.** Each POST contains everything the agent needs — channel identity, callback, MCP config. No persistent connections, no session state.

2. **MCP for capabilities.** The service's features are standard MCP tools. Different services expose different tools. The agent discovers them via MCP tool listing.

3. **Callback for output.** The agent's visible activity — messages, tool reports, status — goes through the callback. Simple POSTs, no protocol negotiation.

4. **Service-agnostic.** The agent doesn't know or care what service it's talking to. Miriad, Slack, Discord — all invisible behind the same interface.

5. **Fresh credentials per message.** Callback URLs and MCP configs are provided with every delivery. The service can rotate freely. The agent uses what it's given.

6. **Connection string simplicity.** One URL to connect. One secret to manage. Copy, paste, done.

## 6. Examples

### Minimal — Slack adapter, no MCP

```json
{
  "channel": {
    "id": "C04ABCDEF",
    "name": "general",
    "service": "Slack",
    "context": "The Slack workspace of Sanity (sanity-io.slack.com)."
  },
  "message": { "id": "msg_1", "sender": "alice", "content": "hello" },
  "callback": "https://slack-adapter.example.com/cb/C04ABCDEF/tok_abc123"
}
```

```
POST https://slack-adapter.example.com/cb/C04ABCDEF/tok_abc123
Content-Type: application/json

{ "type": "message", "content": "Hello Alice!" }
```

### Full — Miriad with MCP tools

```json
{
  "channel": {
    "id": "01HX...",
    "name": "engineering",
    "service": "Miriad — multi-agent collaboration studio",
    "context": "A channel in @svale's Miriad space. Used for protocol design and infrastructure work."
  },
  "message": {
    "id": "01HY...",
    "sender": "svale",
    "content": "@timber can you review the auth spec?"
  },
  "callback": "https://miriad.example.com/channels/01HX.../inbox/tok_eyJhbG",
  "mcp": {
    "url": "https://miriad.example.com/mcp/01HX...",
    "headers": { "Authorization": "Bearer eyJhbGciOi..." }
  }
}
```

Agent:
1. Passes `mcp` config to MCP client library — connects, authenticates automatically
2. Discovers tools: `artifact_read`, `artifact_edit`, `send_message`, `get_roster`, ...
3. Calls `artifact_read({ slug: "auth-spec" })` via MCP
4. POSTs to callback: `{ "type": "status", "status": "reviewing auth spec" }`
5. POSTs to callback: `{ "type": "message", "content": "I've reviewed the auth spec. Two issues..." }`

## 7. Open Questions

1. **Multiple MCP servers?** Should `mcp` be an array? A service might provide separate servers for different capability sets.

2. **Streaming callback?** NDJSON streaming (keep connection open, send multiple JSON objects) instead of separate POSTs? Lower overhead for chatty agents.

3. **Proactive messages?** This protocol is reactive (message in → agent acts). What about unprompted agent messages? Options: (a) agent keeps last callback URL, (b) separate endpoint, (c) out of scope.

4. **Structured content?** Should `message.content` support images, files, embeds? Or is that always through MCP tools?

5. **Batch delivery?** Multiple messages in one POST for catch-up scenarios?

6. **Agent identity?** Should the agent declare itself (name, avatar, capabilities)? Or is that handled at connection time?
