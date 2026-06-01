# Human Handoff Integration

## Session bootstrap (Frontend / Laravel → FastAPI)

Before any Socket.IO, handoff, or RTC call, create the chatbot session through FastAPI.

```
POST /api/subsidy-chatbot/session
X-Shared-Key: {shared_key}

{
  "chatbot_id": "uuid",   // required: chat_bots.id
  "client_id": "uuid"     // optional: users.id
}
```

Response:

```
{
  "id": "uuid",           // session_id used everywhere else
  "chatbot_id": "uuid",
  "status": "active",
  "started_at": "iso8601",
  "last_activity_at": "iso8601"
}
```

Important:
- `chatbot_id` is required only when creating the session
- after the session exists, all later FastAPI routes, Socket.IO events, LiveKit/RTC calls, and Laravel webhooks continue using `session_id`
- FastAPI resolves `chatbot_id` and `organization_id` from `chatbot_sessions`, so Laravel does not need to resend them on handoff or agent events

## Part 1: Chat Flow (User ↔ Bot)

Before handoff, the user talks to the bot via Socket.IO. All chat events use the same room as handoff events — the plain `session_id` UUID.

### 1. Join the session room

Must be called before sending any messages. Validates the session exists and joins the room.

```
emit: join_chatbot_session
{
  "session_id": "uuid"
}
```

**Receive (self only):**

```
event: chatbot_session_joined
{
  "session_id": "uuid"
}
```

### 2. Send a message

```
emit: chatbot_message
{
  "session_id": "uuid",
  "message": "string"
}
```

### 3. Receive the response

Tokens are streamed as they are generated. The final event fires once the full response is ready.

**Per-token (during generation):**

```
event: chatbot_token
{
  "session_id": "uuid",
  "token": "string"
}
```

**Final (after generation completes):**

```
event: chatbot_response
{
  "session_id": "uuid",
  "response": "string",           // full assembled response
  "is_casual": false,
  "confidence_tier": "high" | "medium" | "low" | null,
  "confidence_score": 0.82,       // 0.0–1.0; null for casual turns
  "citations": [...] | null,
  "handoff_triggered": true | false,
  "handoff_reason": "no_hits" | "weak_retrieval" | null
}
```

Use `chatbot_token` events to render the streaming bubble and `chatbot_response` to finalise it with citations and confidence. Discard any in-progress streaming bubble if `is_casual` is true — no badge or citations for small talk.

---

## Part 2: Outgoing Webhooks (FastAPI → Laravel)

FastAPI POSTs to `LARAVEL_WEBHOOK_URL` with header `X-Shared-Key`. These are fire-and-forget notifications — FastAPI does not wait for a response. Laravel should use them to update its own state and drive agent assignment.

### Handoff Requested

Fired immediately when a user emits `request_agent_handoff`. Laravel should use `session_id` to look up the session, identify the user, and resolve the linked chatbot/org if needed. Once an agent is assigned, Laravel should instruct the agent app to call `join_handoff_session` via Socket.IO (see Part 2).

```
POST {LARAVEL_WEBHOOK_URL}
X-Shared-Key: {shared_key}

{
  "session_id": "uuid"
}
```

Resolved and cancelled state changes are handled entirely by FastAPI (`HandoffService.resolve` / `HandoffService.cancel`). Laravel does not receive separate webhooks for those events — use the DB as the source of truth for handoff status.

---

## Part 2: Agent Socket.IO

The agent app connects to FastAPI's Socket.IO server after Laravel assigns them to a session. Laravel is responsible for passing the `session_id` and `agent_id` to the agent app — the agent app does not discover these itself.

### 1. Join the handoff session

Called once after Laravel notifies the agent app of the assignment. This claims the handoff record in the DB and admits the agent into the shared room.

```
emit: join_handoff_session
{
  "session_id": "uuid",
  "agent_id": "uuid",      // users.id — not agents.id
  "shared_key": "string"   // AGENT_WS_SHARED_KEY — shared secret configured on both sides
}
```

**Receive (private to agent only):** sent immediately after joining, contains the full prior conversation so the agent has context before typing.

```
event: handoff_joined
{
  "session_id": "uuid",
  "agent_id": "uuid",
  "history": [
    {
      "role": "client" | "assistant" | "agent",
      "content": "string",
      "timestamp": "iso8601"
    }
  ]
}
```

**Receive (broadcast to the whole room):** sent at the same time as `handoff_joined` — this is what the user sees to know an agent has arrived.

```
event: handoff_accepted
{
  "session_id": "uuid",
  "agent_id": "uuid",
  "agent_name": "string"   // display name pulled from users table
}
```

### 2. Send a message

Sent each time the agent types a reply. The server saves the message to the DB and broadcasts it to the room.

```
emit: agent_message
{
  "session_id": "uuid",
  "agent_id": "uuid",
  "message": "string"
}
```

**Receive (broadcast to room, including the agent):**

