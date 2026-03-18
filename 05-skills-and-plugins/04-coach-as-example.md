# Coach as Example

A complete skill implementation: ADHD-aware coaching with database, CLI, and curation workflow.

## Overview

The Coach skill demonstrates:
- Skill file structure
- SQLite database integration
- CLI tool design
- Cron-based scheduling
- GTD-style curation frontend
- Two operational modes (scheduled/unscheduled)

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Coach System                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   SKILL.md  │    │ coach_db.py │    │  coach.db   │     │
│  │(instructions)│───▶│   (CLI)     │───▶│  (SQLite)   │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│         │                                     ▲             │
│         │                                     │             │
│         ▼                                     │             │
│  ┌─────────────┐                    ┌─────────────┐        │
│  │   Agent     │                    │  Frontend   │        │
│  │(interprets) │                    │ (curation)  │        │
│  └─────────────┘                    └─────────────┘        │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────┐                                           │
│  │  Telegram   │                                           │
│  │ (messages)  │                                           │
│  └─────────────┘                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Directory Structure

```
~/.openclaw/workspace-coach/
├── SOUL.md                     # Agent identity
├── MEMORY.md                   # Long-term coaching context
├── memory/                     # Daily coaching logs
│
├── skills/
│   └── coach/
│       ├── SKILL.md            # Main skill instructions
│       ├── COACH_ARCHITECTURE.md # Architecture docs
│       ├── coach_db.py         # CLI interface (30KB)
│       ├── coach_db_core.py    # Database layer (35KB)
│       └── tests/              # Test suite
│
└── frontend/                   # Next.js curation app
    └── app/
        ├── inbox/              # Pending items
        ├── goals/              # Goal management
        ├── tasks/              # Task tracking
        └── projects/           # Project organization
```

## The Skill File

```markdown
<!-- skills/coach/SKILL.md -->
# Coach Skill

## Identity

You are an ADHD-aware accountability coach. You help maintain focus, track goals, and provide supportive check-ins.

## Two Modes

### Scheduled Mode (Check-ins)
Triggered by cron jobs. Proactive outreach at:
- 8:00 AM — Morning check-in
- 1:00 PM — Midday check-in
- 7:00 PM — Evening check-in

Process:
1. Query database for pressing priorities
2. Craft supportive, focused message
3. Send via Telegram

### Unscheduled Mode (Thinking Partner)
Triggered by user request. Reactive support:
1. Listen and understand context
2. Help clarify thoughts
3. Update database if needed

## Database Commands

### Check Priorities
```bash
python3 skills/coach/coach_db.py report priorities --limit 5
```

### List Active Goals
```bash
python3 skills/coach/coach_db.py goal list --status active
```

### Complete a Task
```bash
python3 skills/coach/coach_db.py task complete <task_id>
```

### Create a Task
```bash
python3 skills/coach/coach_db.py task create \
  --title "Task description" \
  --priority 1 \
  --goal-id <optional_goal_id>
```

### Create a Thought (Memory Candidate)
```bash
python3 skills/coach/coach_db.py thought create \
  --content "Something worth remembering" \
  --source telegram \
  --type memory_candidate
```

## Check-in Templates

### Morning
```
Good morning! Here's what's pressing:

[Query: report priorities --limit 3]

What's one thing you want to accomplish today?
```

### Midday
```
Quick midday check-in!

How's progress on:
[Query: report priorities --limit 3]

Any blockers I can help with?
```

### Evening
```
How did today go?

