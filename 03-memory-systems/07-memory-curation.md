# Memory Curation

The key to effective long-term memory: agent-suggested, human-curated workflows.

## The Curation Problem

Without curation, memory systems fail in predictable ways:

**Too much noise:**
```
MEMORY.md grows to 50,000 lines
↓
Every search returns marginally relevant results
↓
Agent context gets polluted
↓
Responses become unfocused
```

**Too little signal:**
```
User forgets to add important context
↓
Agent keeps asking same questions
↓
Frustration on both sides
↓
Memory system abandoned
```

**The solution:** Combine agent awareness with human judgment.

## The Curation Workflow

```
Daily Activity
     ↓
Daily Log (memory/YYYY-MM-DD.md)
     ↓ (automatic)
Agent reviews during heartbeats/check-ins
     ↓
Suggests memory candidates
     ↓
Human reviews suggestions
     ↓ (approve/reject/edit)
Promoted to MEMORY.md
     ↓
Periodic reconciliation
```

This workflow ensures:
- Nothing important is lost (agent catches it)
- Nothing irrelevant is promoted (human curates it)
- Memory stays high-signal over time

## Setting Up the Workflow

### Step 1: Configure Daily Logging

Ensure your agent writes to daily logs:

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      
      memory: {
        // Daily logs auto-created in memory/ subfolder
        dailyLogs: {
          enabled: true,
          path: "memory",           // Relative to workspace
          format: "YYYY-MM-DD.md",  // Date format
        },
      },
    },
  },
}
```

**Daily log structure:**
```markdown
# 2024-01-15

## Conversations
- Discussed project Alpha deadline (moved to Jan 20)
- Reviewed PR #142 — approved with minor changes

## Decisions
- Will use PostgreSQL instead of MongoDB for analytics

