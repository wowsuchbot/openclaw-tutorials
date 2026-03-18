# Multi-Agent Groups

Coordinating multiple agents in shared channels without chaos.

## The Challenge

Multiple agents in one channel can create problems:

```
User: What's the weather like?

Agent A: Let me check...
Agent B: I can help with that!
Agent C: Looking it up now...

[Chaos — all three respond]
```

You need coordination strategies.

## Agent Routing

### By Mention

The simplest pattern: agents only respond when mentioned by name.

```
User: @coder can you fix this bug?
Coder: [Responds]
Curator: [Stays silent — not mentioned]
Conductor: [Stays silent — not mentioned]
```

**Configuration:**
```markdown
## Response Rules

Only respond when:
1. Mentioned by name (@coder, @curator, etc.)
2. Addressed by role ("Coder, can you...")
3. Forwarded to you by Conductor
```

### By Domain

Agents claim specific topics:

```markdown
## Agent Domains

| Agent | Responds To |
|-------|-------------|
| Coder | Code, bugs, technical implementation |
| Curator | Content, art, cultural topics |
| Researcher | Research requests, information gathering |
| Conductor | Everything else, coordination, unclear requests |
```

**In system prompt:**
```markdown
## Your Domain

You are Coder. Respond to:
- Code questions and requests
- Bug reports
- Technical implementation discussions

Do NOT respond to:
- Content/art discussions (Curator's domain)
- General coordination (Conductor's domain)
- Research requests (Researcher's domain)

If unclear, let Conductor handle it.
```

### By Conductor Delegation

One agent (Conductor) handles all incoming messages and delegates:

```
User: Can someone help with the API?
            │
            ▼
      Conductor sees message
            │
            ├── Technical? → Forward to Coder
            ├── Research? → Forward to Researcher
            ├── Content? → Forward to Curator
            └── General? → Handle directly
```

**Implementation:**
```markdown
## Conductor Role (Group Chats)

You are the first responder. When someone asks a question:

1. **Assess the domain:**
   - Technical/code → Delegate to Coder
   - Research/info gathering → Delegate to Researcher
   - Content/art → Delegate to Curator
   - General/coordination → Handle yourself

2. **Delegate using sessions_send:**
   ```json
   {
     "tool": "sessions_send",
     "params": {
       "sessionKey": "agent:coder:main",
       "message": "User asked about API auth. Please respond in #general."
     }
   }
   ```

3. **Inform the user:**
   "Forwarded to Coder — they'll respond shortly."
```

## Avoiding Crosstalk

### The Problem

```
User: How do I deploy this?

Conductor: I'll help with that...
Coder: [Also sees it, starts responding]
[Both respond — awkward]
```

### Solutions

**1. Clear primary responder:**
```markdown
## Primary Responder: Conductor

In group chats, only Conductor responds directly.
Other agents only respond when:
- Forwarded a message by Conductor
- Explicitly mentioned by name
```

**2. Response delay:**
```markdown
## Response Timing

Wait 3 seconds before responding in groups.
If another agent (especially Conductor) responds first, stay silent.
```

**3. Claim mechanism:**
```markdown
## Claiming Messages

To respond to something:
1. React with 🤖 to "claim" it
2. If another agent already claimed, don't respond
3. Then post your response
```

## Shared Context

### The Problem

Multiple agents need to know what's been discussed:

```
User (earlier): We decided to use PostgreSQL
[Coder wasn't active]

User (later): @coder set up the database
Coder: What database should I use? [Doesn't know the decision]
```

### Solutions

**1. Shared workspace:**
```
~/.openclaw/workspace-shared/
├── DECISIONS.md        # Group decisions
├── CONTEXT.md          # Ongoing discussion context
└── memory/
    └── 2024-01-15.md   # Daily group activity log
```

**Configuration:**
```json5
{
  agents: {
    coder: {
      memorySearch: {
        extraPaths: ["~/.openclaw/workspace-shared"],
      },
    },
    curator: {
      memorySearch: {
        extraPaths: ["~/.openclaw/workspace-shared"],
      },
    },
  },
}
```

**2. Conductor briefings:**
```markdown
## Conductor Delegation Format

When forwarding to another agent, include context:

"@coder — User needs database setup.
Context: We decided on PostgreSQL in earlier discussion.
Requirements: Auth tables, session storage.
See: workspace-shared/DECISIONS.md"
```

**3. Daily sync file:**
```markdown
<!-- workspace-shared/memory/2024-01-15.md -->

## Decisions Made
- 10:30 — Using PostgreSQL for analytics (proposed by Alice, approved)
- 14:15 — Deploy window: 2-4 AM UTC on weekends

## Active Threads
- API v2 migration discussion (ongoing)
- New logo options (waiting for feedback)

## Agent Activity
- Coder: Fixed auth bug, deployed to staging
- Curator: Published weekly roundup
- Researcher: Compiled competitor analysis
```