```
event: agent_message_received
{
  "session_id": "uuid",
  "agent_id": "uuid",
  "message": "string",
  "timestamp": "iso8601"
}
```

### 3. Receive user messages

User messages during an active handoff are forwarded to the agent's room in real time. The user does not need to do anything differently — FastAPI detects the active handoff and routes automatically.

```
event: user_message_received
{
  "session_id": "uuid",
  "message": "string",
  "timestamp": "iso8601"
}
```

### 4. End the conversation

Emitted by the agent when the issue is resolved. This closes the handoff record in the DB and re-enables the bot for the user.

```
emit: resolve_handoff
{
  "session_id": "uuid"
}
```

**Receive (broadcast to room):**

```
event: handoff_resolved
{
  "session_id": "uuid"
}
```

---

## Part 3: User Socket.IO

The user must have already joined the session room via `join_chatbot_session` (Part 1) before any of these events will work — all handoff events use the same room.

### 1. Request an agent

Emitted when the user taps the "Talk to Agent" button. The frontend should disable the button immediately after emitting to prevent duplicate requests.

```
emit: request_agent_handoff
{
  "session_id": "uuid"
}
```

**Receive (self only):** confirms the request was received and a notification has been sent to Laravel. Show a waiting state in the UI.

```
event: handoff_triggered
{
  "session_id": "uuid",
  "client_name": "string",   // the requesting user's display name
  "message": "string"        // e.g. "Your request has been received."
}
```

If no agent joins within 45 seconds, FastAPI cancels the wait and emits:

```
event: handoff_timeout
{
  "session_id": "uuid"
}
```

Show "No agent is available right now. Please try again later." and re-enable the request button. The handoff record stays in the DB — it is not automatically retried.

### 2. Cancel a pending handoff

Emitted when the user changes their mind before an agent joins. This cancels the pending handoff request and emits a user-only confirmation event.

```
emit: cancel_handoff
{
  "session_id": "uuid"
}
```

**Receive (self only):**

```
event: handoff_cancelled
{
  "session_id": "uuid"
}
```

### 3. Bot suggests handoff (low-confidence turns)

When the bot cannot confidently answer, `chatbot_response` includes a flag. The frontend should show a prompt asking the user if they want to speak to an agent. If they accept, emit `request_agent_handoff` (same as step 1).

```
event: chatbot_response
{
  ...
  "handoff_triggered": true,
  "handoff_reason": "no_hits" | "weak_retrieval"
}
```

`handoff_reason` values:
- `"no_hits"` — bot found no relevant content; show "Talk to Agent" prompt
- `"weak_retrieval"` — bot found something but confidence was low; show "Talk to Agent" prompt

### 4. Agent joins (or timeout)

When an agent accepts the handoff, this is broadcast to the entire room — both the user and the agent receive it. Use `agent_name` to show a message like "田中 誠 has joined the conversation."

```
event: handoff_accepted
{
  "session_id": "uuid",
  "agent_id": "uuid",
  "agent_name": "string"
}
```

### 5. Exchange messages

The user continues sending messages exactly as before via `chatbot_message`. No change is needed on the frontend — FastAPI detects the active handoff and routes the message to the agent instead of the bot.

**Receive agent messages:**

```
event: agent_message_received
{
  "session_id": "uuid",
  "agent_id": "uuid",
  "message": "string",
  "timestamp": "iso8601"
}
```

### 6. Handoff ends

Emitted when the agent resolves the session. Show a message like "The conversation with the agent has ended." and re-enable the "Talk to Agent" button if the user may want to request again.

```
event: handoff_resolved
{
  "session_id": "uuid"
}
```

---

## Event Reference

| Event | Direction | Who sees it |
|---|---|---|
| `join_handoff_session` | Agent → Server | — |
| `handoff_joined` | Server → Agent | Agent only |
| `handoff_accepted` | Server → Room | Agent + User |
| `agent_message` | Agent → Server | — |
| `agent_message_received` | Server → Room | Agent + User |
| `user_message_received` | Server → Room | Agent + User |
| `resolve_handoff` | Agent → Server | — |
| `request_agent_handoff` | User → Server | — |
| `cancel_handoff` | User → Server | — |
| `handoff_triggered` | Server → User | User only |
| `handoff_cancelled` | Server → User | User only |
| `handoff_timeout` | Server → User | User only |
| `handoff_resolved` | Server → Room | Agent + User |

**Room:** all events broadcast to the room use the plain `session_id` UUID as the room name.

**agent_id:** always `users.id`, not `agents.id`.

**Agent auth:** `join_handoff_session` requires `shared_key` matching `AGENT_WS_SHARED_KEY` configured on the FastAPI server. This is a temporary shared secret — will be replaced with PAT validation when Laravel implements personal access tokens.

**Timeout:** configurable via `HANDOFF_AGENT_TIMEOUT_SECONDS` (default 45s). Timer starts when `handoff_triggered` is emitted and cancels automatically if the agent joins in time.
