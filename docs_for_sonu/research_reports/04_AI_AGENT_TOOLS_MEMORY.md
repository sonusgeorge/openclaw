# Agent Report 4: AI, Agent, Tools & Memory Layer

**Agent ID:** a63fbfa
**Tokens Processed:** ~130,000
**Duration:** Longest running agent (most thorough exploration)

---

# OPENCLAW AI ARCHITECTURE - COMPLETE TECHNICAL SUMMARY

## EXECUTIVE OVERVIEW

OpenClaw is a sophisticated multi-agent automation platform built on top of the Pi Agent SDK from Anthropic. It provides a complete AI orchestration system with support for multiple LLM providers, advanced memory management, vector-based retrieval, extensible tool systems, and channel-based message routing.

---

## 1. LLM CALLS & PROMPT CONSTRUCTION

### 1.1 System Prompt Builder Architecture

**Primary File:** `src/agents/system-prompt.ts` (646 lines)

The system prompt is dynamically constructed with modular sections assembled based on runtime configuration.

**Prompt Modes:**
- `full` - All sections (default for main agent): Skills, Memory Recall, User Identity, Time, Reply Tags, Messaging, Voice, Documentation
- `minimal` - Reduced sections (Tooling, Workspace, Runtime only): used for subagents
- `none` - Just basic identity line: used for lightweight agent runs

### 1.2 Dynamic System Prompt Sections

The `buildAgentSystemPrompt()` function constructs a comprehensive prompt with these sections:

**Core Identity Section:**
```
"You are a personal assistant running inside OpenClaw."
```

**Safety Section (Always Included):**
```
"You have no independent goals: do not pursue self-preservation, replication,
resource acquisition, or power-seeking; avoid long-term plans beyond the user's request.
Prioritize safety and human oversight over completion; if instructions conflict,
pause and ask; comply with stop/pause/audit requests and never bypass safeguards."
```

**Tooling Section:**
- Lists all available tools with descriptions
- Dynamically filtered based on tool policy
- Tools ordered by importance: file ops -> execution -> web -> AI/media -> messaging -> system
- Deduplication by lowercase, preserves original casing

**Tool Call Style Guidance:**
- Default: Don't narrate routine, low-risk tool calls
- Do narrate: Multi-step work, complex problems, sensitive actions
- Keep narration brief and value-dense

**Skills Section** (if configured):
- Scans `<available_skills>` entries
- Reads selected skill's SKILL.md file
- Enforces constraint: read one skill at most