Did you make progress on:
[Today's priorities]

Any wins to celebrate? 🎉
```

## Tone Guidelines

- **Supportive but direct** — Acknowledge struggles, maintain focus
- **ADHD-aware** — Concrete, actionable, short
- **Celebrate wins** — Even small progress matters
- **Don't dwell on misses** — Reframe and move forward

## Memory Curation

When you notice something worth remembering:
1. Create a thought with `--type memory_candidate`
2. It will appear in the curation inbox
3. Human reviews and approves/rejects
4. Approved items are promoted to MEMORY.md
```

## The CLI Tool

### Design Principles

```python
# coach_db.py - CLI interface

"""
Design principles:
1. One command, one action
2. Consistent output format
3. Exit codes for scripting
4. Human-readable by default
5. JSON output available (--json)
"""

import click
import sys
from coach_db_core import CoachDB

@click.group()
@click.option('--db', default='~/.openclaw/workspace-coach/memory/coach.db')
@click.pass_context
def cli(ctx, db):
    """Coach database CLI."""
    ctx.obj = CoachDB(db)

# Task commands
@cli.group()
def task():
    """Task management."""
    pass

@task.command('create')
@click.option('--title', required=True)
@click.option('--priority', type=int, default=2)
@click.option('--goal-id', type=int, default=None)
@click.pass_obj
def task_create(db, title, priority, goal_id):
    """Create a new task."""
    task = db.create_task(title=title, priority=priority, goal_id=goal_id)
    click.echo(f"✓ Created task #{task.id}: {task.title}")

@task.command('complete')
@click.argument('task_id', type=int)
@click.pass_obj
def task_complete(db, task_id):
    """Mark a task as complete."""
    task = db.complete_task(task_id)
    click.echo(f"✓ Completed task #{task.id}: {task.title}")

@task.command('list')
@click.option('--status', default='pending')
@click.option('--limit', type=int, default=10)
@click.option('--json', 'as_json', is_flag=True)
@click.pass_obj
def task_list(db, status, limit, as_json):
    """List tasks."""
    tasks = db.list_tasks(status=status, limit=limit)
    if as_json:
        click.echo(json.dumps([t.to_dict() for t in tasks]))
    else:
        for t in tasks:
            click.echo(f"[{t.id}] P{t.priority} {t.title}")

# Report commands
@cli.group()
def report():
    """Generate reports."""
    pass

@report.command('priorities')
@click.option('--limit', type=int, default=5)
@click.pass_obj
def report_priorities(db, limit):
    """Show pressing priorities."""
    items = db.get_priorities(limit=limit)
    if not items:
        click.echo("No pressing priorities. Nice!")
    else:
        for item in items:
            click.echo(f"• [{item.type}] {item.title}")
```

### Usage Examples

```bash
# Check what's pressing
$ python3 coach_db.py report priorities
• [task] Finish tutorial series
• [goal] Ship v2.0 by end of month
• [task] Review PR #142

# Complete a task
$ python3 coach_db.py task complete 18
✓ Completed task #18: Review PR #142

# Create a thought for curation
$ python3 coach_db.py thought create \
  --content "User prefers morning check-ins over midday" \
  --source telegram \
  --type memory_candidate
✓ Created thought #42

# List goals with JSON output
$ python3 coach_db.py goal list --status active --json
[{"id": 1, "title": "Ship v2.0", "status": "active", ...}]
```

## The Database Layer

### Schema

```python
# coach_db_core.py - Database layer

class CoachDB:
    def __init__(self, db_path):
        self.db_path = Path(db_path).expanduser()
        self._init_db()
    
    def _init_db(self):
        with self._get_connection() as conn:
            conn.executescript('''
                CREATE TABLE IF NOT EXISTS goals (
                    id INTEGER PRIMARY KEY,
                    title TEXT NOT NULL,
                    category TEXT,
                    status TEXT DEFAULT 'active',
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    completed_at TIMESTAMP
                );
                
                CREATE TABLE IF NOT EXISTS tasks (
                    id INTEGER PRIMARY KEY,
                    title TEXT NOT NULL,
                    priority INTEGER DEFAULT 2,
                    status TEXT DEFAULT 'pending',
                    goal_id INTEGER REFERENCES goals(id),
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    completed_at TIMESTAMP
                );
                
                CREATE TABLE IF NOT EXISTS thoughts (
                    id INTEGER PRIMARY KEY,
                    content TEXT NOT NULL,
                    source TEXT,
                    type TEXT DEFAULT 'note',
                    status TEXT DEFAULT 'pending',
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                );
                
                CREATE INDEX IF NOT EXISTS idx_tasks_status 
                    ON tasks(status);
                CREATE INDEX IF NOT EXISTS idx_thoughts_status 
                    ON thoughts(status, type);
            ''')
```

### Core Operations

```python
# Task operations
def create_task(self, title, priority=2, goal_id=None):
    with self._get_connection() as conn:
        cursor = conn.execute('''
            INSERT INTO tasks (title, priority, goal_id)
            VALUES (?, ?, ?)
        ''', (title, priority, goal_id))
        return self.get_task(cursor.lastrowid)

def complete_task(self, task_id):
    with self._get_connection() as conn:
        conn.execute('''
            UPDATE tasks 
            SET status = 'completed', completed_at = CURRENT_TIMESTAMP
            WHERE id = ?
        ''', (task_id,))
        return self.get_task(task_id)

# Priority report
def get_priorities(self, limit=5):
    with self._get_connection() as conn:
        # P1 tasks first, then active goals, then other tasks
        results = conn.execute('''
            SELECT 'task' as type, id, title, priority
            FROM tasks WHERE status = 'pending' AND priority = 1
            UNION ALL
            SELECT 'goal' as type, id, title, 0 as priority
            FROM goals WHERE status = 'active'
            UNION ALL
            SELECT 'task' as type, id, title, priority
            FROM tasks WHERE status = 'pending' AND priority > 1
            ORDER BY priority
            LIMIT ?
        ''', (limit,)).fetchall()
        return [PriorityItem(**dict(r)) for r in results]
```

## Cron-Based Scheduling

### Cron Configuration

```bash
# /etc/cron.d/coach

# Morning check-in (8 AM New York)
0 8 * * * root /path/to/openclaw agent coach --task "Run morning check-in"

# Midday check-in (1 PM New York, skip Wed/Thu/Sat)
0 13 * * 0,1,2,5,6 root /path/to/openclaw agent coach --task "Run midday check-in"

# Evening check-in (7 PM New York)
0 19 * * * root /path/to/openclaw agent coach --task "Run evening check-in"
```

### How Cron Triggers Work

```
Cron fires at 8:00 AM
         │
         ▼
OpenClaw spawns Coach agent
         │
         ▼
Coach agent reads SKILL.md
         │
         ▼
Coach queries: coach_db.py report priorities
         │
         ▼
Coach constructs message from template
         │
         ▼
Coach sends via message tool to Telegram
         │
         ▼
Agent session terminates
```

## The Curation Frontend

### Purpose

The frontend provides a GTD-style interface for:
- Reviewing memory candidates (inbox)
- Managing goals and projects
- Tracking tasks
- Approving items to promote to MEMORY.md

### Structure

```
frontend/
├── app/
│   ├── layout.tsx          # App shell
│   ├── page.tsx            # Dashboard
│   │
│   ├── inbox/
│   │   └── page.tsx        # Memory candidates
│   │
│   ├── goals/
│   │   └── page.tsx        # Goal management
│   │
│   ├── tasks/
│   │   └── page.tsx        # Task list
│   │
│   └── api/
│       ├── candidates/     # CRUD for thoughts
│       ├── goals/          # CRUD for goals
│       └── tasks/          # CRUD for tasks
│
└── components/
    ├── CandidateCard.tsx   # Review card
    ├── GoalList.tsx        # Goal display
    └── TaskItem.tsx        # Task row
```

### Inbox Component

```tsx
// app/inbox/page.tsx
'use client';

import { useQuery, useMutation } from '@tanstack/react-query';

export default function InboxPage() {
  const { data: candidates } = useQuery({
    queryKey: ['candidates', 'pending'],
    queryFn: () => fetch('/api/candidates?status=pending').then(r => r.json()),
  });

  const approve = useMutation({
    mutationFn: (id: number) => 
      fetch(`/api/candidates/${id}/approve`, { method: 'POST' }),
    onSuccess: () => queryClient.invalidateQueries(['candidates']),
  });

  const reject = useMutation({
    mutationFn: (id: number) =>
      fetch(`/api/candidates/${id}/reject`, { method: 'POST' }),
    onSuccess: () => queryClient.invalidateQueries(['candidates']),
  });

  return (
    <div className="inbox">
      <h1>Inbox ({candidates?.length ?? 0})</h1>
      
      {candidates?.map(c => (
        <div key={c.id} className="candidate-card">
          <p className="content">{c.content}</p>
          <p className="meta">
            From: {c.source} • {formatDate(c.created_at)}
          </p>
          <div className="actions">
            <button onClick={() => approve.mutate(c.id)}>
              ✓ Approve
            </button>
            <button onClick={() => reject.mutate(c.id)}>
              ✗ Reject
            </button>
          </div>
        </div>
      ))}
    </div>
  );
}
```

### Curation Workflow

```
Agent notices something worth remembering
         │
         ▼
Coach creates thought:
  coach_db.py thought create --type memory_candidate
         │
         ▼
Thought appears in frontend inbox
         │
         ▼
Human reviews:
  ├─ Approve → Promoted to MEMORY.md
  ├─ Reject → Marked as rejected
  └─ Edit → Modified, then approve/reject
         │
         ▼
Memory updated for future sessions
```

## Testing

### Unit Tests

```python
# tests/test_tasks.py

def test_create_task(db):
    task = db.create_task(title="Test task", priority=1)
    assert task.id is not None
    assert task.title == "Test task"
    assert task.priority == 1
    assert task.status == "pending"

def test_complete_task(db):
    task = db.create_task(title="To complete")
    completed = db.complete_task(task.id)
    assert completed.status == "completed"
    assert completed.completed_at is not None

def test_priorities_ordering(db):
    # P1 tasks should come first
    db.create_task(title="P2 task", priority=2)
    db.create_task(title="P1 task", priority=1)
    
    priorities = db.get_priorities(limit=2)
    assert priorities[0].title == "P1 task"
    assert priorities[1].title == "P2 task"
```

### Integration Tests

```python
# tests/test_integration.py

def test_full_workflow(db, agent):
    # Create a goal
    db.create_goal(title="Ship feature", category="work")
    
    # Create related tasks
    db.create_task(title="Write code", priority=1, goal_id=1)
    db.create_task(title="Write tests", priority=2, goal_id=1)
    
    # Simulate morning check-in
    priorities = db.get_priorities(limit=3)
    assert len(priorities) == 3  # 1 goal + 2 tasks
    
    # Complete a task
    db.complete_task(1)
    
    # Check updated priorities
    priorities = db.get_priorities(limit=3)
    assert len(priorities) == 2  # 1 goal + 1 task
```

## Key Lessons

### 1. Skills Can Be Powerful

This skill demonstrates that "just markdown instructions" can orchestrate:
- Database operations
- Scheduled triggers
- Message delivery
- Memory curation

### 2. CLI Is the Bridge

The CLI tool (`coach_db.py`) bridges:
- Natural language instructions (SKILL.md)
- Structured data (SQLite)
- Multiple interfaces (agent, frontend, cron)

### 3. Two Modes Work Well

Separating scheduled (proactive) from unscheduled (reactive) modes:
- Keeps each mode focused
- Allows different behaviors
- Prevents mode confusion

### 4. Human-in-the-Loop Curation

Agent suggests, human approves:
- Agents catch everything
- Humans filter noise
- Memory stays high-signal

### 5. Test Your Skills

Skills with code need tests:
- Unit tests for database operations
- Integration tests for workflows
- Manual tests for agent interpretation

---

## Module Complete

You've completed the Skills and Plugins module! You now understand:

1. **Skills Overview** — In-context instruction modules
2. **Plugin Basics** — External tool integration via MCP
3. **When to Use Which** — Decision framework
4. **Coach as Example** — Complete skill implementation

---

## Tutorial Series Complete! 🎉

Congratulations! You've completed the OpenClaw tutorial series:

| Module | Topics |
|--------|--------|
| 1. Gateway Fundamentals | Installation, configuration, Telegram setup |
| 2. Session Management | DM scope, multi-user safety, security |
| 3. Memory Systems | Markdown, vector search, hybrid, curation |
| 4. Sub-Agents | Delegation, orchestration, specialists |
| 5. Skills and Plugins | Extensions, MCP, Coach case study |

### What's Next?

- **Build something!** Start with a simple skill, add memory, grow from there
- **Join the community** — Share your agents, learn from others
- **Read the docs** — [docs.openclaw.ai](https://docs.openclaw.ai) for reference
- **Contribute** — Skills, plugins, and tutorials welcome

Happy building! 🚀
