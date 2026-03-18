# Module 2: Session Management

Session management is **the most critical topic** for transitioning from single-user to multi-user deployments. Get this wrong and users will see each other's conversations.

## What You'll Learn

1. [Session Basics](./01-session-basics.md) — Keys, lifecycle, storage
2. [DM Scope Explained](./02-dm-scope-explained.md) — Why `main` is unsafe for multi-user
3. [Multi-User Transition](./03-multi-user-transition.md) — Migration checklist
4. [Security Patterns](./04-security-patterns.md) — Context leak prevention

## Time Required

~45 minutes for all sections

## Why This Matters

Consider this scenario:

1. **User A** messages your bot: "My social security number is 123-45-6789"
2. **User B** messages your bot: "What did the last person tell you?"
3. With default settings, **User B can see User A's message**

This happens because the default `session.dmScope: "main"` shares a single session across all DMs.

## Key Concept: Session Keys

Every conversation has a **session key** that determines:
- Which conversation history is loaded
- Which memory context is available
- Whether tool execution is sandboxed

```
Session Key Format: agent:<agentId>:<channel>:<scope>:<identifier>

Examples:
  agent:main:telegram:dm:123456789     # Telegram DM from user 123456789
  agent:main:telegram:group:-1001234   # Telegram group -1001234
  agent:coder:discord:dm:user#1234     # Discord DM to coder agent
```

## The Critical Setting: `dmScope`

```json5
{
  session: {
    // This ONE setting determines user isolation
    dmScope: "per-channel-peer",  // SAFE for multi-user
  },
}
```

| Value | Behavior | Use Case |
|-------|----------|----------|
| `main` | All DMs share one session | Single user ONLY |
| `per-peer` | Isolate by sender ID | Multi-user, single channel |
| `per-channel-peer` | Isolate by channel + sender | **Multi-user (recommended)** |
| `per-account-channel-peer` | Isolate by account + channel + sender | Multiple bot accounts |

## Quick Safety Check

```bash
# Check your current setting
grep -i "dmScope" ~/.openclaw/openclaw.json

# If it shows "main" or is missing, you're at risk!
# Add this immediately:
```

```json5
{
  session: {
    dmScope: "per-channel-peer",
  },
}
```

## Module Contents

### [Session Basics](./01-session-basics.md)
- What is a session?
- Session key structure
- Lifecycle: creation, persistence, expiration
- Storage: transcripts, state

### [DM Scope Explained](./02-dm-scope-explained.md)
- How `dmScope` works
- Visual diagrams of each mode
- When to use which setting
- Common mistakes

### [Multi-User Transition](./03-multi-user-transition.md)
- Migration checklist
- Handling existing sessions
- Testing isolation
- Rollback plan

### [Security Patterns](./04-security-patterns.md)
- Context leak prevention
- Group chat isolation
- Tool execution sandboxing
- Audit logging

## Quick Start: Make It Safe Now

If you're running a bot that multiple people might message:

```json5
// ~/.openclaw/openclaw.json
{
  session: {
    // CRITICAL: Isolate each user's conversation
    dmScope: "per-channel-peer",
  },
  
  channels: {
    telegram: {
      enabled: true,
      dmPolicy: "pairing",  // Require explicit approval
    },
  },
}
```

Then restart:
```bash
openclaw gateway restart
```

## Next Steps

Start with [Session Basics](./01-session-basics.md) to understand the full picture →
