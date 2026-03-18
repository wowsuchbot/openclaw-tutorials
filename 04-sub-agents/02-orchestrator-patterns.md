# Orchestrator Patterns

Design multi-agent workflows for complex tasks.

## Architecture Patterns

### Hub and Spoke

One orchestrator coordinates multiple specialists.

```
                    ┌──────────────┐
                    │  Orchestrator│
                    │  (hub)       │
                    └──────┬───────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │  Coder   │      │ Reviewer │      │ Deployer │
   │ (spoke)  │      │ (spoke)  │      │ (spoke)  │
   └──────────┘      └──────────┘      └──────────┘
```

**Configuration:**
```json5
{
  agents: {
    // Hub: coordinates, doesn't do work
    conductor: {
      model: "claude-sonnet-4-20250514",
      systemPrompt: `You coordinate work between specialists.
        
When given a task:
1. Break it into subtasks
2. Assign to appropriate specialist
3. Collect and synthesize results
4. Report to user

Never do specialist work yourself.`,
      tools: {
        sessions_spawn: true,
        sessions_send: true,
        sessions_list: true,
      },
    },
    
    // Spokes: specialists
    coder: {
      systemPrompt: "Write and fix code.",
      tools: { read: true, edit: true, bash: true },
    },
    reviewer: {
      systemPrompt: "Review code for quality and security.",
      tools: { read: true, grep: true },
    },
    deployer: {
      systemPrompt: "Handle deployments and infrastructure.",
      tools: { bash: true, read: true },
    },
  },
}
```

**Use cases:**
- Task routing based on type
- Single point of user contact
- Clear separation of concerns

### Pipeline

Sequential processing through stages.

```
┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
│ Input  │──▶│ Stage1 │──▶│ Stage2 │──▶│ Output │
│        │   │Research│   │ Draft  │   │ Polish │
└────────┘   └────────┘   └────────┘   └────────┘
```

**Workflow:**
```json5
// Stage 1: Research
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "researcher",
    "mode": "run",
    "task": "Research topic X",
  }
}
// Result: "Key findings: A, B, C"

// Stage 2: Draft (uses Stage 1 output)
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "writer",
    "mode": "run",
    "task": "Write draft based on research",
    "context": {
      "research": "Key findings: A, B, C"
    }
  }
}
// Result: "Draft document..."

// Stage 3: Polish (uses Stage 2 output)
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "editor",
    "mode": "run",
    "task": "Edit and polish draft",
    "context": {
      "draft": "Draft document..."
    }
  }
}
// Result: Final polished document
```

**Use cases:**
- Content creation workflows
- Data processing pipelines
- Review and approval chains

### Parallel Workers

Multiple agents working simultaneously on independent tasks.

```
         ┌──────────────┐
         │ Coordinator  │
         └──────┬───────┘
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
┌────────┐ ┌────────┐ ┌────────┐
│Worker 1│ │Worker 2│ │Worker 3│
│ Task A │ │ Task B │ │ Task C │
└───┬────┘ └───┬────┘ └───┬────┘
    │          │          │
    └──────────┼──────────┘
               ▼
        Collect Results
```

**Implementation:**
```json5
// Spawn all workers in parallel
[
  {
    "tool": "sessions_spawn",
    "params": {
      "agentId": "researcher",
      "mode": "run",
      "task": "Research competitor A"
    }
  },
  {
    "tool": "sessions_spawn",
    "params": {
      "agentId": "researcher",
      "mode": "run", 
      "task": "Research competitor B"
    }
  },
  {
    "tool": "sessions_spawn",
    "params": {
      "agentId": "researcher",
      "mode": "run",
      "task": "Research competitor C"
    }
  }
]

// Results return as each completes
// Coordinator synthesizes when all done
```

**Use cases:**
- Independent research tasks
- Parallel code changes
- Batch processing

### Event-Driven

Agents react to events and messages.

```
┌────────────────────────────────────────────────────────┐
│                    Message Bus                          │
└─────┬──────────────┬──────────────┬──────────────┬─────┘
      │              │              │              │
      ▼              ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Watcher  │  │ Analyzer │  │ Alerter  │  │ Archiver │
│(monitors)│  │(processes│  │(notifies)│  │ (stores) │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
```

**Configuration with heartbeats:**
```json5
{
  agents: {
    watcher: {
      systemPrompt: `Monitor for events. On detection:
1. Analyze event type
2. Send to appropriate handler via sessions_send`,
      
      heartbeat: {
        enabled: true,
        interval: "5m",  // Check every 5 minutes
      },
    },
    
    analyzer: {
      systemPrompt: "Process incoming events and extract insights.",
    },
    
    alerter: {
      systemPrompt: "Send notifications for important events.",
      tools: { message: true },
    },
  },
}
```

