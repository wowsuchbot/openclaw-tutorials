# Markdown Memory

The foundation of OpenClaw's memory system — plain Markdown files that you and your agent can read, write, and search.

## The Two Memory Layers

### MEMORY.md — Long-Term Curated Memory

**Location:** `~/.openclaw/workspace/MEMORY.md`

**Purpose:** 
- Durable facts that persist indefinitely
- User preferences and context
- Important decisions and lessons learned
- Reference information the agent should always know

**When loaded:** Main/private sessions only (NOT group chats)

**Who writes:** You (manual curation) or agent (when asked)

### memory/YYYY-MM-DD.md — Daily Logs

**Location:** `~/.openclaw/workspace/memory/2026-03-18.md`

**Purpose:**
- Running notes from today's sessions
- Temporary context and observations
- Work-in-progress items
- Things that might become MEMORY.md entries

**When loaded:** Today + yesterday, always (all session types)

**Who writes:** Agent (during conversations) or you (manual notes)

## Session Loading Behavior

| File | DM Session | Group Session | Sub-Agent |
|------|------------|---------------|-----------|
| `MEMORY.md` | ✅ Loaded | ❌ Not loaded | ❌ Not loaded |
| Today's daily log | ✅ Loaded | ✅ Loaded | ✅ Loaded |
| Yesterday's daily log | ✅ Loaded | ✅ Loaded | ✅ Loaded |
| Older daily logs | 🔍 Searchable | 🔍 Searchable | 🔍 Searchable |

**Why not load MEMORY.md everywhere?**
- Security: Contains personal context
- Performance: Keeps group/shared sessions light
- Privacy: Prevents leaking your data to group members

## MEMORY.md Best Practices

### Recommended Structure

```markdown
# Long-Term Memory

## About the User
- Name: Max
- Timezone: America/New_York
- Communication style: Direct, prefers concise answers

## Preferences
- Code style: TypeScript, functional patterns
- AI interactions: No emojis unless requested
- Documentation: Prefer examples over theory

## Current Projects
- **OpenClaw Tutorials** (active)
  - Goal: Comprehensive learning path
  - Status: Writing Module 3 (Memory)
- **Cryptoart Curation** (ongoing)
  - Platform: cryptoart.social

## People
- **Alice** — Collaborator on Project X, prefers async comms
- **Bob** — Technical reviewer, very detail-oriented

## Key Decisions
- 2026-03-15: Chose Gemini embeddings for cost/quality balance
- 2026-03-10: Migrated to per-channel-peer session isolation

## Lessons Learned
- Always verify dmScope before opening bot to new users
- Daily logs work better than trying to remember everything
- Vector search + keyword search hybrid catches more relevant results

## Recurring Tasks
- Heartbeat checks: Email, calendar, weather (rotate 2-4x daily)
- Memory review: Weekly, promote important items here

## Reference
- OpenClaw docs: https://docs.openclaw.ai
- Project repo: ~/projects/openclaw-tutorials
```

### Writing Guidelines

