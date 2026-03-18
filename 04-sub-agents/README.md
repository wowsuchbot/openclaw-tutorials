# Module 4: Sub-Agents

Break complex tasks into specialized workers. Orchestrate multiple agents for powerful workflows.

## What You'll Learn

1. [Delegation Basics](./01-delegation-basics.md) — Spawning and communicating with sub-agents
2. [Orchestrator Patterns](./02-orchestrator-patterns.md) — Designing multi-agent workflows
3. [Specialized Workers](./03-specialized-workers.md) — Creating purpose-built agents

## Time Required

~60 minutes for all sections

## Why Sub-Agents?

A single agent has limitations:

- **Context overflow** — Long tasks exhaust context windows
- **Skill confusion** — One agent doing many things does none well
- **No parallelism** — Sequential processing is slow

Sub-agents solve these:

```
                    ┌──────────────┐
                    │  Conductor   │
                    │  (routes &   │
                    │  coordinates)│
                    └──────┬───────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │  Coder   │      │ Curator  │      │Researcher│
   │(builds)  │      │(content) │      │ (gathers)│
   └──────────┘      └──────────┘      └──────────┘
```

## Key Concepts

### Session vs Agent

- **Agent** — Configuration: model, tools, workspace, system prompt
- **Session** — Runtime instance of an agent with conversation state

One agent can have multiple sessions. Sub-agents spawn new sessions.

### Communication Patterns

```
┌─────────────┐                      ┌─────────────┐
│   Parent    │──── sessions_spawn ──│  Sub-Agent  │
│   Session   │◄──── result ─────────│   Session   │
└─────────────┘                      └─────────────┘

┌─────────────┐                      ┌─────────────┐
│   Agent A   │──── sessions_send ───│   Agent B   │
│   Session   │      (async msg)     │   Session   │
└─────────────┘                      └─────────────┘
```

### Spawn Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| `run` | Execute task, return result, terminate | One-off tasks |
| `start` | Start session, keep running | Long-running workers |
| `connect` | Join existing session | Resume previous work |

## Quick Example

### Spawning a Sub-Agent

```json5
// Parent agent spawns coder sub-agent
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "coder",
    "mode": "run",
    "task": "Fix the authentication bug in src/auth.ts"
  }
}

// Coder does the work, returns result
// Result appears in parent's context
```

### Sending Messages Between Agents

```json5
// Conductor sends work to coder (async)
{
  "tool": "sessions_send",
  "params": {
    "sessionKey": "agent:coder:main",
    "message": "When you have a chance, refactor the login component"
  }
}

// Coder picks up message on next heartbeat
```

## Configuration Overview

### Define Multiple Agents

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    // Shared defaults
    defaults: {
      workspace: "~/.openclaw/workspace",
    },
    
    // Main orchestrator
    main: {
      model: "claude-sonnet-4-20250514",
      systemPrompt: "You are a coordinator that delegates tasks.",
    },
    
    // Specialized coder
    coder: {
      model: "claude-sonnet-4-20250514",
      workspace: "~/.openclaw/workspace-coder",
      systemPrompt: "You are a coding specialist.",
      tools: {
        sessions_spawn: false,  // Can't spawn sub-agents
      },
    },
    
    // Research agent
    researcher: {
      model: "claude-sonnet-4-20250514",
      workspace: "~/.openclaw/workspace-researcher",
      systemPrompt: "You gather information and summarize findings.",
    },
  },
}
```

### Agent Mandates

Each agent should have clear boundaries:

| Agent | Does | Doesn't |
|-------|------|---------|
| Conductor | Coordinates, approves, user interface | Write code, browse social |
| Coder | Builds, ships, manages TODOs | Post publicly, research |
| Researcher | Gathers info for other agents | Act independently |

## Session Tools

### sessions_spawn

Create a sub-agent session for a task:

```json5
{
  "tool": "sessions_spawn",
  "params": {
    "agentId": "coder",           // Which agent to spawn
    "mode": "run",                // "run" | "start" | "connect"
    "task": "Implement feature X", // Initial prompt
    "context": {                   // Optional additional context
      "branch": "feature/x",
      "deadline": "today"
    }
  }
}
```

### sessions_send

Send a message to an existing session:

```json5
{
  "tool": "sessions_send",
  "params": {
    "sessionKey": "agent:coder:main",  // Target session
    "message": "Also add tests for that feature"
  }
}
```

### sessions_list

See active sessions:

```json5
{
  "tool": "sessions_list",
  "params": {}
}

// Returns:
// [
//   { "key": "agent:coder:main", "status": "active", "lastActivity": "..." },
//   { "key": "agent:researcher:main", "status": "idle", "lastActivity": "..." }
// ]
```

## Module Contents

### [Delegation Basics](./01-delegation-basics.md)
- When to delegate vs do it yourself
- Spawn modes explained
- Passing context to sub-agents
- Handling results

### [Orchestrator Patterns](./02-orchestrator-patterns.md)
- Hub-and-spoke architecture
- Pipeline processing
- Parallel workers
- Event-driven coordination

### [Specialized Workers](./03-specialized-workers.md)
- Designing agent capabilities
- Workspace separation
- Tool restrictions
- Real-world examples (Coder, Researcher, etc.)

## Next Steps

Start with [Delegation Basics](./01-delegation-basics.md) to learn when and how to spawn sub-agents →
