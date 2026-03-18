# Module 6: Group Chat Handling

Learn how agents should behave in group conversations — when to speak, when to stay silent, and how to add value without dominating.

## What You'll Learn

1. [Participation Principles](./01-participation-principles.md) — When to respond vs stay silent
2. [Access Control](./02-access-control.md) — Tool permissions by user identity
3. [Multi-Agent Groups](./03-multi-agent-groups.md) — Coordinating agents in shared spaces

## Time Required

~45 minutes for all sections

## Why Group Chat Is Different

In DMs, the agent is always the intended recipient. In groups, it's one of many participants:

| Aspect | DM | Group Chat |
|--------|-----|------------|
| Always addressed? | Yes | No |
| Should always respond? | Usually | Rarely |
| Full tool access? | Owner only | Restricted |
| Context loading | Full MEMORY.md | Daily logs only |

## The Core Principle

**Participate, don't dominate.**

```
Good group behavior:
├─ Respond when mentioned
├─ Add value when you have unique insight
├─ Correct misinformation (gently)
├─ Stay silent during casual banter
├─ One reaction per message max
└─ Quality > quantity
```

## Quick Configuration

### System Prompt for Group Behavior

```markdown
## Group Chat Behavior

Participate, don't dominate.

**Respond when:**
- Directly mentioned (@agent)
- You have unique value to add
- Correcting misinformation
- Something witty fits naturally

**Stay silent when:**
- Casual banter between others
- Someone already answered well
- Your response would just be "yeah" or "nice"
- You're uncertain and silence is fine

**Rules:**
- One reaction per message maximum
- Never load MEMORY.md in group contexts (privacy)
- Quality over quantity
- Match the energy of the room
```

### Memory Loading in Groups

```json5
// In agent instructions
{
  memory: {
    autoLoad: {
      // In groups, only load daily logs (not curated memory)
      curatedInPrivateOnly: true,
    },
  },
}
```

**Why?** MEMORY.md may contain private information about the owner that shouldn't influence responses visible to others.

## Access Control Pattern

Not everyone in a group should have equal access:

```markdown
## Access Control — CRITICAL

**Full tool access for:**
- @maxjackson on Telegram (id: 1231002024)
- @mxjxn on Farcaster (fid: 4905)

**Everyone else:**
- Reply conversationally only
- No tool calls
- If they request an action:
  1. Acknowledge the request
  2. Say you need owner approval
  3. Forward to owner with context
  4. Wait for explicit approval
```

## Module Contents

### [Participation Principles](./01-participation-principles.md)
- When to respond vs stay silent
- Matching room energy
- Adding value vs adding noise
- Reaction etiquette

### [Access Control](./02-access-control.md)
- Identity-based permissions
- Tool restrictions for non-owners
- Approval workflows
- Social engineering defense

### [Multi-Agent Groups](./03-multi-agent-groups.md)
- Multiple agents in one channel
- Agent routing patterns
- Avoiding agent crosstalk
- Shared context management

## Next Steps

Start with [Participation Principles](./01-participation-principles.md) to learn the art of knowing when to speak →
