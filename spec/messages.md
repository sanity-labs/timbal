# Timbal Message Semantics (Timbal Messages/1.0)

**tl;dr** — Standard message types for agentic conversations: user, agent, tool_call, tool_result, and thinking. Each type defines metadata and content schemas for use with Timbal framing. Supports streaming with immediate renderability. Optional `sender` field enables multi-actor threads.

## 1. Overview

This specification defines **message semantics** for agentic conversations transported via Timbal framing. It standardizes the most common message patterns:

* **User messages**: Input from humans
* **Agent messages**: Responses from the agent
* **Tool calls**: Agent requests to execute tools
* **Tool results**: Outcomes of tool execution
* **Thinking**: Agent's reasoning process (optional visibility)

All message types use the Timbal framing layer for streaming and synchronization.

## 2. Architecture

```
┌─────────────────────────────────────────┐
│         Application Logic               │
│  (Agent behavior, tool execution, etc)  │
└─────────────────────────────────────────┘
                   ↕
┌─────────────────────────────────────────┐
│       Message Semantics Layer           │  ← This specification
│   (user, agent, tool_call, etc.)        │
└─────────────────────────────────────────┘
                   ↕
┌─────────────────────────────────────────┐
│        Timbal Framing Layer             │
│   (NDJSON streaming, sync, ordering)    │
└─────────────────────────────────────────┘
```

## 3. Message Type Overview

### Core Types

| Type | Direction | Streamable | Description |
|------|-----------|------------|-------------|
| `user` | Human → Agent | No | User input |
| `agent` | Agent → Human | Yes | Agent response |
| `tool_call` | Agent → System | Yes | Tool invocation request |
| `tool_result` | System → Agent | No | Tool execution result |
| `thinking` | Agent internal | Yes | Reasoning process |

### Multi-Agent Extensions

| Type | Direction | Streamable | Description |
|------|-----------|------------|-------------|
| `status` | Agent → UI | No | Transient status indicator |
| `error` | Agent → Human | No | User-facing error |
| `agent_complete` | Agent → Agent | No | Sub-agent completion signal |
| `agent_message` | Agent → Agent | No | Inter-agent communication |

## 4. Common Structure

All messages use Timbal text streaming mode with metadata. The final value follows this structure:

```json
{
  "type": "<message_type>",
  "content": "<text content>",
  "sender": "<optional sender identifier>",
  ...additional fields per type
}
```

### 4.1 Sender Field

All message types support an optional `sender` field:

* `sender` (optional): Identifier for the message author (e.g., user ID, agent callsign)

**Single-actor threads**: The `sender` field MAY be omitted when there is only one user and one agent. The sender is implied by the message type (`user` messages come from the user, `agent` messages come from the agent).

**Multi-actor threads**: The `sender` field is REQUIRED when multiple users or multiple agents participate in the same thread. This enables:
* Distinguishing which user sent a message
* Distinguishing which agent generated a response
* Proper attribution in UI rendering

**Framing pattern** (see framing.md for details):
```
{"i":"<ulid>","m":{"type":"<type>","sender":"<id>",...}}  ← Start frame (sender in metadata)
{"i":"<ulid>","a":"<text>"}                               ← Append frames (streaming)
{"i":"<ulid>","t":"<timestamp>","v":{...complete}}        ← Set frame (finalize)
```

When streaming, the `sender` SHOULD be included in the start frame metadata to enable immediate UI rendering with proper attribution.

## 5. Message Types

### 5.1 User Message

Human input to the agent.

**Metadata fields:**
* `type` (required): `"user"`
* `sender` (optional): User identifier

**Final value:**
```json
{
  "type": "user",
  "content": "What's the weather in San Francisco?",
  "sender": "alice"
}
```

**Typical framing** (non-streamed, sent as complete):
```
{"i":"01J...","t":"2025-01-15T14:30:00.000Z","v":{"type":"user","content":"What's the weather in San Francisco?","sender":"alice"}}
```

**Notes:**
* User messages are typically sent complete (no streaming)
* The `content` field contains the user's text input
* The `sender` field identifies which user sent the message (required in multi-user threads)

### 5.2 Agent Message

Agent response to the user.

**Metadata fields:**
* `type` (required): `"agent"`
* `sender` (optional): Agent identifier (e.g., callsign)

