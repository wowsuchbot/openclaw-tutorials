# Delegation Basics

Learn when to delegate, how to spawn sub-agents, and how to handle results.

## When to Delegate

### Good Candidates for Delegation

**Complex, self-contained tasks:**
```
"Implement user authentication with OAuth2 support"
→ Spawn coder: clear scope, defined output
```

**Parallel research:**
```
"Research competitors A, B, and C"
→ Spawn 3 researchers in parallel
```

**Specialized work:**
```
"Review this PR for security issues"
→ Spawn security-focused agent with specific tools
```

### Poor Candidates for Delegation

**Quick lookups:**
```
"What's in config.json?"
→ Just read it yourself (faster)
```

**Conversational context needed:**
```
"Continue what we were discussing"
→ Sub-agent lacks conversation history
```

**Approval required:**
```
"Post this to Twitter"
→ Parent should handle approval workflows
```

### Decision Framework

```
Is the task...
├─ Self-contained? (clear inputs/outputs)
│  ├─ Yes → Consider delegating
│  └─ No → Handle yourself
│
├─ Time-consuming? (minutes, not seconds)
│  ├─ Yes → Delegation worthwhile
│  └─ No → Overhead not worth it
│
├─ Specialized? (needs specific tools/context)
│  ├─ Yes → Delegate to specialist
│  └─ No → Any agent can handle
│
└─ Parallelizable? (multiple independent parts)
   ├─ Yes → Spawn multiple workers
   └─ No → Sequential is fine
```

## Spawn Modes

### Mode: run

Execute a task and return result. Session terminates after.

```json5
// Spawn, wait for result, sub-agent exits
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "coder",
    "mode": "run",
    "task": "Fix bug #123 in auth.ts"
  }
}

// Result returned directly:
// "Fixed bug #123: The issue was an off-by-one error in token validation.
//  Changes: src/auth.ts (lines 45-52)"
```

**Use for:**
- One-off tasks
- Clear completion criteria
- No follow-up expected

### Mode: start

Start a session that keeps running (background worker).

```json5
// Start and get session key for later communication
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "researcher",
    "mode": "start",
    "task": "Monitor competitor announcements and summarize weekly"
  }
}

// Returns session key:
// { "sessionKey": "agent:researcher:monitor-123" }

// Later, send messages:
{
  "tool": "sessions_send",
  "params": {
    "sessionKey": "agent:researcher:monitor-123",
    "message": "Add competitor D to monitoring list"
  }
}
```

**Use for:**
- Long-running tasks
- Periodic check-ins
- Stateful workers

### Mode: connect

Connect to an existing session (resume previous work).

```json5
// Connect to existing session
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "coder",
    "mode": "connect",
    "sessionKey": "agent:coder:feature-x",
    "task": "Continue where you left off on feature X"
  }
}
```

**Use for:**
- Resuming interrupted work
- Multiple interactions with same context
- Stateful conversations

## Passing Context

### Basic Context

```json5
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "coder",
    "mode": "run",
    "task": "Implement the login form",
    "context": {
      "requirements": "Must support email and OAuth",
      "design": "See figma.com/design/123",
      "deadline": "End of day"
    }
  }
}
```

### File Context

```json5
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "coder",
    "mode": "run",
    "task": "Refactor this file",
    "context": {
      "files": ["src/auth.ts", "src/utils/validation.ts"],
      "readInstructions": "Read these files before starting"
    }
  }
}
```

### Conversation Context

```json5
// Pass relevant conversation snippets
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "coder",
    "mode": "run",
    "task": "Implement what we discussed",
    "context": {
      "priorDiscussion": "User wants authentication with:\n- Email/password\n- Google OAuth\n- Remember me option",
      "decisions": [
        "Use Passport.js",
        "Store sessions in Redis"
      ]
    }
  }
}
```

### Memory Sharing

Sub-agents can access their own workspace memory:

```json5
// Coder has separate workspace with own MEMORY.md
{
  agents: {
    coder: {
      workspace: "~/.openclaw/workspace-coder",
      // Has access to:
      // ~/.openclaw/workspace-coder/MEMORY.md
      // ~/.openclaw/workspace-coder/memory/*.md
    },
  },
}
```

For shared context, use explicit file paths or `extraPaths`:

```json5
{
  agents: {
    coder: {
      workspace: "~/.openclaw/workspace-coder",
      memorySearch: {
        // Also search shared workspace
        extraPaths: ["~/.openclaw/workspace/shared"],
      },
    },
  },
}
```

## Handling Results

### Synchronous Results (mode: run)

```json5
// Spawn and wait
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "coder",
    "mode": "run",
    "task": "Count lines of code in src/"
  }
}

// Result appears in parent's context:
// "Total lines: 15,432
//  - TypeScript: 12,100
//  - CSS: 2,500  
//  - Other: 832"

// Parent can use result immediately
```

