# Session Basics

Understanding how OpenClaw manages conversations — the foundation of multi-user safety.

## What is a Session?

A **session** is a conversation context that includes:

- **History** — Past messages between user and agent
- **State** — Current conversation metadata
- **Memory access** — Which memory files are loaded
- **Tool context** — Sandboxing and permissions

Each session is identified by a unique **session key**.

## Session Key Structure

```
agent:<agentId>:<channel>:<chatType>:<identifier>

Components:
  agentId    — Which agent handles this session (e.g., "main", "coder")
  channel    — Source platform (e.g., "telegram", "discord")
  chatType   — "dm" or "group"
  identifier — User/group ID from the platform
```

### Examples

```
# Telegram DM from user 123456789 to main agent
agent:main:telegram:dm:123456789

# Telegram group -1001234567890
agent:main:telegram:group:-1001234567890

# Telegram forum topic 42 in group
agent:main:telegram:group:-1001234567890:topic:42

# Discord DM to coder agent
agent:coder:discord:dm:user#1234

# Sub-agent spawned session
agent:main:subagent:a1b2c3d4
```

## Session Lifecycle

### Creation

Sessions are created **on first message**:

```
User sends "Hello" → 
  Gateway checks dmScope →
  Generates session key →
  Creates session file →
  Loads context →
  Routes to LLM
```

### Persistence

Sessions are persisted to disk as **JSONL** (JSON Lines) files:

```
~/.openclaw/agents/<agentId>/sessions/
├── sessions.json              # Session index
├── <uuid>.jsonl               # Transcript (one JSON per line)
└── <uuid>.jsonl.lock          # Active session lock
```

**Transcript format (JSONL):**
```json
{"role":"user","content":"Hello","timestamp":"2026-03-18T10:00:00Z"}
{"role":"assistant","content":"Hi! How can I help?","timestamp":"2026-03-18T10:00:01Z"}
{"role":"user","content":"What's the weather?","timestamp":"2026-03-18T10:01:00Z"}
```

### Session Index

`sessions.json` maps session keys to UUIDs:

```json
{
  "agent:main:telegram:dm:123456789": {
    "uuid": "f8282125-d96c-45a0-84cb-eb5fb6843d0a",
    "created": "2026-03-15T08:00:00Z",
    "lastActive": "2026-03-18T10:01:00Z",
    "messageCount": 47
  }
}
```

### Expiration and Compaction

Sessions don't expire by default, but they can be **compacted**:

```json5
{
  agents: {
    defaults: {
      compaction: {
        // Trigger compaction when context nears limit
        reserveTokensFloor: 20000,
        
        // Memory flush before compaction
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
        },
      },
    },
  },
}
```

**Compaction flow:**
1. Session approaches token limit
2. Memory flush triggered (agent writes important context to memory files)
3. Old messages summarized or dropped
4. Session continues with reduced history

## Session State

Beyond history, sessions track:

```json
{
  "key": "agent:main:telegram:dm:123456789",
  "uuid": "f8282125-...",
  "state": {
    "model": "anthropic/claude-sonnet-4-20250514",
    "modelOverride": null,
    "activationMode": "always",
    "lastCompaction": null,
    "memoryFlushed": false
  }
}
```

### Per-Session Overrides

Users can change settings within their session:

```
/model gpt-4o           → Sets modelOverride
/activation mention     → Sets activationMode
/clear                  → Resets history (keeps state)
```

These persist in the session file, not the global config.

## Session Isolation

**The key question:** When User A and User B both message your bot, do they:

1. **Share a session** — See each other's history (UNSAFE)
2. **Have separate sessions** — Completely isolated (SAFE)

This is controlled by `session.dmScope`:

```json5
{
  session: {
    dmScope: "per-channel-peer",  // Separate session per user per channel
  },
}
```

### Visual: Shared vs Isolated

**Shared Session (dmScope: "main"):**
```
User A: "My password is secret123"
User B: "What was the last message?"
Agent:  "The last message was 'My password is secret123'"  ← DATA LEAK!
```

**Isolated Sessions (dmScope: "per-channel-peer"):**
```
Session: agent:main:telegram:dm:userA
  User A: "My password is secret123"
  Agent:  "I won't share that with anyone."

Session: agent:main:telegram:dm:userB
  User B: "What was the last message?"
  Agent:  "This is our first conversation!"  ← SAFE
```

## Viewing Sessions

### CLI

```bash
# List all sessions
openclaw sessions list

# Output:
# KEY                                    MESSAGES  LAST ACTIVE
# agent:main:telegram:dm:123456789       47        2 hours ago
# agent:main:telegram:group:-1001234     12        1 day ago

# View session history
openclaw sessions history "agent:main:telegram:dm:123456789"

# Clear a session
openclaw sessions clear "agent:main:telegram:dm:123456789"
```

### Filesystem

```bash
# Find session files
ls ~/.openclaw/agents/main/sessions/

# Read transcript directly
cat ~/.openclaw/agents/main/sessions/<uuid>.jsonl | jq .

# Check session index
cat ~/.openclaw/agents/main/sessions/sessions.json | jq .
```

## Session Tools

Agents have tools to interact with sessions:

| Tool | Purpose |
|------|---------|
| `session_status` | Get current session info |
| `sessions_list` | List visible sessions |
| `sessions_history` | Read another session's history |
| `sessions_send` | Send message to another session |
| `sessions_spawn` | Create sub-agent session |

**Visibility control:**
```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",  // See own session + spawned sub-agents
    },
  },
}
```

## Common Patterns

### Pattern: Check Current Session

Agent can introspect:
```
Tool: session_status
Result: {
  "key": "agent:main:telegram:dm:123456789",
  "messageCount": 47,
  "model": "anthropic/claude-sonnet-4-20250514",
  "tokensUsed": 12500
}
```

### Pattern: Cross-Session Communication

Agent in one session can message another:
```
Tool: sessions_send
Params: {
  "sessionKey": "agent:coder:main",
  "message": "User requested a code review"
}
```

### Pattern: Spawn Sub-Agent

Delegate task to isolated sub-agent:
```
Tool: sessions_spawn
Params: {
  "task": "Research the history of Bitcoin",
  "agentId": "researcher"
}
```

## Summary

| Concept | Description |
|---------|-------------|
| **Session Key** | Unique identifier for a conversation |
| **Session File** | JSONL transcript on disk |
| **Session State** | Metadata (model, activation, etc.) |
| **Session Isolation** | Controlled by `dmScope` |
| **Session Tools** | Agent tools for session interaction |

## Next Steps

Now that you understand sessions, learn about [dmScope in detail](./02-dm-scope-explained.md) — the setting that makes or breaks multi-user safety →