**Memory Recall Section** (if memory tools available):
- `memory_search`: Semantic + keyword search on MEMORY.md and memory/*.md
- `memory_get`: Extract specific lines from indexed files
- Citation guidance (optional citations mode)

**User Identity Section** (if owner configured):
```
"Owner numbers: <numbers>. Treat messages from these numbers as the user."
```

**Workspace Section:**
```
"Your working directory is: <workspaceDir>"
```

**Reply Tags Section:**
- `[[reply_to_current]]` - Reply to triggering message
- `[[reply_to:<id>]]` - Reply to specific message

**Messaging Section:**
- Routing: Current session auto-routes to source channel
- Cross-session: Use `sessions_send(sessionKey, message)`
- Inline buttons support (if enabled)

**Reasoning Format Section** (if reasoning enabled):
```
"ALL internal reasoning MUST be inside <think>...</think>.
Format every reply as <think>...</think> then <final>...</final>"
```

**Runtime Section:**
```
"Runtime: agent=<agentId> | host=<host> | os=<os> (<arch>) |
node=<node> | model=<model> | channel=<channel> | thinking=<thinkLevel>"
```

**Project Context Section** (if context files provided):
- Loads SOUL.md for persona/tone guidance
- Injects all context files in order

**Silent Replies Section:**
```
"When you have nothing to say, respond with ONLY: STOP"
```

**Heartbeats Section:**
```
"If you receive a heartbeat poll, and there is nothing needing attention,
reply exactly: HEARTBEAT_OK"
```

---

## 2. MODEL SELECTION & PROVIDER INTEGRATION

### 2.1 Supported LLM Providers

**Native Providers:**

1. **Anthropic Claude** - claude-3-5-sonnet, claude-opus-4-6, claude-3-opus, claude-3-sonnet
   - Vision, Extended thinking, Full context tracking

2. **OpenAI** - gpt-4-turbo, gpt-4o, gpt-4o-mini, gpt-3.5-turbo
   - Vision, Function calling, Explicit token counting

3. **Google Gemini** - gemini-2.0-flash, gemini-1.5-pro, gemini-1.5-flash
   - Vision, gemini-2.0-flash-thinking-exp for reasoning

4. **GitHub Copilot** - gpt-4-turbo, gpt-3.5-turbo
   - Device flow authentication, 10k requests/month

5. **Amazon Bedrock** - Claude, Llama, Titan
   - Multi-region, cross-account access

6. **Additional** - OpenCode-Zen, Kimi-Coding, Minimax, Z.ai, Venice, Ollama

### 2.2 Model Resolution Flow

```
1. User specifies model (full or alias)
2. resolveModelRef() checks: Direct match -> Alias expansion -> Default fallbacks
3. resolveConfiguredModelRef() applies: Allowlist/denylist filtering -> Default selection
4. Model context window validation: Check against DEFAULT_CONTEXT_TOKENS (100,000)
```

### 2.3 Model Context Window Management

**Constants:**
```typescript
DEFAULT_CONTEXT_TOKENS = 100000      // Default assumption
CONTEXT_WINDOW_WARN_BELOW_TOKENS = 8000  // Warning threshold
CONTEXT_WINDOW_HARD_MIN_TOKENS = 2000    // Hard minimum
```

**Overflow handling:**
1. Context window guard checks before execution
2. If insufficient: Triggers session compaction
3. Message pruning: Removes oldest messages
4. Context eviction: Removes least-important context files
5. Fallback: Switches to model with larger context window

### 2.4 Auth Profile Management

**Files:** `src/agents/auth-profiles.ts` (600+ lines)

**Profile Storage:** `~/.openclaw/agents/<agentId>/auth.json`

**Profile Rotation Logic:**
1. Explicit user-configured order
2. Round-robin by last usage
3. Prioritize "last good" profile
4. Avoid profiles in cooldown

**Failure Handling:**
- Exponential backoff (1s -> 2s -> 4s -> 8s...)
- Max failures before permanent disable: configurable (default 3)
- Cooldown automatically expires

### 2.5 Fallback Chain Configuration

```typescript
agents.defaults.model = {
  primary: "anthropic/claude-opus-4-6",
  fallbacks: [
    "anthropic/claude-3-5-sonnet-20241022",
    "openai/gpt-4o",
    "google/gemini-2-0-flash"
  ]
}
```

**Fallback Triggers:** Auth failures, billing/quota limits, context overflow, rate limiting, timeout, model unavailable

### 2.6 Thinking/Extended Reasoning

**Levels:** off, brief, default, extended, verbose

**Configuration:**
```typescript
agents.defaults.thinkingDefault = "default"  // Global
agents.<agentId>.thinking = "extended"       // Per-agent
channels.<id>.thinking = "brief"             // Per-channel
// Runtime toggle: /reasoning [off|on|stream]
```

---

## 3. AGENT ORCHESTRATION

### 3.1 Core Agent Entry Point

**File:** `src/agents/pi-embedded-runner.ts` (1000+ lines)

Handles: Session management, auth profile resolution, LLM invocation with fallback, tool execution, context pruning, error recovery

### 3.2 Session Management Architecture

**Session Hierarchy:**
```
Main Session
+-- Subagent Session 1
|   +-- Subagent Session 2 (nested)
+-- Subagent Session 2
+-- Background Session (cron/scheduled)
```

**Session Storage:**
```
~/.openclaw/sessions/<sessionKey>/
+-- transcript.jsonl     # Conversation history
+-- memory/              # Session-specific memory
+-- context.json         # Session metadata
```

**Session Types:**
1. Main Session: Primary user interaction
2. Sub-agent Session: Spawned via sessions_spawn
3. Group Session: Multi-user with gating
4. DM Session: Direct message
5. Broadcast Session: Multi-channel
6. Cron Session: Scheduled background

**Session Lifecycle:**
```
1. Create/Resume -> Load auth, transcript, verify permissions
2. Message Reception -> Parse, check policy, enqueue in lane
3. Command Processing -> Lock, resolve model, build prompt, execute
4. Tool Execution Loop -> LLM returns tools, execute, collect results, resume LLM
5. Response Delivery -> Serialize, route via channel, save transcript
6. Cleanup -> Unlock, update stats, log metrics
```

### 3.3 Lane-Based Concurrency Control

Sessions queued by lane to prevent concurrent execution:
- Lane = sessionKey (default)
- Sessions in same lane execute sequentially
- Different lanes execute in parallel

### 3.4 Message Processing Pipeline

1. Message Reception -> Parse payload, extract sender/content/media
2. Group Policy Evaluation -> Apply routing rules, check activation
3. Message Categorization -> Command? Mention? Broadcast? Reaction?
4. Group Activation Check -> Skip if inactive, activate if broadcasting
5. Routing -> Determine target agent and session
6. Execution Enqueue -> Add to lane queue
7. Response Delivery -> Format per channel, send, save transcript

### 3.5 Subagent Spawning

Via `sessions_spawn` tool:
- Validates agentId is in agents_list
- Creates new session: {parentSessionKey}.{uuid}
- Copies parent auth profiles
- Sets minimal system prompt
- Depth limit: 2 (no further spawning)
- Isolated workspace

### 3.6 Session Compaction

When context window overflows:
1. Keep recent N messages (sliding window)
2. Summarize older messages via LLM
3. Store in compact form, remove from active context
4. 30-60% context reduction
5. If still insufficient: remove context files, fallback to larger model

### 3.7 Error Handling & Failover

**Error Classification:**
- Auth -> Profile cooldown, try next
- Billing -> Show message, switch model
- Context overflow -> Compact session, retry
- Rate limit -> Exponential backoff
- Timeout -> Switch model
- Service unavailable -> Retry with backoff

---

## 4. TOOL/FUNCTION DEFINITIONS (25+)

### 4.1 Core File Operation Tools

| Tool | Purpose | Key Features |
|------|---------|-------------|
| `read` | Read file contents | Line range support, binary detection, workspace validation |
| `write` | Create/overwrite files | Atomic write, mkdir -p, permission preservation |
| `edit` | Precise file edits | Fuzzy matching, regex, atomic changes, diff output |
| `apply_patch` | Multi-file patches | Unified diff, atomic multi-file, rollback |
| `grep` | Search file contents | ripgrep-based, regex, context lines, pagination |
| `find` | Find files by pattern | Glob matching, exclusions, 1000 file limit |
| `ls` | List directory | Size, permissions, tree view |

### 4.2 Execution Tools

| Tool | Purpose | Key Features |
|------|---------|-------------|
| `exec` | Run shell commands | PTY support, timeout (300s), working directory, env vars |
| `process` | Manage background sessions | list/stop/read on running processes |

**exec security:**
- Workspace jail (commands run in workspace directory)
- Configurable shell (bash default)
- Environment variable passthrough
- Timeout enforcement (default: 300s)
- Kill on abort
- PTY mode for interactive CLIs

### 4.3 Web Tools

| Tool | Purpose | Key Features |
|------|---------|-------------|
| `web_search` | Search the web | Brave Search API, max 10 results, safe search |
| `web_fetch` | Fetch URL content | Readability extraction, 100KB limit, redirect handling |
| `browser` | Control web browser | Playwright: navigate, click, type, screenshot, extract |

### 4.4 AI & Media Tools

| Tool | Purpose | Key Features |
|------|---------|-------------|
| `image` | Analyze images | Configured image model, base64 or URL input |
| `canvas` | A2UI canvas | Present, evaluate, snapshot rich UI elements |
| `nodes` | IoT/device control | List, describe, notify, camera, screen capture |

### 4.5 Communication Tools

| Tool | Purpose | Key Features |
|------|---------|-------------|
| `message` | Send messages | Polls, reactions, inline buttons, channel actions |
| `sessions_send` | Cross-session messaging | Send to any session by key |
| `sessions_spawn` | Create sub-agents | Isolated workspace, minimal prompt |
| `sessions_list` | List sessions | Filter by active, channel, agent |
| `sessions_history` | Fetch history | Any session's conversation log |
| `session_status` | Status card | Usage, time, reasoning level |
| `agents_list` | List available agents | For sessions_spawn targeting |

### 4.6 System Tools

| Tool | Purpose | Key Features |
|------|---------|-------------|
| `cron` | Job scheduling | List, add, update, remove, run, view history |
| `gateway` | System control | Restart, apply config, run updates |

### 4.7 Memory Tools

| Tool | Purpose |
|------|---------|
| `memory_search` | Semantic + keyword search on MEMORY.md and memory/*.md |
| `memory_get` | Extract specific lines from indexed files |

---

## 5. MEMORY SYSTEM

### 5.1 Architecture

**Extensions:**
- `extensions/memory-core/` - Core memory framework
- `extensions/memory-lancedb/` - Vector DB backend (LanceDB)

**Storage backends:**
- SQLite with sqlite-vec (default, in-process)
- LanceDB (optional, external)

### 5.2 Memory Storage Schema

```sql
CREATE TABLE memories (
  id TEXT PRIMARY KEY,
  agent_id TEXT,
  content TEXT,
  embedding BLOB,          -- Vector (sqlite-vec format)
  source TEXT,             -- "memory" | "sessions"
  created_at INTEGER,
  updated_at INTEGER,
  metadata JSONB
);
```

### 5.3 Hybrid Search Algorithm

**Configuration:**
```typescript
{
  hybrid: {
    enabled: true,
    vectorWeight: 0.7,      // Semantic similarity
    textWeight: 0.3,        // BM25 keyword match
    candidateMultiplier: 4  // Fetch 4x candidates, rank top N
  }
}
```

**Search Flow:**
1. Vector search: Fetch 4x candidates using embedding similarity
2. BM25 text search: Fetch 4x candidates using keyword matching
3. Blend scores: vector * 0.7 + text * 0.3
4. Return top N results (configurable maxResults)

### 5.4 Memory Sources

1. **MEMORY.md** - User's persistent notes and facts
2. **memory/*.md** - Structured memory files by topic
3. **Session transcripts** - Past conversation history

### 5.5 Memory Sync Configuration

```typescript
{
  sync: {
    onSessionStart: boolean,     // Sync when session starts
    onSearch: boolean,           // Background sync on search
    watch: boolean,              // File system watcher
    watchDebounceMs: number,     // Debounce period
    intervalMinutes: number,     // Periodic sync interval
    sessions: {
      deltaBytes: number,        // Min bytes changed
      deltaMessages: number      // Min messages changed
    }
  }
}
```

### 5.6 Embedding Providers

- OpenAI text-embedding-3-small (default)
- Ollama (local, any embedding model)
- Google Gemini (embedding-001)
- Auto-detection based on configured LLM provider

---

## 6. CONTEXT PRUNING

**File:** `src/agents/pi-extensions/context-pruning/`

**Strategies:**
- **TTL-based**: Flush every N milliseconds
- **Token-based**: Trim until under limit
- **Importance-based**: Keep recent, summarize old
- **Tool-based**: Remove old tool invocation context

**Integration:** Hooks into Pi Agent SDK's context event:
```typescript
api.on("context", (event, ctx) => {
  const pruned = pruneContextMessages({
    messages: event.messages,
    settings: runtime.settings,
    isToolPrunable: runtime.isToolPrunable,
    contextWindowTokensOverride: runtime.contextWindowTokens,
  });
  return pruned !== event.messages ? { messages: pruned } : undefined;
});
```

---

## 7. CRON & SCHEDULED EXECUTION

**Library:** Croner (^10.0.1)

**Configuration:**
```typescript
{
  cron: {
    "daily-briefing": {
      schedule: "0 8 * * *",          // Every day at 8 AM
      agentId: "default",
      message: "Give me a morning briefing",
      sessionKey: "cron:daily-briefing",
      enabled: true
    },
    "check-emails": {
      schedule: "*/30 * * * *",       // Every 30 minutes
      agentId: "email-agent",
      message: "Check for new important emails",
      sessionKey: "cron:email-check",
      enabled: true
    }
  }
}
```

**Execution:** Creates a cron session, runs the agent with the configured message, stores results in session transcript.

---

## 8. SKILLS SYSTEM

**Location:** `skills/` directory

**Structure per skill:**
```
skills/<skill-name>/
+-- SKILL.md          # Instructions loaded into system prompt
+-- [supporting files]
```

**Lifecycle:**
1. Skills discovered from filesystem
2. Available skills listed in system prompt
3. Agent reads SKILL.md when skill is selected
4. Skill instructions become part of active prompt
5. One skill active at a time

**Remote skills:** Installable from Clawhub (skills.install, skills.update gateway methods)

---

## 9. PLUGIN TOOL SYSTEM

**Plugin API for tools:**
```typescript
api.defineTool({
  id: "custom-tool",
  name: "custom_tool",
  description: "Description for the LLM",
  parameters: {
    type: "object",
    properties: {
      query: { type: "string", description: "Search query" }
    },
    required: ["query"]
  },
  execute: async (params, context) => {
    return { result: "..." };
  }
});
```

**Tool Policy:** Controls which tools are available per model, group, or channel. Blocked tools are filtered before prompt construction.

---

## KEY STATISTICS

| Metric | Value |
|--------|-------|
| System prompt builder | 646 lines |
| Agent runner | 1000+ lines |
| Auth profiles | 600+ lines |
| Built-in tools | 25+ |
| LLM providers | 8+ |
| Memory search weights | 70% vector / 30% BM25 |
| Context token default | 100,000 |
| Context warn threshold | 8,000 |
| Context hard minimum | 2,000 |
| Tool execution timeout | 300s (exec), 30s (others) |
| Subagent depth limit | 2 |
| Session compaction reduction | 30-60% |
| Embedding providers | 3 (OpenAI, Ollama, Gemini) |
