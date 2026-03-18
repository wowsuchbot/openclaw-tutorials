# DM Scope Explained

The `session.dmScope` setting is **the most important configuration** for multi-user deployments. This page explains exactly how it works.

## The Problem

By default, OpenClaw uses a single "main" session for all direct messages:

```
User A messages bot → Session: "main"
User B messages bot → Session: "main"  ← Same session!
```

This means User A and User B:
- See each other's conversation history
- Share the same context window
- Can access each other's sensitive information

## The Solution: dmScope

```json5
{
  session: {
    dmScope: "per-channel-peer",
  },
}
```

This creates separate sessions:
```
User A messages bot → Session: "agent:main:telegram:dm:userA"
User B messages bot → Session: "agent:main:telegram:dm:userB"  ← Different!
```

## dmScope Values

### `main` (Default — UNSAFE for multi-user)

**All DMs share one session.**

```
┌─────────────┐
│   User A    │──┐
└─────────────┘  │     ┌─────────────┐
                 ├────▶│   Session   │
┌─────────────┐  │     │   "main"    │
│   User B    │──┘     └─────────────┘
└─────────────┘
                 │     
┌─────────────┐  │     
│   User C    │──┘     
└─────────────┘        
```

**Session key:** `main`

**Use case:** Single user, personal assistant

**Risk:** Complete context sharing between all users

### `per-peer` (Basic isolation)

**Separate session per sender, across all channels.**

```
┌─────────────┐     ┌─────────────────────┐
│   User A    │────▶│ Session: peer:userA │
└─────────────┘     └─────────────────────┘
      │
      │ (same user, different channel)
      ▼
┌─────────────┐     ┌─────────────────────┐
│   User A    │────▶│ Session: peer:userA │  ← Same session!
│  (Discord)  │     └─────────────────────┘
└─────────────┘

┌─────────────┐     ┌─────────────────────┐
│   User B    │────▶│ Session: peer:userB │
└─────────────┘     └─────────────────────┘
```

**Session key:** `agent:main:peer:<peerId>`

**Use case:** Multi-user, single channel, or when you WANT cross-channel continuity

**Risk:** If same user has different IDs on different platforms, they get separate sessions. If IDs happen to collide (unlikely), they share.

### `per-channel-peer` (Recommended)

**Separate session per sender, per channel.**

```
┌─────────────┐     ┌──────────────────────────────┐
│   User A    │────▶│ Session: telegram:dm:userA   │
│ (Telegram)  │     └──────────────────────────────┘
└─────────────┘

┌─────────────┐     ┌──────────────────────────────┐
│   User A    │────▶│ Session: discord:dm:userA    │  ← Different!
│  (Discord)  │     └──────────────────────────────┘
└─────────────┘

┌─────────────┐     ┌──────────────────────────────┐
│   User B    │────▶│ Session: telegram:dm:userB   │
│ (Telegram)  │     └──────────────────────────────┘
└─────────────┘
```

**Session key:** `agent:main:<channel>:dm:<peerId>`

**Use case:** Multi-user, multi-channel (most common)

**Risk:** None significant. Same user on different channels has different sessions, which is usually desired for isolation.

### `per-account-channel-peer` (Maximum isolation)

**Separate session per bot account, per channel, per sender.**

```
┌─────────────┐     ┌───────────────────────────────────────┐
│   User A    │────▶│ Session: bot1:telegram:dm:userA       │
│  (to Bot1)  │     └───────────────────────────────────────┘
└─────────────┘

┌─────────────┐     ┌───────────────────────────────────────┐
│   User A    │────▶│ Session: bot2:telegram:dm:userA       │  ← Different!
│  (to Bot2)  │     └───────────────────────────────────────┘
└─────────────┘
```

**Session key:** `agent:main:<accountId>:<channel>:dm:<peerId>`

**Use case:** Multiple bot accounts on same channel (e.g., multiple Telegram bots)

**Risk:** Most verbose session keys, but maximum isolation.

## Comparison Table

