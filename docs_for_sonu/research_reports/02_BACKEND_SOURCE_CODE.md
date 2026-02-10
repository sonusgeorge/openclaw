# Agent Report 2: Backend Source Code & API Layer

**Agent ID:** ab28d87
**Tokens Processed:** ~83,000
**Duration:** ~300 seconds

---

# OPENCLAW BACKEND SOURCE CODE - COMPLETE TECHNICAL SUMMARY

## EXECUTIVE SUMMARY

OpenClaw is a production-grade agentic AI platform built on Node.js/TypeScript that orchestrates multi-LLM agent execution across 15+ messaging channels. The system implements:

- Real-time WebSocket gateway (80+ API methods)
- Multi-provider LLM support (OpenAI, Anthropic, AWS Bedrock, Google Gemini, etc.)
- Plugin-based extensibility system
- Sophisticated auth profile management with fallback chains
- Enterprise-grade features (context pruning, execution approvals, session compaction)
- Distributed node/mobile client coordination

---

## SECTION 1: GATEWAY SERVER ARCHITECTURE

### 1.1 Core Server Entry Point
**File**: `/src/gateway/server.impl.ts` (600+ lines)

**Main Function**: `startGatewayServer(port = 18789, opts: GatewayServerOptions)`

**Initialization Sequence**:
1. Load and validate configuration
2. Handle legacy config migration
3. Load plugins and channels
4. Create runtime state (HTTP/WebSocket servers)
5. Initialize cron service
6. Start discovery service (Bonjour/mDNS)
7. Setup Canvas host (rendering service)
8. Attach WebSocket handlers
9. Setup maintenance timers

### 1.2 HTTP Server Setup
**File**: `/src/gateway/server-http.ts` (250+ lines)

**Route Priority**:
1. Hooks API: `POST /hooks/wake`, `/hooks/agent` (token-based)
2. Canvas/A2UI: `GET/POST /a2ui/*`, `/canvas/*`
3. OpenAI API: `POST /v1/chat/completions` (SSE streaming)
4. OpenResponses API: `POST /v1/responses`
5. Tools Invoke: `POST /tools/invoke`
6. Control UI: `GET /*` (static assets)
7. 404 Not Found

### 1.3 WebSocket Server
**File**: `/src/gateway/server/ws-connection.ts` (280+ lines)

**Connection Lifecycle**:
1. Server sends challenge with nonce
2. Client responds with connect message
3. Server validates auth and registers client
4. Message handler attached
5. On disconnect: cleanup

**Client Structure**:
```typescript
{
  connId: string;                // UUID
  socket: WebSocket;
  clientIp: string;
  connect: {
    clientName: string;
    clientDisplayName: string;
    mode: "frontend" | "backend";
    role: "operator" | "node";
    scopes: string[];
  };
  presenceKey?: string;
  subscriptions?: Set<string>;
}
```

---

## SECTION 2: GATEWAY API ENDPOINTS (80+)

### Complete Method List by Category

**Connection & Auth** (3): connect, device.token.rotate, device.token.revoke

**Health & Status** (5): health, logs.tail, status, usage.status, usage.cost

**Agent Execution** (7): agent, agent.wait, agent.identity.get, agents.list, agents.files.list, agents.files.get, agents.files.set

**Chat (WebChat)** (3): chat.history, chat.send, chat.abort

**Sessions** (7): sessions.list, sessions.preview, sessions.patch, sessions.reset, sessions.delete, sessions.compact, last-heartbeat

**Configuration** (5): config.get, config.set, config.apply, config.patch, config.schema

**Models** (1): models.list

**Channels** (2): channels.status, channels.logout

**Messaging** (1): send

**Tools & Skills** (3): skills.status, skills.bins, skills.install, skills.update

**Voice & TTS** (7): voicewake.get/set, tts.status/providers/enable/disable/convert

**Cron** (6): cron.list/status/add/update/remove/run/runs

**Node Management** (11): node.list/describe/rename/invoke/invoke.result/event, node.pair.request/list/approve/reject/verify

**Device Management** (4): device.pair.list/approve/reject, device.token.rotate/revoke

**Approvals** (5): exec.approval.request/resolve, exec.approvals.get/set, exec.approvals.node.get/set

**Wizard** (4): wizard.start/next/cancel/status

**System** (3): system-presence, system-event, talk.mode

### Authorization Model

**Scopes**:
- `operator.admin` - Full access
- `operator.read` - Read-only
- `operator.write` - Send/execute
- `operator.pairing` - Device pairing
- `operator.approvals` - Approval management

**Method -> Scope Mapping**:
- READ_METHODS (22 methods) -> operator.read OR write OR admin
- WRITE_METHODS (13 methods) -> operator.write OR admin
- ADMIN_METHODS (prefix-based) -> operator.admin
- PAIRING_METHODS (8 methods) -> operator.pairing OR admin
- APPROVAL_METHODS (2 methods) -> operator.approvals OR admin

---

## SECTION 3: LLM INTEGRATION SYSTEM

### 3.1 Model Resolution Pipeline

**File**: `/src/agents/pi-embedded-runner/run.ts` (500+ lines)

**Execution Flow**:
1. Model Lookup -> resolveModel()
2. Context Window Guard -> evaluateContextWindowGuard()
3. Auth Profile Selection -> resolveAuthProfileOrder()
4. API Key Retrieval Loop (per candidate profile)
5. Execution Attempt -> runEmbeddedAttempt()
6. Error Handling & Fallback -> classify error, cooldown, try next

### 3.2 Supported LLM Providers