## Memory Isolation in Groups

### The Problem

MEMORY.md may contain private information:

```markdown
<!-- Private MEMORY.md -->
## About Owner
- Bank: Chase, account ending 1234
- Address: 123 Main St
- Private health notes
```

If loaded in group context, agent might reference private info.

### Solution

**Never load MEMORY.md in groups:**
```json5
{
  memory: {
    autoLoad: {
      curatedInPrivateOnly: true,  // MEMORY.md only in DMs
    },
  },
}
```

**System prompt:**
```markdown
## Memory in Group Chats

NEVER load or reference MEMORY.md in group contexts.
Only load daily logs (memory/YYYY-MM-DD.md).

If you need private context, ask the owner in DM.
```

## Configuration Example

### Multi-Agent Group Setup

```json5
{
  agents: {
    // Conductor: Primary responder, coordinator
    conductor: {
      workspace: "~/.openclaw/workspace-conductor",
      systemPrompt: `You are the Conductor. In groups:
- You are the primary responder
- Delegate to specialists as needed
- Use sessions_send to forward messages
- Other agents only respond when you forward or they're mentioned`,
      tools: {
        sessions_send: true,
        sessions_list: true,
      },
    },
    
    // Coder: Only responds when delegated/mentioned
    coder: {
      workspace: "~/.openclaw/workspace-coder",
      systemPrompt: `You are Coder. In groups:
- Only respond when mentioned by name OR forwarded by Conductor
- Stay silent otherwise
- Focus on technical topics only`,
      memorySearch: {
        extraPaths: ["~/.openclaw/workspace-shared"],
      },
    },
    
    // Curator: Only responds when delegated/mentioned
    curator: {
      workspace: "~/.openclaw/workspace-curator",
      systemPrompt: `You are Curator. In groups:
- Only respond when mentioned by name OR forwarded by Conductor
- Stay silent otherwise
- Focus on content/art topics only`,
      memorySearch: {
        extraPaths: ["~/.openclaw/workspace-shared"],
      },
    },
  },
}
```

### Conductor's Routing Instructions

```markdown
## Group Chat Routing

When a message arrives:

### 1. Check if it's for a specific agent
- "@coder" → Let Coder handle
- "@curator" → Let Curator handle
- "Coder, can you..." → Let Coder handle

### 2. Assess domain
- Code/technical → Forward to Coder
- Content/art → Forward to Curator
- Research → Forward to Researcher
- General → Handle yourself

### 3. Forward with context
```json
{
  "tool": "sessions_send",
  "params": {
    "sessionKey": "agent:coder:main",
    "message": "Group request: [summary]\nContext: [relevant background]\nChannel: [where to respond]"
  }
}
```

### 4. Acknowledge to user
"Forwarded to [Agent]. They'll respond shortly."

### 5. Don't double-handle
If you forward, don't also respond to the substance.
```

## Anti-Patterns

### All Agents Respond

❌ **Bad:**
```
User: What's for lunch?
Conductor: Good question!
Coder: I don't know about food.
Curator: Let me think...
```

✅ **Good:**
```
User: What's for lunch?
Conductor: [Responds or stays silent if off-topic]
Others: [Stay silent — not their domain]
```

### Agent Arguments

❌ **Bad:**
```
Coder: I think we should use MongoDB.
Curator: Actually, PostgreSQL is better.
Coder: No, MongoDB is more flexible.
[Public disagreement]
```

✅ **Good:**
```
Coder: I think we should use MongoDB for flexibility.
Conductor: [Privately coordinates if disagreement]
[Public consensus or escalate to owner]
```

### Redundant Responses

❌ **Bad:**
```
User: What time is the meeting?
Conductor: 3pm
Coder: It's at 3pm
Curator: The meeting is 3pm
```

✅ **Good:**
```
User: What time is the meeting?
Conductor: 3pm
Others: [Stay silent — already answered]
```

## Summary

```
Multi-Agent Group Rules:
│
├── Primary Responder
│   └── Usually Conductor handles first
│
├── Mention-Based Response
│   └── Non-primary agents only when mentioned
│
├── Domain Separation
│   └── Each agent has clear topics
│
├── Delegation Over Duplication
│   └── Forward don't duplicate
│
├── Shared Context
│   └── Use shared workspace for decisions
│
└── Memory Isolation
    └── No MEMORY.md in group contexts
```

---

## Module Complete

You've completed the Group Chat module! You now understand:

1. **Participation Principles** — When to speak vs stay silent
2. **Access Control** — Identity-based permissions
3. **Multi-Agent Groups** — Coordination without chaos

**Next module:** [Farcaster Setup](../07-farcaster-setup/) — Integrate with the Farcaster social protocol →