**Final value:**
```json
{
  "type": "agent",
  "content": "The weather in San Francisco is currently 65°F and sunny.",
  "sender": "weather-bot"
}
```

**Typical framing** (streamed):
```
{"i":"01J...","m":{"type":"agent","sender":"weather-bot"}}
{"i":"01J...","a":"The weather in "}
{"i":"01J...","a":"San Francisco is "}
{"i":"01J...","a":"currently 65°F and sunny."}
{"i":"01J...","t":"2025-01-15T14:30:05.000Z","v":{"type":"agent","content":"The weather in San Francisco is currently 65°F and sunny.","sender":"weather-bot"}}
```

**Notes:**
* Content streams progressively for real-time display
* Metadata in start frame enables immediate UI rendering (show agent name before content arrives)
* The `sender` field identifies which agent generated the response (required in multi-agent threads)

### 5.3 Tool Call

Agent request to execute a tool.

**Metadata fields:**
* `type` (required): `"tool_call"`
* `toolCallId` (required): Unique identifier for this call (for correlating with result)
* `name` (required): Tool name
* `sender` (optional): Agent identifier making the call

**Final value:**
```json
{
  "type": "tool_call",
  "toolCallId": "call_abc123",
  "name": "get_weather",
  "arguments": {
    "location": "San Francisco, CA"
  },
  "sender": "weather-bot"
}
```

**Fields:**
* `toolCallId`: Unique identifier linking this call to its result
* `name`: The tool being invoked
* `arguments`: Tool arguments object
* `sender` (optional): Which agent made this tool call

**Typical framing** (streamed arguments):
```
{"i":"01J...","m":{"type":"tool_call","toolCallId":"call_abc123","name":"get_weather","sender":"weather-bot"}}
{"i":"01J...","a":"{\"location\": \"San"}
{"i":"01J...","a":" Francisco, CA\"}"}
{"i":"01J...","t":"2025-01-15T14:30:01.000Z","v":{"type":"tool_call","toolCallId":"call_abc123","name":"get_weather","arguments":{"location":"San Francisco, CA"},"sender":"weather-bot"}}
```

**Notes:**
* Arguments stream as JSON fragments in append frames
* Clients can use partial JSON parsing during streaming for progressive display
* Final `arguments` contains the parsed object

### 5.4 Tool Result

Result of tool execution.

**Metadata fields:**
* `type` (required): `"tool_result"`
* `toolCallId` (required): References the tool_call this responds to
* `status` (required): `"success"` | `"error"`
* `sender` (optional): Identifier for the system/service that executed the tool

**Final value (success):**
```json
{
  "type": "tool_result",
  "toolCallId": "call_abc123",
  "status": "success",
  "output": {
    "temperature": 65,
    "condition": "sunny"
  }
}
```

**Final value (error):**
```json
{
  "type": "tool_result",
  "toolCallId": "call_abc123",
  "status": "error",
  "error": "API rate limit exceeded"
}
```

**Typical framing** (non-streamed):
```
{"i":"01J...","t":"2025-01-15T14:30:02.000Z","v":{"type":"tool_result","toolCallId":"call_abc123","status":"success","output":{"temperature":65,"condition":"sunny"}}}
```

**Notes:**
* Tool results are typically sent complete (no streaming)
* `toolCallId` links this result to the originating tool_call
* `output` contains the tool's return value (success case)
* `error` contains the error message (error case)

### 5.5 Thinking

Agent's reasoning process, optionally visible to users.

**Metadata fields:**
* `type` (required): `"thinking"`
* `sender` (optional): Agent identifier

**Final value:**
```json
{
  "type": "thinking",
  "content": "The user is asking about weather. I should use the get_weather tool with San Francisco as the location parameter.",
  "sender": "weather-bot"
}
```

**Typical framing** (streamed):
```
{"i":"01J...","m":{"type":"thinking","sender":"weather-bot"}}
{"i":"01J...","a":"The user is asking about weather. "}
{"i":"01J...","a":"I should use the get_weather tool..."}
{"i":"01J...","t":"2025-01-15T14:30:00.500Z","v":{"type":"thinking","content":"The user is asking about weather. I should use the get_weather tool with San Francisco as the location parameter.","sender":"weather-bot"}}
```