## Tasks Completed
- Fixed authentication bug (issue #89)
- Updated deployment docs

## Open Questions
- Should we migrate legacy users this quarter?
```

### Step 2: Configure Auto-Loading

Ensure recent context is always available:

```json5
{
  agents: {
    defaults: {
      memory: {
        dailyLogs: { enabled: true },
        
        // Auto-load recent logs
        autoLoad: {
          // Always load today and yesterday
          recentDays: 2,
          
          // Only load MEMORY.md in private sessions
          curatedInPrivateOnly: true,
        },
      },
    },
  },
}
```

### Step 3: Create MEMORY.md

Initialize your curated memory file:

```markdown
# Long-Term Memory

## About Me
- Name: [Your name]
- Working on: [Current projects]
- Preferences: [Communication style, tools, etc.]

## Key Context
[Important background the agent should always know]

## Active Projects
### Project Alpha
- Started: 2024-01-01
- Goal: [Description]
- Status: In progress

## Decisions Log
| Date | Decision | Context |
|------|----------|---------|
| 2024-01-10 | Use PostgreSQL | Analytics need complex queries |

## Patterns to Remember
- I prefer direct feedback over sugar-coating
- Morning is my most productive time
- When I say "later" I usually mean "remind me tomorrow"
```

## Agent-Suggested Curation

### Heartbeat-Based Review

Configure your agent to review daily logs during heartbeats:

```markdown
<!-- In your agent's HEARTBEAT.md -->

## During Each Heartbeat

1. Check today's daily log
2. Look for entries worth promoting to MEMORY.md:
   - Important decisions
   - New project information
   - Preference revelations
   - Recurring patterns
3. If found, suggest to human for approval

## Memory Candidate Format

When suggesting, use:
```
📝 Memory Candidate:
- **What:** [Brief description]
- **From:** [Date/source]
- **Why:** [Why this matters long-term]
- **Suggested section:** [Where in MEMORY.md]
```
```

### Skill-Based Curation (Advanced)

For more sophisticated curation, create a dedicated skill:

```markdown
<!-- skills/curator/SKILL.md -->

# Memory Curator Skill

## Purpose
Review daily logs and suggest memory candidates for human approval.

## Workflow

### 1. Scan Daily Logs
```bash
# Find logs from last 7 days
ls -la memory/*.md | head -7
```

### 2. Identify Candidates

Look for:
- Decisions with rationale
- New information about people/projects
- Stated preferences or patterns
- Corrections to existing memory
- Important dates/deadlines

### 3. Format Suggestions

```markdown
## Memory Candidates (2024-01-15)

### Candidate 1: Project deadline change
**Source:** memory/2024-01-15.md
**Content:** "Project Alpha deadline moved from Jan 15 to Jan 20 per client request"
**Suggested section:** Active Projects > Project Alpha
**Action:** [ ] Approve  [ ] Reject  [ ] Edit

### Candidate 2: Technical preference
**Source:** memory/2024-01-14.md
**Content:** "Prefer TypeScript over JavaScript for new projects"
**Suggested section:** Patterns to Remember
**Action:** [ ] Approve  [ ] Reject  [ ] Edit
```

### 4. Await Human Review
Do not auto-promote. Always wait for explicit approval.
```

## Human Review Interface

### CLI-Based Review

```bash
# View pending suggestions
openclaw memory suggestions

# Approve a suggestion
openclaw memory approve 1

# Reject a suggestion
openclaw memory reject 2

# Edit before approving
openclaw memory edit 3
```

### Chat-Based Review

Configure your agent to present suggestions conversationally:

```
Agent: I noticed some things worth remembering from today:

1. 📝 You decided to use PostgreSQL for analytics (you mentioned
   complex query requirements)

2. 📝 Project Alpha deadline moved to Jan 20

Should I add these to your long-term memory?
- "yes" or "1,2" to approve
- "no" or "skip" to dismiss
- "edit 1: [new text]" to modify
```

### Web-Based Review (GTD-Style)

For a more sophisticated interface, build a curation dashboard:

```
┌─────────────────────────────────────────────────────┐
│  Memory Curation Dashboard                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  📥 INBOX (3 pending)                               │
│  ┌─────────────────────────────────────────────┐   │
│  │ "Use PostgreSQL for analytics"              │   │
│  │ Source: 2024-01-15 • Suggested: Decisions   │   │
│  │ [✓ Approve] [✗ Reject] [✎ Edit]             │   │
│  └─────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────┐   │
│  │ "Project Alpha deadline: Jan 20"            │   │
│  │ Source: 2024-01-15 • Suggested: Projects    │   │
│  │ [✓ Approve] [✗ Reject] [✎ Edit]             │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  📊 STATS                                           │
│  • MEMORY.md: 127 entries                          │
│  • This week: 12 approved, 3 rejected              │
│  • Last reconciliation: 5 days ago                 │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## Real-World Example: Coach System

Here's how an ADHD-aware coaching system implements memory curation:

### Architecture

```
Coach System
├── coach_db.py           # CLI interface to SQLite
├── coach_db_core.py      # Database operations
└── workspace-coach/
    └── frontend/         # Next.js curation app
        ├── app/
        │   ├── inbox/    # Pending candidates
        │   ├── goals/    # Goal management
        │   ├── tasks/    # Task tracking
        │   └── projects/ # Project organization
```

### The Flow

**1. Agent generates candidates during heartbeats:**
```python
# During scheduled check-in
python3 coach_db.py thought create \
  --content "User mentioned preferring morning work sessions" \
  --source telegram \
  --type memory_candidate
```

**2. Candidates appear in web dashboard:**
```typescript
// frontend/app/inbox/page.tsx
export default function InboxPage() {
  const { data: candidates } = useQuery({
    queryKey: ['memory-candidates'],
    queryFn: () => fetch('/api/candidates').then(r => r.json()),
  });
  
  return (
    <div className="inbox">
      {candidates.map(c => (
        <CandidateCard
          key={c.id}
          candidate={c}
          onApprove={() => promoteTmemory(c.id)}
          onReject={() => dismiss(c.id)}
        />
      ))}
    </div>
  );
}
```

**3. Human reviews in GTD-style workflow:**
- Inbox → Review → Approve/Reject
- Approved items get categorized (Goal, Project, Task, Memory)
- Memory items are promoted to MEMORY.md

**4. CLI reflects changes:**
```bash
# Check what was approved
python3 coach_db.py report approved --since yesterday

# View current memory state
python3 coach_db.py memory list
```

### Why This Works

- **Agent catches everything** — Heartbeats ensure nothing is missed
- **Human filters noise** — Only valuable insights are promoted
- **GTD-style inbox** — Familiar workflow reduces friction
- **Database backing** — Candidates persist until reviewed
- **CLI + Web** — Multiple interfaces for different contexts

## Periodic Reconciliation

Over time, MEMORY.md accumulates outdated entries. Schedule periodic reviews:

### Monthly Review Checklist

```markdown
## Memory Reconciliation Checklist

### 1. Remove Outdated Entries
- [ ] Completed projects → Archive or delete
- [ ] Changed preferences → Update or remove old
- [ ] Resolved decisions → Keep if instructive, else remove

### 2. Consolidate Duplicates
- [ ] Similar entries → Merge into one
- [ ] Redundant context → Trim to essentials

### 3. Update Active Sections
- [ ] Current projects → Accurate status?
- [ ] People context → Still relevant?
- [ ] Patterns → Still true?

### 4. Check Signal-to-Noise
- [ ] Read MEMORY.md end-to-end
- [ ] Would a new agent session be well-informed?
- [ ] Any sections feel like clutter?
```

### Automated Reconciliation Prompts

Configure your agent to prompt reconciliation:

```json5
{
  agents: {
    defaults: {
      memory: {
        reconciliation: {
          // Prompt every 30 days
          intervalDays: 30,
          
          // Message to send
          prompt: "It's been a month since we reviewed MEMORY.md. Want to do a quick reconciliation?",
        },
      },
    },
  },
}
```

## Multi-Agent Curation

When running multiple agents, coordinate memory curation:

### Shared vs Agent-Specific Memory

```
~/.openclaw/
├── workspace/              # Shared workspace
│   ├── MEMORY.md           # Shared long-term memory
│   └── memory/             # Shared daily logs
│
├── workspace-coder/        # Coder-specific
│   ├── MEMORY.md           # Coder's technical memory
│   └── memory/
│
└── workspace-researcher/   # Researcher-specific
    ├── MEMORY.md           # Research findings
    └── memory/
```

### Cross-Agent Promotion

```markdown
<!-- In researcher's daily log -->

## Research Finding (2024-01-15)
Discovered that competitor X launched feature Y.
→ PROMOTE TO: shared MEMORY.md (relevant to all agents)
→ NOTIFY: @conductor (strategic decision needed)
```

### Curation Routing

```json5
{
  agents: {
    researcher: {
      memory: {
        curation: {
          // Researcher suggests, conductor approves
          suggestTo: "conductor",
          
          // Certain patterns auto-route to shared memory
          autoPromote: {
            patterns: ["PROMOTE TO: shared"],
            destination: "~/.openclaw/workspace/MEMORY.md",
          },
        },
      },
    },
  },
}
```

## Best Practices

### What to Curate

**Always promote:**
- Explicit preferences ("I prefer X over Y")
- Important decisions with rationale
- Key dates and deadlines
- Corrections ("Actually, I meant...")
- Relationship context (who is who)

**Consider promoting:**
- Recurring patterns observed
- Project status changes
- Technical choices
- Meeting outcomes

**Avoid promoting:**
- Transient tasks (use task manager)
- Conversation filler
- Temporary context
- Duplicates of existing entries

### Curation Hygiene

1. **Review weekly** — Don't let inbox pile up
2. **Edit before approving** — Rephrase for clarity
3. **Date everything** — Context decays; dates help
4. **Reconcile monthly** — Prune outdated entries
5. **Keep it scannable** — Future agents skim, not read

### Signs of Healthy Memory

✅ MEMORY.md under 500 lines
✅ Every entry has clear purpose
✅ Agent responses feel informed
✅ Rarely re-explaining same context
✅ Weekly curation takes < 5 minutes

### Signs of Unhealthy Memory

❌ MEMORY.md over 2000 lines
❌ Entries from 6+ months ago unchanged
❌ Agent keeps asking same questions
❌ Search returns irrelevant results
❌ Curation backlog growing

## Troubleshooting

### "Agent keeps forgetting things"

1. Check if daily logs are being created
2. Verify auto-loading is configured
3. Ensure MEMORY.md is in correct location
4. Check `curatedInPrivateOnly` setting

### "Memory is cluttered"

1. Schedule reconciliation session
2. Archive old project sections
3. Merge duplicate entries
4. Raise curation standards (approve less)

### "Curation takes too long"

1. Batch review weekly instead of daily
2. Configure stricter suggestion filters
3. Use keyboard shortcuts in dashboard
4. Train agent to suggest less, better

### "Agent suggestions are low quality"

1. Update heartbeat instructions
2. Provide examples of good candidates
3. Add negative examples (what NOT to suggest)
4. Consider dedicated curator skill

## Complete Configuration

```json5
// Full memory curation setup
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      
      memory: {
        // Daily logging
        dailyLogs: {
          enabled: true,
          path: "memory",
          format: "YYYY-MM-DD.md",
        },
        
        // Auto-loading
        autoLoad: {
          recentDays: 2,
          curatedInPrivateOnly: true,
        },
        
        // Curation settings
        curation: {
          // Where suggestions queue
          inbox: "~/.openclaw/workspace/memory/inbox.json",
          
          // Suggestion behavior
          suggest: {
            enabled: true,
            during: ["heartbeat", "session-end"],
            maxPerDay: 10,
          },
          
          // Approval routing
          approvalRequired: true,
          notifyChannel: "telegram",
        },
        
        // Reconciliation reminders
        reconciliation: {
          intervalDays: 30,
          prompt: "Time for monthly memory review. Want to reconcile MEMORY.md?",
        },
      },
      
      // Search configuration
      memorySearch: {
        enabled: true,
        provider: "gemini",
        hybrid: { enabled: true },
      },
    },
  },
}
```

## Next Steps

You now have curated memory. For experimental conversation-level memory, see [Session Memory](./08-session-memory.md) →