**DO:**
- Use headers for organization
- Keep entries concise but complete
- Include context for decisions ("Chose X because Y")
- Date important entries
- Update when things change (don't just append)

**DON'T:**
- Duplicate daily log entries verbatim
- Include transient information (today's tasks)
- Store secrets or credentials
- Let it grow unbounded (review and prune)

### Size Guidelines

- **Target:** 2,000-5,000 tokens (~1,500-3,500 words)
- **Maximum practical:** 10,000 tokens
- **If larger:** Split into topic-specific files (e.g., `memory/projects.md`)

## Daily Log Best Practices

### Automatic Structure

```markdown
# 2026-03-18

## Morning Session
- Discussed memory architecture for tutorials
- User wants comprehensive coverage of hybrid search

## Afternoon Session
- Implemented vector search configuration
- Added examples for Gemini provider

## Tasks
- [ ] Write temporal decay section
- [x] Document MMR re-ranking

## Notes
- User mentioned Coach system — look into integration
- Embedding costs: ~$0.001 per 1K tokens (Gemini)

## Action Items for Tomorrow
- Review what was written today
- Start Module 4 (Sub-Agents)
```

### When Agent Should Write Daily Logs

The agent should write to daily logs:
- After significant decisions or discussions
- When user says "remember this"
- At session end (summary of what was covered)
- When encountering information to preserve

### Triggering Memory Writes

**Explicit:**
```
User: "Remember that the deadline is March 30th"
Agent: *writes to memory/2026-03-18.md*
```

**Automatic (pre-compaction):**
```json5
{
  agents: {
    defaults: {
      compaction: {
        memoryFlush: {
          enabled: true,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md",
        },
      },
    },
  },
}
```

## Memory Search vs Auto-Load

### Auto-Loaded (Always in Context)

```
Session starts →
  Load MEMORY.md (if DM)
  Load memory/2026-03-18.md (today)
  Load memory/2026-03-17.md (yesterday)
→ Agent has immediate context
```

### Searched (On-Demand)

```
User: "What did we discuss last week?"

Agent: *calls memory_search("discussed last week")*

Result: memory/2026-03-11.md:15-22
  "Discussed migrating to multi-user deployment..."
```

### When to Use Which

| Scenario | Auto-Load | Search |
|----------|-----------|--------|
| User preferences | ✅ MEMORY.md | — |
| Today's context | ✅ Daily log | — |
| Last week's conversation | — | ✅ memory_search |
| Specific project details | ✅ If in MEMORY.md | ✅ If in daily logs |
| Finding "that thing we talked about" | — | ✅ Semantic search |

## File Naming and Location

### Required Locations

```
~/.openclaw/workspace/
├── MEMORY.md              # UPPERCASE (primary)
└── memory/
    └── YYYY-MM-DD.md      # ISO date format
```

### Case Sensitivity

- `MEMORY.md` — Primary long-term memory (uppercase)
- `memory.md` — Fallback only (lowercase, deprecated)
- If both exist: Only `MEMORY.md` is loaded

### Date Format

Daily logs MUST use ISO format: `YYYY-MM-DD.md`

```
✅ memory/2026-03-18.md
❌ memory/03-18-2026.md
❌ memory/March-18.md
❌ memory/today.md
```

## Multi-Agent Memory

Each agent has its own workspace:

```
~/.openclaw/
├── workspace/              # Main agent
│   ├── MEMORY.md
│   └── memory/
├── workspace-coder/        # Coder agent
│   ├── MEMORY.md
│   └── memory/
└── workspace-coach/        # Coach agent
    ├── MEMORY.md
    └── memory/
```

Configure per-agent:
```json5
{
  agents: {
    list: [
      {
        id: "coder",
        workspace: "~/.openclaw/workspace-coder",
      },
    ],
  },
}
```

## Memory File Tools

Agents can manipulate memory files using standard tools:

```json
// Read current memory
{
  "tool": "read",
  "params": { "path": "MEMORY.md" }
}

// Append to daily log
{
  "tool": "edit",
  "params": {
    "path": "memory/2026-03-18.md",
    "oldString": "## Notes",
    "newString": "## Notes\n- New observation here"
  }
}

// Create new daily log
{
  "tool": "write",
  "params": {
    "path": "memory/2026-03-18.md",
    "content": "# 2026-03-18\n\n## Session\n- First entry"
  }
}
```

## Common Patterns

### Pattern: End-of-Session Summary

```
User: "That's all for now"

Agent: Let me save today's progress.
*writes to memory/2026-03-18.md:*
  ## Evening Session
  - Completed Module 3 Memory Architecture section
  - User interested in Coach integration for curation
  - Next: Hybrid search configuration
```

### Pattern: Promote to Long-Term Memory

During periodic review:
```
Agent reviews memory/2026-03-15.md:
  "Decided to use Gemini for embeddings due to cost/quality balance"

Agent updates MEMORY.md:
  ## Key Decisions
  - 2026-03-15: Chose Gemini embeddings for cost/quality balance
```

### Pattern: Reference Other Files

In daily log:
```markdown
## Project Notes
- See MEMORY.md > Projects > OpenClaw Tutorials for context
- Related file: ~/projects/openclaw-tutorials/README.md
```

## Troubleshooting

### Memory not loading

```bash
# Check file exists
ls -la ~/.openclaw/workspace/MEMORY.md

# Check permissions
cat ~/.openclaw/workspace/MEMORY.md

# Verify workspace path in config
grep -i "workspace" ~/.openclaw/openclaw.json
```

### Agent doesn't write to memory

- Check tool permissions (need `group:fs` or explicit `write`/`edit`)
- Verify workspace is writable
- Agent may need explicit instruction: "Write this to memory"

### Daily log not created

Agent needs instruction or automation:
```json5
{
  // Heartbeat can create daily log
  messages: {
    heartbeat: {
      prompt: "Check if memory/YYYY-MM-DD.md exists; create if needed",
    },
  },
}
```

## Summary

| File | Purpose | When Loaded | Who Writes |
|------|---------|-------------|------------|
| `MEMORY.md` | Curated long-term facts | DM sessions | You + Agent |
| `memory/YYYY-MM-DD.md` | Daily running notes | All sessions (today + yesterday) | Agent + You |

**Key principle:** Write everything to daily logs. Periodically promote important items to MEMORY.md.

## Next Steps

Now that you understand the files, learn to [set up vector search](./03-vector-memory-setup.md) for fast semantic retrieval →
