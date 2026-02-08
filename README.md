# Timbal

**A streaming protocol for real-time agent conversations.**

Timbal defines how messages are encoded, streamed, and synchronized in agentic chat systems. It's designed for progressive rendering, multi-agent coordination, and reliable reconnection.

## Protocol Layers

Timbal is organized into three layers, each building on the one below:

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
│        Framing Layer                    │  How bytes hit the wire
│   (NDJSON streaming, sync, ordering)    │  → spec/framing.md
└─────────────────────────────────────────┘
```

### [Framing Layer](spec/framing.md) — `Timbal/1.0`

The foundation. Defines NDJSON frame encoding, message streaming (start/append/set), ULID-based ordering, and sync/reconnection. Transport-agnostic — works over WebSocket, SSE, HTTP chunked, or stdio.

### [HTTP Protocol Layer](spec/http.md) — `Timbal HTTP/1.0`

RESTful API for managing agent conversation threads. Clients connect via WebSocket for streaming, POST to create threads and send messages. Thread IDs are client-determined UUIDs.

### [Message Semantics Layer](spec/messages.md) — `Timbal Messages/1.0`

Standard message types for agentic conversations: `user`, `agent`, `tool_call`, `tool_result`, `thinking`, plus multi-agent extensions (`status`, `error`, `agent_complete`, `agent_message`).

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
- **Append-only with sync** — Clients reconnect and request only what they missed via timestamp cursors.
- **Transport-agnostic framing** — The same NDJSON frames work over WebSocket, SSE, HTTP chunked, or stdio pipes.

## Status

**Draft** — This specification is under active development. We're iterating on it based on production usage.

## License

[MIT](LICENSE)