**Notes:**
* Thinking messages reveal the agent's reasoning process
* Visibility is implementation-specific (may be hidden in production, shown in debug mode)
* Often rendered differently in UI (e.g., collapsed, dimmed, or in a separate panel)
* The `sender` field identifies which agent is thinking (useful in multi-agent threads)

## 6. Multi-Agent Extensions

These message types support agent lifecycle, inter-agent communication, and operational status in multi-agent systems.

### 6.1 Status

Transient status indicator for ongoing work. Typically replaced or deleted when work completes.

**Metadata fields:**
* `type` (required): `"status"`
* `state` (required): Status state (e.g., `"thinking"`, `"working"`, `"searching"`)
* `sender` (optional): Agent identifier

**Final value:**
```json
{
  "type": "status",
  "state": "searching",
  "detail": "Checking weather API...",
  "sender": "weather-bot"
}
```

**Fields:**
* `state`: Current activity state
* `detail` (optional): Human-readable description
* `sender` (optional): Which agent is reporting status

**Notes:**
* Status messages are ephemeral UI hints, not part of the logical conversation
* Typically deleted (set to `null`) when work completes
* State changes atomically (no streaming)

### 6.2 Error

User-facing error (distinct from `tool_result` errors which are internal to the agent loop).

**Final value:**
```json
{
  "type": "error",
  "content": "Failed to connect to weather service",
  "code": "service_unavailable",
  "recoverable": true,
  "sender": "weather-bot"
}
```

**Fields:**
* `content`: Human-readable error message
* `code` (optional): Machine-readable error code
* `recoverable` (optional): Whether the operation can be retried
* `sender` (optional): Which agent or system generated the error

**Notes:**
* Displayed prominently to users
* May trigger retry UI or end the conversation
* Different from tool errors which the agent handles internally

### 6.3 Agent Complete

Signals that a spawned sub-agent finished with a result.

**Final value:**
```json
{
  "type": "agent_complete",
  "agentId": "550e8400-e29b-41d4-a716-446655440000",
  "result": {
    "summary": "Weather data retrieved successfully",
    "data": { "temp": 65, "condition": "sunny" }
  }
}
```

**Fields:**
* `agentId`: Thread ID of the completed sub-agent
* `result`: Result payload from the sub-agent

**Notes:**
* Enables parent agents to coordinate work and collect results
* Result schema is application-specific

### 6.4 Agent Message

Inter-agent communication. One agent sends a message to another.

**Final value:**
```json
{
  "type": "agent_message",
  "senderId": "550e8400-e29b-41d4-a716-446655440000",
  "payload": {
    "action": "delegate",
    "task": "fetch weather for San Francisco"
  },
  "replyTo": "01J..."
}
```

**Fields:**
* `senderId`: Thread ID of the sending agent
* `payload`: Message content (string or object, schema determined by agents)
* `replyTo` (optional): Message ID this is responding to

**Notes:**
* Enables actor-model coordination patterns
* Messages are queued and delivered as events
* Payload schema is application-specific

## 7. Message Ordering

Messages follow standard Timbal ordering rules (ULID sort). A typical exchange:

```
01J...001  user         "What's the weather?"
01J...002  thinking     "I should call the weather API..."
01J...003  tool_call    get_weather(location: "San Francisco")
01J...004  tool_result  {"temperature": 65, "condition": "sunny"}
01J...005  agent        "It's 65°F and sunny in San Francisco."
```

**Interleaving**: Tool calls and thinking may be interleaved with agent content when agents work on multiple tasks concurrently.

## 8. Streaming Behavior

### 8.1 Which Messages Stream?

| Type | Typically Streams | Reason |
|------|-------------------|--------|
| `user` | No | User input is complete before sending |
| `agent` | Yes | Progressive response display |
| `tool_call` | Yes | Arguments may be generated progressively |
| `tool_result` | No | Execution completes before sending |
| `thinking` | Yes | Reveals reasoning in real-time |
| `status` | No | State changes atomically |
| `error` | No | Error is complete when raised |
| `agent_complete` | No | Completion is atomic |
| `agent_message` | No | Messages are atomic |

### 8.2 Client Rendering During Streaming

For streaming messages, clients construct the display value as:

