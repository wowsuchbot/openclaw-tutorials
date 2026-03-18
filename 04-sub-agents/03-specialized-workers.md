# Specialized Workers

Design purpose-built agents with focused capabilities.

## Why Specialize?

A generalist agent struggles with competing priorities:

```
Generalist Agent:
- "Should I write code or review it?"
- "Should I research or implement?"
- "Should I be thorough or fast?"
```

Specialized agents excel at specific tasks:

```
Coder: "My job is to write working code."
Reviewer: "My job is to find problems."
Researcher: "My job is to gather information."
```

## Anatomy of a Specialist

Each specialized agent needs:

1. **Clear mandate** — What it does (and doesn't do)
2. **Appropriate tools** — Only what's needed
3. **Focused workspace** — Relevant context only
4. **Specific instructions** — How to approach tasks

### Example: Coder Agent

```json5
{
  agents: {
    coder: {
      model: "claude-sonnet-4-20250514",
      workspace: "~/.openclaw/workspace-coder",
      
      systemPrompt: `You are a coding specialist.

## Your Role
Write, fix, and refactor code. You do NOT:
- Post to social media
- Send emails or messages
- Make business decisions
- Deploy to production (deployer does that)

## Approach
1. Read existing code first — understand patterns
2. Make minimal, focused changes
3. Follow existing conventions
4. Test when possible
5. Document non-obvious logic

## Output
Always report:
- What files you changed
- What the changes do
- Any concerns or trade-offs`,

      tools: {
        // Code tools
        read: true,
        edit: true,
        write: true,
        glob: true,
        grep: true,
        bash: true,
        
        // Can't delegate
        sessions_spawn: false,
        sessions_send: false,
        
        // Can't communicate externally
        message: false,
        webfetch: true,  // Can lookup docs
      },
    },
  },
}
```

### Example: Researcher Agent

```json5
{
  agents: {
    researcher: {
      model: "claude-sonnet-4-20250514",  // Cost-effective for high volume
      workspace: "~/.openclaw/workspace-researcher",
      
      systemPrompt: `You are a research specialist.

## Your Role
Gather, organize, and summarize information. You do NOT:
- Write production code
- Make decisions
- Take actions
- Contact anyone

## Approach
1. Understand what information is needed
2. Search comprehensively
3. Verify from multiple sources
4. Organize findings clearly
5. Highlight key insights

## Output
Structure all findings as:
- Summary (3-5 bullet points)
- Details (organized by topic)
- Sources (with links/references)
- Confidence (high/medium/low)`,

      tools: {
        // Research tools
        webfetch: true,
        read: true,
        glob: true,
        grep: true,
        
        // Can write research notes
        write: true,
        
        // Can't modify code
        edit: false,
        bash: false,
        
        // Can't delegate or communicate
        sessions_spawn: false,
        message: false,
      },
    },
  },
}
```

### Example: Reviewer Agent

```json5
{
  agents: {
    reviewer: {
      model: "claude-sonnet-4-20250514",
      workspace: "~/.openclaw/workspace-reviewer",
      
      systemPrompt: `You are a code review specialist.

## Your Role
Review code for quality, security, and correctness. You do NOT:
- Fix issues yourself
- Write new features
- Approve deployments

## Review Checklist
1. **Correctness** — Does it do what it should?
2. **Security** — Any vulnerabilities?
3. **Performance** — Any bottlenecks?
4. **Readability** — Is it maintainable?
5. **Testing** — Are there tests?

## Output
For each issue found:
- Severity: critical/major/minor/nitpick
- Location: file:line
- Problem: what's wrong
- Suggestion: how to fix

Conclude with: APPROVED / NEEDS CHANGES / REJECTED`,

      tools: {
        // Read-only access
        read: true,
        glob: true,
        grep: true,
        
        // Can run tests
        bash: true,
        
        // Cannot modify code
        edit: false,
        write: false,
        
        // Cannot delegate
        sessions_spawn: false,
      },
    },
  },
}
```

## Real-World Specialist: Coach

Here's a complete example from a production system — an ADHD-aware coaching agent:

### Configuration

```json5
{
  agents: {
    coach: {
      model: "claude-sonnet-4-20250514",
      workspace: "~/.openclaw/workspace-coach",
      
      systemPrompt: `You are an ADHD-aware accountability coach.

## Your Role
Help maintain focus, track goals, and provide check-ins. You operate in two modes:

### Scheduled Mode (Check-ins)
Triggered by cron. You:
1. Query current priorities from coach_db
2. Craft a supportive, focused message
3. Send via Telegram

### Unscheduled Mode (Thinking Partner)
Triggered by user request. You:
1. Listen and understand context
2. Help clarify thoughts
3. Update coach_db if needed

## Tools
- coach_db.py: Your CLI for task/goal management
- message: For sending check-ins

## Tone
- Supportive but direct
- ADHD-aware: concrete, actionable
- Celebrate wins, don't dwell on misses`,
    },
  },
}
```

### Workspace Structure

```
~/.openclaw/workspace-coach/
├── SOUL.md                 # Agent identity
├── MEMORY.md               # Coaching patterns and preferences
├── memory/                 # Daily coaching logs
├── skills/
│   └── coach/
│       ├── SKILL.md        # Skill definition
│       ├── coach_db.py     # CLI interface
│       ├── coach_db_core.py # Database layer
│       └── tests/          # Test suite
```

### Skill Definition

```markdown
<!-- skills/coach/SKILL.md -->
# Coach Skill