| dmScope | Session Key Pattern | Users Isolated? | Channels Isolated? | Accounts Isolated? |
|---------|--------------------|-----------------|--------------------|-------------------|
| `main` | `main` | ❌ No | ❌ No | ❌ No |
| `per-peer` | `peer:<peerId>` | ✅ Yes | ❌ No | ❌ No |
| `per-channel-peer` | `<channel>:dm:<peerId>` | ✅ Yes | ✅ Yes | ❌ No |
| `per-account-channel-peer` | `<account>:<channel>:dm:<peerId>` | ✅ Yes | ✅ Yes | ✅ Yes |

## Which Should You Use?

### Single User (Personal Assistant)

```json5
{
  session: {
    dmScope: "main",  // OK — you're the only user
  },
}
```

### Multi-User (Public Bot)

```json5
{
  session: {
    dmScope: "per-channel-peer",  // REQUIRED for safety
  },
}
```

### Multi-User + Multi-Bot

```json5
{
  session: {
    dmScope: "per-account-channel-peer",  // Maximum isolation
  },
}
```

## Configuration

### Basic Setup

```json5
// ~/.openclaw/openclaw.json
{
  session: {
    dmScope: "per-channel-peer",
  },
}
```

### With Additional Options

```json5
{
  session: {
    dmScope: "per-channel-peer",
    
    // Optional: session timeout
    timeoutMinutes: 60,  // Clear inactive sessions after 1 hour
    
    // Optional: max history
    maxHistoryMessages: 100,  // Keep last 100 messages
  },
}
```

## Common Mistakes

### Mistake 1: Forgetting to Set dmScope

```json5
// ❌ BAD — defaults to "main"
{
  channels: {
    telegram: { enabled: true },
  },
}

// ✅ GOOD — explicitly set
{
  session: {
    dmScope: "per-channel-peer",
  },
  channels: {
    telegram: { enabled: true },
  },
}
```

### Mistake 2: Using "main" for Multi-User

```json5
// ❌ DANGEROUS — users share context
{
  session: { dmScope: "main" },
  channels: {
    telegram: {
      dmPolicy: "open",  // Anyone can message
    },
  },
}

// ✅ SAFE — users isolated
{
  session: { dmScope: "per-channel-peer" },
  channels: {
    telegram: {
      dmPolicy: "open",
    },
  },
}
```

### Mistake 3: Changing dmScope Without Migration

If you change dmScope on a running system, existing sessions may behave unexpectedly:

```bash
# Before change:
#   User A's session: "main"
#   User A has 50 messages of history

# After change to per-channel-peer:
#   User A's session: "agent:main:telegram:dm:123456789"
#   User A starts fresh (old "main" session orphaned)
```

See [Multi-User Transition](./03-multi-user-transition.md) for migration steps.

## Verifying Your Setup

### Check Current Config

```bash
grep -A5 "session" ~/.openclaw/openclaw.json
```

### Test Isolation

1. Message bot from User A
2. Ask "What was my last message?"
3. Message bot from User B (different account)
4. Ask "What was my last message?"

If User B sees User A's message, you have a problem!

### Check Session Keys in Logs

```bash
openclaw logs --follow | grep "session"

# Look for:
# [session] Created: agent:main:telegram:dm:123456789
# [session] Created: agent:main:telegram:dm:987654321
```

If you see the same session key for different users, dmScope is wrong.

## Impact on Other Features

### Memory Search

With isolated sessions, memory search is scoped:
- Each user's `memory_search` only searches their agent's memory
- Group context is shared within the group session

### Sub-Agents

Sub-agent sessions inherit isolation:
```
Parent session: agent:main:telegram:dm:123456789
Child session:  agent:main:subagent:<uuid>
```

### Tool Visibility

```json5
{
  tools: {
    sessions: {
      visibility: "tree",  // Can see own session + sub-agents only
    },
  },
}
```

With `per-channel-peer` + `visibility: "tree"`, users cannot see other users' sessions.

## Summary

| Your Situation | Recommended dmScope |
|----------------|---------------------|
| Only you use the bot | `main` |
| Multiple users, one channel | `per-channel-peer` |
| Multiple users, multiple channels | `per-channel-peer` |
| Multiple bots, multiple users | `per-account-channel-peer` |

**When in doubt, use `per-channel-peer`** — it's safe for all scenarios.

## Next Steps

Ready to migrate? See [Multi-User Transition](./03-multi-user-transition.md) for the step-by-step process →