```javascript
// During streaming (after start frame, before set frame)
value = { ...metadata, content: buffer }

// Example for agent:
// metadata = { type: "agent" }
// buffer = "Hello wor"
// value = { type: "agent", content: "Hello wor" }
```

This follows the Timbal framing text streaming mode (see framing.md §6.1).

## 9. Extending Message Types

Implementations MAY define additional message types. Custom types SHOULD:

* Use a namespaced type value (e.g., `"x-myapp-status"`)
* Follow the same metadata/content structure
* Document their streaming behavior

**Example custom type:**
```json
{
  "type": "x-myapp-status",
  "content": "Processing request...",
  "progress": 0.5
}
```

## 10. Examples

### 10.1 Complete Conversation Flow

```
← {"i":"01J...001","t":"2025-01-15T14:30:00.000Z","v":{"type":"user","content":"What's the weather in SF?"}}
← {"i":"01J...002","m":{"type":"thinking"}}
← {"i":"01J...002","a":"User wants weather info. I'll call get_weather."}
← {"i":"01J...002","t":"2025-01-15T14:30:00.200Z","v":{"type":"thinking","content":"User wants weather info. I'll call get_weather."}}
← {"i":"01J...003","m":{"type":"tool_call","toolCallId":"call_1","name":"get_weather"}}
← {"i":"01J...003","a":"{\"location\":\"San Francisco\"}"}
← {"i":"01J...003","t":"2025-01-15T14:30:00.300Z","v":{"type":"tool_call","toolCallId":"call_1","name":"get_weather","arguments":{"location":"San Francisco"}}}
← {"i":"01J...004","t":"2025-01-15T14:30:01.000Z","v":{"type":"tool_result","toolCallId":"call_1","status":"success","output":{"temp":65,"condition":"sunny"}}}
← {"i":"01J...005","m":{"type":"agent","sender":"weather-bot"}}
← {"i":"01J...005","a":"It's "}
← {"i":"01J...005","a":"65°F and sunny "}
← {"i":"01J...005","a":"in San Francisco!"}
← {"i":"01J...005","t":"2025-01-15T14:30:02.000Z","v":{"type":"agent","content":"It's 65°F and sunny in San Francisco!","sender":"weather-bot"}}
```

### 10.2 Tool Error Handling

```
← {"i":"01J...003","t":"2025-01-15T14:30:01.000Z","v":{"type":"tool_result","toolCallId":"call_1","status":"error","error":"Service temporarily unavailable"}}
← {"i":"01J...004","m":{"type":"agent","sender":"weather-bot"}}
← {"i":"01J...004","a":"I'm sorry, I couldn't fetch the weather data. Please try again later."}
← {"i":"01J...004","t":"2025-01-15T14:30:02.000Z","v":{"type":"agent","content":"I'm sorry, I couldn't fetch the weather data. Please try again later.","sender":"weather-bot"}}
```

### 10.3 Multiple Tool Calls

```
← {"i":"01J...003","m":{"type":"tool_call","toolCallId":"call_1","name":"get_weather"}}
← {"i":"01J...004","m":{"type":"tool_call","toolCallId":"call_2","name":"get_time"}}
← {"i":"01J...003","a":"{\"location\":\"SF\"}"}
← {"i":"01J...004","a":"{\"timezone\":\"America/Los_Angeles\"}"}
← {"i":"01J...003","t":"...","v":{...}}
← {"i":"01J...004","t":"...","v":{...}}
← {"i":"01J...005","t":"...","v":{"type":"tool_result","toolCallId":"call_1",...}}
← {"i":"01J...006","t":"...","v":{"type":"tool_result","toolCallId":"call_2",...}}
```

## 11. Conformance

A conformant implementation of Timbal Messages/1.0:

* Uses the five core message types (`user`, `agent`, `tool_call`, `tool_result`, `thinking`)
* Follows the metadata and value schemas defined in §5
* Uses Timbal framing text streaming mode with `type` in metadata
* Links tool_call and tool_result via `toolCallId`
* Ignores unknown fields for forward compatibility
* MAY include `sender` field on any message type (REQUIRED for multi-actor threads per §4.1)
* MAY support multi-agent extensions (`status`, `error`, `agent_complete`, `agent_message`) per §6
* MAY support additional custom message types (§9)

---

**Version**: Timbal Messages/1.0
**Status**: Draft