### Asynchronous Results (mode: start)

```json5
// Start background worker
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "researcher",
    "mode": "start",
    "task": "Research X and report back"
  }
}

// Returns immediately with session key
// Worker sends results via sessions_send when done

// Configure agent to receive async results:
```

```markdown
<!-- In researcher's instructions -->
When task is complete, send results back:
{
  "tool": "sessions_send",
  "params": {
    "sessionKey": "agent:main:123",
    "message": "Research complete: [summary]"
  }
}
```

### Error Handling

```json5
// Spawn might fail
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "nonexistent",
    "mode": "run",
    "task": "Do something"
  }
}

// Error result:
// { "error": "Agent 'nonexistent' not found" }
```

Handle gracefully:
```
If spawn fails:
1. Log the error
2. Try alternative agent or approach
3. Report to user if critical
```

### Result Parsing

Sub-agents should return structured results when possible:

```markdown
<!-- In sub-agent's instructions -->
When completing tasks, format results as:

## Result
[Brief summary]

## Details
[Specifics]

## Next Steps
[If any]
```

Parent can parse and act on structure:
```
Coder result indicates "Next Steps: Add tests"
→ Either spawn another task or note for later
```

## Real-World Example

### Bug Fix Workflow

```
User: "There's a bug in the login flow"
            │
            ▼
      ┌─────────────┐
      │ Conductor   │
      │ (main)      │
      └─────┬───────┘
            │
            │ 1. Analyze bug report
            │ 2. Gather context
            │
            ▼
      ┌─────────────────────────────────────┐
      │ sessions_spawn                       │
      │ agentId: coder                       │
      │ mode: run                            │
      │ task: "Fix login bug. User reports   │
      │        auth fails after password     │
      │        reset. Check auth.ts"         │
      │ context:                             │
      │   - Error logs attached              │
      │   - "User reset password yesterday"  │
      └─────────────────────────────────────┘
            │
            ▼
      ┌─────────────┐
      │   Coder     │
      │ (sub-agent) │
      └─────┬───────┘
            │
            │ 1. Read auth.ts
            │ 2. Identify issue (stale token)
            │ 3. Fix and test
            │ 4. Return result
            │
            ▼
      ┌─────────────────────────────────────┐
      │ Result:                              │
      │ "Fixed: Token wasn't invalidated on  │
      │  password reset. Added invalidation  │
      │  in auth.ts:67. Tested locally."     │
      └─────────────────────────────────────┘
            │
            ▼
      ┌─────────────┐
      │ Conductor   │
      │ (receives)  │
      └─────┬───────┘
            │
            │ Report to user
            │
            ▼
User: "Bug fixed! The issue was..."
```

### Configuration for This Workflow

```json5
{
  agents: {
    main: {
      model: "claude-sonnet-4-20250514",
      systemPrompt: `You are the conductor. When users report bugs:
1. Gather context (error messages, steps to reproduce)
2. Delegate to coder with clear task description
3. Report results back to user`,
      tools: {
        sessions_spawn: true,
        sessions_send: true,
      },
    },
    
    coder: {
      model: "claude-sonnet-4-20250514",
      workspace: "~/.openclaw/workspace-coder",
      systemPrompt: `You are a coding specialist. When given bugs:
1. Read relevant files
2. Identify root cause
3. Implement fix
4. Test if possible
5. Return clear summary of changes`,
      tools: {
        sessions_spawn: false,  // Coder doesn't delegate
        read: true,
        edit: true,
        bash: true,
      },
    },
  },
}
```

## Best Practices

### Clear Task Descriptions

**Bad:**
```
"Fix it"
```

**Good:**
```
"Fix the authentication bug where login fails after password reset.
 Error: TokenInvalidError in auth.ts
 Steps to reproduce: Reset password, try to login
 Expected: Login succeeds with new password"
```

### Appropriate Context

**Too little:**
```
"Implement the feature"
→ Sub-agent has no idea what feature
```

**Too much:**
```
"Here's the entire conversation history (10,000 tokens)..."
→ Overwhelms context, slow, expensive
```

**Just right:**
```
"Implement OAuth login. Requirements:
- Google and GitHub providers
- Store user in existing User model
- See auth/ folder for patterns"
```

### Result Expectations

Tell sub-agents what you expect back:

```json5
{
  "task": "Research competitor pricing",
  "context": {
    "expectations": "Return a table with: Company, Free tier, Paid plans, Key features"
  }
}
```

### Timeout Handling

Long tasks should have checkpoints:

```json5
{
  "task": "Migrate the database",
  "context": {
    "checkpoints": "Report progress every 100 records",
    "timeout": "30 minutes max"
  }
}
```

## Next Steps

Now that you understand delegation basics, learn about orchestrating multiple agents in [Orchestrator Patterns](./02-orchestrator-patterns.md) →