| Provider | Models | Auth | Features |
|----------|--------|------|----------|
| **OpenAI** | gpt-4o, gpt-4-turbo, o1, o3 | API Key | Vision, Functions, Streaming |
| **Anthropic** | claude-3-5-sonnet, claude-3-opus, claude-3-haiku | API Key | Vision, Functions, Extended Thinking |
| **AWS Bedrock** | claude-3-*, mistral-*, titan-* | AWS SDK | Vision, Functions, Streaming |
| **Google Gemini** | gemini-pro, gemini-2.0-flash | API Key | Vision, Functions, Streaming |
| **GitHub Copilot** | gpt-4, gpt-3.5-turbo | GitHub Token | Token exchange required |
| **OpenRouter** | 200+ models | API Key | Multi-provider aggregator |
| **Ollama** | Any local model | HTTP endpoint | On-device execution |

**Thinking Levels**: off, on, extended, deep, ultra, x-high

### 3.3 Auth Profile System

**Profile Types**: api-key, oauth, token, aws-sdk

**Resolution Priority**:
1. Explicit profileId
2. Provider auth override
3. AWS SDK default chain
4. Auth profile order
5. Environment variables

**Cooldown System**: Rate limit 60s, auth failure skip, session override support

### 3.4 Model Fallback Chain

Fallback triggers: rate-limit, on-error, auth, context-overflow, always

---

## SECTION 4: AGENT EXECUTION & ORCHESTRATION

### 4.1 Agent Command Entry Point
**File**: `/src/commands/agent.ts` (500+ lines)

**Execution Steps**:
1. Validate message and routing
2. Load agent workspace
3. Resolve session
4. Update skills snapshot
5. Build workspace skill snapshot
6. Select model with fallback chain
7. Register agent run context
8. Apply thread safety via lanes
9. Execute agent (runEmbeddedPiAgent)
10. Handle result (deliver, error, timeout)
11. Update session store

### 4.2 Session Management

**Storage**: `~/.openclaw/sessions.json`

**Session Key Format**: `{channel}:{accountId}:{conversationId}`

### 4.3 Concurrent Execution (Lane System)

Per-session serialization to prevent context conflicts. Independent sessions run in parallel.

---

## SECTION 5: MIDDLEWARE, AUTH & SECURITY

### 5.1 Gateway Authentication Modes
Public, Local Loopback, Bearer Token, Tailscale, OAuth, Password (legacy)

### 5.2 WebSocket Handshake
1. Client connects
2. Server sends challenge with nonce
3. Client sends connect with token/password/clientName
4. Server validates and responds
5. 30s timeout

### 5.3 Execution Approval Workflow
Tool invocation triggers policy check -> If approval required -> Generate approval ID -> Wait for user response (5 min timeout) -> Execute or reject

---

## SECTION 6: WEBSOCKET & STREAMING

### 6.1 Message Protocol

Frame types: request, response, event, subscribe/unsubscribe

### 6.2 Agent Event Streaming

Event types: text-delta (150ms batched), tool-invocation, tool-result, error, complete

**Delta Batching**: 80-90% reduction in WebSocket messages

### 6.3 Block Streaming (Chunking)

Config: minChars 800, maxChars 1200, idleMs 1000
Respects per-channel text limits

---

## SECTION 7: DATABASE & STORAGE

### 7.1 Session Storage
Location: `~/.openclaw/sessions.json`

### 7.2 Auth Profile Store
Location: `~/.openclaw/.auth/profiles.json` (AES-256-GCM encrypted)

### 7.3 Memory/Vector Database
Location: `~/.openclaw/memory/{agentId}.sqlite`
Uses sqlite-vec for vector search
Hybrid search: 70% vector, 30% BM25, 4x candidate multiplier

---

## SECTION 8: CHANNEL INTEGRATION (15+)

Channels: WhatsApp (Baileys), Telegram (grammY), Signal (libsignal FFI), Discord, Slack (Bolt), Teams (Graph API), Google Chat, Matrix, iMessage (BlueBubbles), Feishu, LINE, Web, Custom

**Channel Manager Lifecycle**: Load config -> For each enabled channel -> Load runtime -> Create monitor -> Start monitoring

**Delivery Policies**: allow, deny, require-approval

---

## SECTION 9: PLUGIN SYSTEM

### Plugin Loader Discovery
Sources: Bundled, Installed (npm), Workspace (user directory)

### Plugin API Capabilities
- addGatewayMethod()
- defineTool()
- on/off (event hooks)
- registerChannel()
- registerService()
- getStorage(namespace)
- getConfig/getConfigSchema()
- getLogger()

### Plugin Types
Gateway Handler, Tool, Hook, Channel

---

## SECTION 10: CONFIGURATION SYSTEM

**Root Sections**: agents, gateway, models, channels (whatsapp, telegram, etc.), plugins, memory, cron, commands, discovery

**Config Loading**: JSON5 format, Zod validation, runtime overrides via env vars, legacy migration support

---

## SECTION 11: ERROR HANDLING & RECOVERY

**Failover Reasons**: auth, rate_limit, context_overflow, billing, image_size, image_dimension, unknown

**Recovery Strategies**:
- Rate limit: 60s cooldown, next profile
- Auth: Mark failed, next profile
- Context overflow: Session compaction, retry
- Billing: Fail immediately

---

## SECTION 12: PERFORMANCE OPTIMIZATIONS

- Delta batching: 150ms window, 80-90% message reduction
- Slow client detection: Drop messages if bufferedAmount > threshold
- Version-based presence updates: Clients skip redundant updates

---

## KEY STATISTICS

- Total files: 500+
- Total lines: 100,000+ TypeScript
- Gateway methods: 80+
- Supported channels: 15+
- LLM providers: 8+
- Bundled plugins: 15+
- Message buffer drop: 80-90% reduction
- Rate limit cooldown: 60s
- Tool execution timeout: 30s
- Approval timeout: 5 minutes