**Session communication:**
```json5
// Watcher detects event, sends to analyzer
{
  "tool": "sessions_send",
  "params": {
    "sessionKey": "agent:analyzer:main",
    "message": "New event: PR #123 merged to main"
  }
}

// Analyzer processes, sends to alerter
{
  "tool": "sessions_send",
  "params": {
    "sessionKey": "agent:alerter:main",
    "message": "Alert: Large PR merged. Changes: 500 lines in auth module."
  }
}
```

**Use cases:**
- Monitoring systems
- Webhook processing
- Reactive workflows

## Real-World Orchestration

### Development Team Simulation

```
User Request: "Add user settings page"
                    │
                    ▼
            ┌──────────────┐
            │  Conductor   │
            │  (PM role)   │
            └──────┬───────┘
                   │
    1. Create ticket │
    2. Assign to coder │
                   │
                   ▼
            ┌──────────────┐
            │    Coder     │
            │(implements)  │
            └──────┬───────┘
                   │
    3. Code complete │
    4. Request review │
                   │
                   ▼
            ┌──────────────┐
            │   Reviewer   │
            │ (code review)│
            └──────┬───────┘
                   │
    5. Feedback ───┼─── 6. Approved
       │           │           │
       ▼           │           ▼
┌──────────────┐   │   ┌──────────────┐
│    Coder     │   │   │   Deployer   │
│(fix issues)  │   │   │  (ship it)   │
└──────────────┘   │   └──────────────┘
                   │
                   ▼
            Report to User
```

**Conductor's workflow instructions:**
```markdown
## Development Workflow

When user requests a feature:

### 1. Planning
- Break into tasks
- Identify affected files
- Estimate complexity

### 2. Implementation
```json
sessions_spawn({
  agentId: "coder",
  mode: "run",
  task: "[feature description]",
  context: { files: [...], requirements: [...] }
})
```

### 3. Review
```json
sessions_spawn({
  agentId: "reviewer", 
  mode: "run",
  task: "Review changes in [files]",
  context: { focus: "security, performance" }
})
```

### 4. Handle Review Results
- If issues: send back to coder
- If approved: proceed to deploy

### 5. Deployment (if approved)
```json
sessions_spawn({
  agentId: "deployer",
  mode: "run", 
  task: "Deploy to staging"
})
```

### 6. Report
Summarize all steps to user.
```

### Research and Report

```
User: "Analyze our competition"
              │
              ▼
       ┌─────────────┐
       │ Coordinator │
       └──────┬──────┘
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
┌────────┐┌────────┐┌────────┐
│Research││Research││Research│
│Comp. A ││Comp. B ││Comp. C │
└───┬────┘└───┬────┘└───┬────┘
    │         │         │
    └─────────┼─────────┘
              ▼
       ┌─────────────┐
       │   Writer    │
       │(synthesize) │
       └──────┬──────┘
              │
              ▼
       ┌─────────────┐
       │   Editor    │
       │  (polish)   │
       └──────┬──────┘
              │
              ▼
       Final Report
```

**Implementation:**
```json5
// Phase 1: Parallel research
const researchTasks = [
  { agentId: "researcher", task: "Research competitor A: pricing, features, market position" },
  { agentId: "researcher", task: "Research competitor B: pricing, features, market position" },
  { agentId: "researcher", task: "Research competitor C: pricing, features, market position" },
];

// Spawn all in parallel
for (const task of researchTasks) {
  sessions_spawn({ ...task, mode: "run" });
}

// Collect results...

// Phase 2: Synthesis
sessions_spawn({
  agentId: "writer",
  mode: "run",
  task: "Create competitive analysis report",
  context: {
    research: [resultA, resultB, resultC]
  }
});

// Phase 3: Polish
sessions_spawn({
  agentId: "editor",
  mode: "run",
  task: "Polish report for executive audience",
  context: { draft: writerResult }
});
```

## Coordination Strategies

### State Tracking

Maintain workflow state in files:

```markdown
<!-- workflow-state.md -->
# Feature: User Settings Page

## Status: In Review

## Tasks
- [x] Planning (completed 2024-01-15)
- [x] Implementation (completed 2024-01-16)
- [ ] Review (in progress)
- [ ] Deployment (pending)

## Assignments
- Coder: @coder-session-123
- Reviewer: @reviewer-session-456

## Notes
- Implementation took 2 hours
- 3 files changed: settings.tsx, api/settings.ts, db/user.ts
```