## Commands

### Check-in
```bash
# Get pressing priorities
python3 coach_db.py report priorities --limit 5

# Get active goals
python3 coach_db.py goal list --status active
```

### Task Management
```bash
# Complete a task
python3 coach_db.py task complete <id>

# Create a task
python3 coach_db.py task create --title "..." --priority 1
```

### Goal Tracking
```bash
# Check goal progress
python3 coach_db.py goal progress <id>

# Update goal status
python3 coach_db.py goal update <id> --status <status>
```

## Check-in Template

Morning:
"Good morning! Here's what's pressing:
[priorities from coach_db]

What's one thing you want to accomplish today?"

Evening:
"How did today go? Did you make progress on:
[today's priorities]

Any wins to celebrate?"
```

### Scheduled Triggers (Cron)

```bash
# Morning check-in (8 AM NY)
0 8 * * * /path/to/openclaw agent coach --task "Run morning check-in"

# Evening check-in (7 PM NY)
0 19 * * * /path/to/openclaw agent coach --task "Run evening check-in"
```

## Designing Your Own Specialists

### Step 1: Define the Mandate

Answer these questions:
- What does this agent DO?
- What does it NOT do?
- When is it used?
- What's its output?

```markdown
## Agent: [Name]

### Does
- [Primary responsibility]
- [Secondary responsibility]

### Does Not
- [Explicitly excluded]
- [Boundary with other agents]

### Triggered By
- [What causes this agent to run]

### Outputs
- [What it produces]
```

### Step 2: Select Tools

Only include necessary tools:

```json5
// Template: Start minimal, add as needed
{
  tools: {
    // Reading (most agents need these)
    read: true,
    glob: true,
    grep: true,
    
    // Writing (only if agent creates content)
    write: false,
    edit: false,
    
    // Execution (only if agent runs commands)
    bash: false,
    
    // External (only if agent needs outside info)
    webfetch: false,
    
    // Delegation (usually only orchestrator)
    sessions_spawn: false,
    sessions_send: false,
    
    // Communication (only if agent sends messages)
    message: false,
  },
}
```

### Step 3: Design the Workspace

```
~/.openclaw/workspace-{agent}/
├── SOUL.md              # Agent identity (if using SOUL pattern)
├── MEMORY.md            # Agent-specific long-term memory
├── memory/              # Daily logs
├── skills/              # Agent-specific skills (optional)
│   └── {skill}/
│       └── SKILL.md
└── {work-products}/     # Whatever the agent creates
```

### Step 4: Write Instructions

Template for systemPrompt:

```markdown
You are a [role] specialist.

## Your Role
[1-2 sentences on primary function]

You do NOT:
- [Explicit boundary 1]
- [Explicit boundary 2]

## Approach
1. [First step]
2. [Second step]
3. [Third step]

## Tools Available
- [tool1]: [what it's for]
- [tool2]: [what it's for]

## Output Format
[How to structure responses]

## Important
[Any critical reminders]
```

### Step 5: Test the Agent

```bash
# Direct invocation
openclaw agent {agentId} --task "Test task"

# Check it follows boundaries
openclaw agent {agentId} --task "Do something outside your mandate"
# Should refuse or redirect

# Check it uses correct tools
openclaw agent {agentId} --task "Complete a typical task" --verbose
# Should only use expected tools
```

## Specialist Patterns

### The Executor

Does work, produces artifacts.

```json5
{
  agents: {
    executor: {
      systemPrompt: "You execute tasks and produce results.",
      tools: {
        read: true,
        write: true,
        edit: true,
        bash: true,
      },
    },
  },
}
```

Examples: Coder, Writer, Designer

### The Analyzer

Examines, doesn't modify.

```json5
{
  agents: {
    analyzer: {
      systemPrompt: "You analyze and report findings. Never modify.",
      tools: {
        read: true,
        glob: true,
        grep: true,
        // No write/edit
      },
    },
  },
}
```

Examples: Reviewer, Auditor, Researcher

### The Communicator

Sends messages, manages relationships.

```json5
{
  agents: {
    communicator: {
      systemPrompt: "You handle external communication.",
      tools: {
        message: true,
        webfetch: true,
        read: true,
      },
    },
  },
}
```

Examples: Support agent, Notification agent

### The Monitor

Watches and alerts, doesn't act.

```json5
{
  agents: {
    monitor: {
      systemPrompt: "You watch for conditions and alert others.",
      heartbeat: { enabled: true, interval: "5m" },
      tools: {
        read: true,
        webfetch: true,
        sessions_send: true,  // Can alert other agents
        // No execution tools
      },
    },
  },
}
```

Examples: Log watcher, Price tracker, Availability checker

## Multi-Specialist Teams

### Team Configuration

```json5
{
  agents: {
    // Orchestrator
    main: {
      model: "claude-sonnet-4-20250514",
      systemPrompt: "Coordinate the team.",
      tools: {
        sessions_spawn: true,
        sessions_send: true,
        sessions_list: true,
      },
    },
    
    // Specialists
    coder: { /* ... */ },
    reviewer: { /* ... */ },
    researcher: { /* ... */ },
    deployer: { /* ... */ },
  },
}
```

### Routing Table

Document in orchestrator's instructions:

```markdown
## Agent Routing

| Task Type | Agent | Notes |
|-----------|-------|-------|
| Write/fix code | coder | Include file paths |
| Review changes | reviewer | After coder completes |
| Research topic | researcher | Can parallelize |
| Deploy changes | deployer | Needs approval first |
| Security audit | auditor | For sensitive changes |
```

### Handoff Protocol

```markdown
## Between Agents

When handing off:
1. Summarize completed work
2. List open questions
3. Provide relevant context
4. Specify expected output

Example handoff to reviewer:
"Coder completed auth feature. Changes:
- src/auth.ts (new file)
- src/api/login.ts (modified)
- tests/auth.test.ts (new file)

Please review for security and correctness.
Focus: Token handling, password hashing"
```

## Troubleshooting Specialists

### Agent Does Too Much

**Symptom:** Agent performs tasks outside its mandate

**Fix:** Stronger boundaries in systemPrompt
```markdown
## You MUST NOT
- Write code (coder does this)
- Send messages (communicator does this)
- Make decisions (orchestrator does this)

If asked to do these, respond:
"That's outside my role. Please delegate to [agent]."
```

### Agent Does Too Little

**Symptom:** Agent refuses valid tasks

**Fix:** Clarify scope
```markdown
## You CAN
- Read any file in the workspace
- Run test commands
- Write analysis to memory/

## You CANNOT
- Modify source code
- Run deployment commands
```

### Agent Uses Wrong Tools

**Symptom:** Agent calls tools it shouldn't

**Fix:** Disable at config level
```json5
{
  tools: {
    edit: false,  // Explicitly disabled
    bash: false,
  },
}
```

### Agent Loses Context

**Symptom:** Agent forgets instructions mid-task

**Fix:** Add reminders to systemPrompt
```markdown
## Remember
At each step, verify you're staying within your mandate:
- Am I doing my job, not someone else's?
- Am I using only my allowed tools?
- Is my output in the expected format?
```

---

## Module Complete

You've completed the Sub-Agents module! You now understand:

1. **Delegation Basics** — When and how to spawn sub-agents
2. **Orchestrator Patterns** — Hub-spoke, pipeline, parallel, event-driven
3. **Specialized Workers** — Designing focused agents with clear mandates

**Next module:** [Skills and Plugins](../05-skills-and-plugins/README.md) — Extend agent capabilities with reusable components →
