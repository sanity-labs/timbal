# Timbal

**Protocols for real-time agent communication.**

Timbal is a family of protocols for agentic systems: how messages are streamed, how agents connect to channels, and how tools are discovered.

## Protocols

### [Chorus](spec/chorus.md) — Channel Protocol

Connect agents to shared channels on any platform. A service POSTs messages to the agent with a callback URL and MCP config. The agent posts output back and uses MCP tools to interact with the channel.

```
Service  ──POST message + callback + MCP──→  Agent
         ←──POST to callback (messages)────
         ←──MCP tool calls (actions)───────
```

One URL to connect. One secret to manage. Works with any service — Miriad, Slack, Discord, whatever.

→ [Read the spec](spec/chorus.md)

### Timbal Framing — Streaming Protocol

The wire format for streaming agent conversations. NDJSON frames with progressive rendering, stream multiplexing, and extensible control frames.

```
┌─────────────────────────────────────────┐
│       Message Semantics Layer           │  What messages mean
│   (user, agent, tool_call, etc.)        │  → spec/messages.md
└─────────────────────────────────────────┘
                   ↕
┌─────────────────────────────────────────┐
│      HTTP Protocol Layer                │  How to manage threads
│  (REST API, WebSocket transport)        │  → spec/http.md
└─────────────────────────────────────────┘
                   ↕
┌─────────────────────────────────────────┐
│        Framing Layer (core)             │  How bytes hit the wire
│   (NDJSON, streaming, multiplexing)     │  → spec/framing.md
└─────────────────────────────────────────┘
                   ↕
┌─────────────────────────────────────────┐
│        Extensions                       │  Application-specific ops
│   (sync, subscribe, filters, ...)       │  → spec/extensions/
└─────────────────────────────────────────┘
```

### [Framing Layer](spec/framing.md) — `Timbal/1.0`

The foundation. Defines two frame types:

- **Message frames** (`i` field) — Start, append, and set frames for streaming messages. ULID-based ordering. Stream multiplexing via `s` field.
- **Control frames** (`c` field) — Typed extension point for application-specific operations. Core only mandates `error`.

Transport-agnostic — works over WebSocket, SSE, HTTP chunked, or stdio.

### [HTTP Protocol Layer](spec/http.md) — `Timbal HTTP/1.0`

RESTful API for managing agent conversation threads. Clients connect via WebSocket for streaming, POST to create threads and send messages. Thread IDs are client-determined UUIDs.

### [Message Semantics Layer](spec/messages.md) — `Timbal Messages/1.0`

Standard message types for agentic conversations: `user`, `agent`, `tool_call`, `tool_result`, `thinking`, plus multi-agent extensions (`status`, `error`, `agent_complete`, `agent_message`).

### Extensions

Extensions define control frame types for application-specific behavior. The core spec provides the extension point; extensions fill it in.

- **[Sync Extension](spec/extensions/sync.md)** — History synchronization, reconnection with timestamp cursors, stream subscribe/unsubscribe. The standard pattern for WebSocket-based real-time apps.

## Quick Example

A complete agent conversation over the wire:

```
← {"i":"01J...001","t":"...","v":{"type":"user","content":"What's the weather in SF?"}}
← {"i":"01J...002","m":{"type":"agent","sender":"weather-bot"}}
← {"i":"01J...002","a":"It's "}
← {"i":"01J...002","a":"65°F and sunny "}
← {"i":"01J...002","a":"in San Francisco!"}
← {"i":"01J...002","t":"...","v":{"type":"agent","content":"It's 65°F and sunny in San Francisco!","sender":"weather-bot"}}
```

**What's happening:**
1. User message arrives complete (set frame with `v`)
2. Agent response starts streaming (start frame with `m` metadata)
3. Text arrives progressively (append frames with `a`)
4. Response finalizes (set frame with complete `v`)

The client can render immediately after the start frame — metadata tells it who's speaking before any content arrives.

## Key Design Decisions

- **ULID ordering** — Messages are sorted by their ULID identifiers. No sequence numbers needed.
- **Metadata-first rendering** — Start frames carry metadata (type, sender), enabling UI rendering before content streams.
- **Two streaming modes** — Text mode (metadata + content appends) for chat, object mode (JSON fragment appends) for structured data.
- **Stream multiplexing** — A single connection can carry multiple independent streams (channels, threads, event feeds) via the `s` field.
- **Core vs extensions** — The framing layer defines the universal data format. Application-specific operations (sync, subscribe, filters) are control frame extensions — not baked into the core.
- **Transport-agnostic framing** — The same NDJSON frames work over WebSocket, SSE, HTTP chunked, or stdio pipes.

## Status

**Draft** — This specification is under active development. We're iterating on it based on production usage.

## License

[MIT](LICENSE)