### Session Keys Convention

Use consistent naming for session keys:

```
agent:{agentId}:{purpose}

Examples:
agent:coder:feature-auth
agent:researcher:competitor-analysis
agent:reviewer:pr-142
```

### Result Aggregation

Collect and structure results:

```markdown
<!-- results.md -->
# Competitive Analysis Results

## Competitor A (researcher-session-1)
- Pricing: Free tier, $10/mo pro
- Strengths: Easy onboarding
- Weaknesses: Limited integrations

## Competitor B (researcher-session-2)
- Pricing: $5/mo only
- Strengths: Low price
- Weaknesses: Missing features

## Competitor C (researcher-session-3)
- Pricing: Enterprise only
- Strengths: Full-featured
- Weaknesses: Complex, expensive
```

### Error Recovery

Handle failures gracefully:

```markdown
## On Sub-Agent Failure

1. Log the failure:
   - Which agent?
   - What task?
   - What error?

2. Decide recovery:
   - Retry: Same agent, same task
   - Reassign: Different agent
   - Escalate: Notify user

3. Update state:
   - Mark task as failed
   - Record error details
   - Adjust timeline

4. Continue workflow:
   - If recoverable: retry/reassign
   - If critical: pause and notify user
```

## Configuration Best Practices

### Agent Capabilities Matrix

```json5
{
  agents: {
    conductor: {
      // Can delegate, can't do specialist work
      tools: {
        sessions_spawn: true,
        sessions_send: true,
        sessions_list: true,
        read: true,      // Can read for context
        edit: false,     // Can't edit code
        bash: false,     // Can't run commands
      },
    },
    
    coder: {
      // Full dev access, can't delegate
      tools: {
        sessions_spawn: false,
        read: true,
        edit: true,
        write: true,
        bash: true,
        glob: true,
        grep: true,
      },
    },
    
    reviewer: {
      // Read-only, can't modify
      tools: {
        sessions_spawn: false,
        read: true,
        edit: false,
        glob: true,
        grep: true,
      },
    },
  },
}
```

### Workspace Isolation

```
~/.openclaw/
├── workspace/              # Conductor (shared state, coordination files)
│   ├── MEMORY.md
│   └── workflows/
│
├── workspace-coder/        # Coder (code-related memory)
│   ├── MEMORY.md
│   └── memory/
│
├── workspace-researcher/   # Researcher (research notes)
│   ├── MEMORY.md
│   └── memory/
│
└── workspace-reviewer/     # Reviewer (review guidelines)
    ├── MEMORY.md
    └── memory/
```

### Model Selection by Role

```json5
{
  agents: {
    // Conductor: balanced, good at planning
    conductor: {
      model: "claude-sonnet-4-20250514",
    },
    
    // Coder: best at code generation
    coder: {
      model: "claude-sonnet-4-20250514",  // Or Opus for complex tasks
    },
    
    // Researcher: fast, good at summarization
    researcher: {
      model: "claude-sonnet-4-20250514",  // Cost-efficient for high volume
    },
    
    // Reviewer: thorough analysis
    reviewer: {
      model: "claude-sonnet-4-20250514",
    },
  },
}
```

## Anti-Patterns

### Over-Delegation

❌ **Bad:** Delegating trivial tasks
```
User: "What time is it?"
Conductor: *spawns time-agent*
```

✅ **Good:** Handle simple tasks directly
```
User: "What time is it?"
Conductor: "It's 3:42 PM"
```

### Delegation Chains

❌ **Bad:** A delegates to B delegates to C delegates to D
```
Conductor → Planner → Coder → Tester → Deployer
(Too many hops, context lost)
```

✅ **Good:** Flat hierarchy
```
Conductor → Coder
Conductor → Tester  
Conductor → Deployer
```

### Missing Context

❌ **Bad:** Vague delegation
```
sessions_spawn({
  agentId: "coder",
  task: "Fix the bug"
})
```

✅ **Good:** Rich context
```
sessions_spawn({
  agentId: "coder",
  task: "Fix authentication bug",
  context: {
    error: "TokenExpired at auth.ts:45",
    repro: "Login, wait 1 hour, refresh",
    expected: "Token should auto-refresh"
  }
})
```

### No Result Handling

❌ **Bad:** Fire and forget
```
sessions_spawn({ ... })
// Never check result
```

✅ **Good:** Process results
```
const result = sessions_spawn({ ... })
if (result.error) {
  // Handle error
}
// Use result for next step
```

## Next Steps

Now design your own specialized workers in [Specialized Workers](./03-specialized-workers.md) →
