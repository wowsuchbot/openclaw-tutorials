# Memory Architecture

Understanding how OpenClaw's memory layers work together.

## Design Philosophy

OpenClaw's memory is **plain Markdown** at its core. The files are the source of truth — vector indexes are just acceleration layers on top.

This means:
- **Human-readable** — You can browse and edit memory files directly
- **Portable** — Copy files to a new machine, memory comes with
- **Version-controllable** — Git track your memory evolution
- **Agent-writable** — Agent can update memory using standard file tools

## Memory Layers

### Layer 1: Markdown Files (Source of Truth)

```
~/.openclaw/workspace/
├── MEMORY.md           # Long-term curated memory
└── memory/
    ├── 2026-03-18.md   # Daily log (today)
    ├── 2026-03-17.md   # Daily log (yesterday)
    └── ...             # Older logs
```

**Key insight:** The agent "remembers" only what gets written to disk.

### Layer 2: Session Context Loading

At session start, OpenClaw automatically loads:

| File | When Loaded |
|------|-------------|
| `memory/YYYY-MM-DD.md` (today) | Always |
| `memory/YYYY-MM-DD.md` (yesterday) | Always |
| `MEMORY.md` | Main/private sessions only |

This gives the agent immediate context without searching.

### Layer 3: Vector Index (Search Acceleration)

```
~/.openclaw/memory/main.sqlite
```

Contains:
- Chunked text (~400 tokens per chunk)
- Embedding vectors for each chunk
- File path and line number metadata
- Provider/model fingerprint

**Auto-updates:** Watches memory files, reindexes on change (debounced).

### Layer 4: Hybrid Search Layer

Combines:
- **Vector search** — Semantic similarity ("meaning")
- **BM25 search** — Keyword relevance ("exact terms")

### Layer 5: Post-Processing

- **MMR** — Maximal Marginal Relevance for diversity
- **Temporal decay** — Boost recent memories

## Data Flow

### Writing Memory

```
Agent writes to memory/2026-03-18.md
         │
         ▼
    File watcher detects change (1.5s debounce)
         │
         ▼
    Chunker splits into ~400 token chunks
         │
         ▼
    Embedding provider generates vectors
         │
         ▼
    SQLite stores chunks + vectors
```

### Searching Memory

```
Agent calls memory_search("project deadline")
         │
         ▼
    Query embedded by same provider
         │
         ▼
    Parallel retrieval:
    ├── Vector: top K by cosine similarity
    └── BM25: top K by keyword relevance
         │
         ▼
    Merge with configurable weights
         │
         ▼
    Optional: MMR re-ranking (diversity)
         │
         ▼
    Optional: Temporal decay (recency)
         │
         ▼
    Return top results with snippets
```

## File Types and Purposes

### MEMORY.md — Long-Term Curated Memory

**Purpose:** Durable facts, preferences, decisions, lessons learned

**Format:**
```markdown
# Long-Term Memory

## Preferences
- User prefers concise answers
- Timezone: America/New_York

## Projects
- OpenClaw tutorials (active)
- Cryptoart curation

## People
- Alice: collaborator on Project X
- Bob: technical reviewer

## Decisions
- 2026-03-15: Chose Gemini for embeddings (cost/quality balance)

## Lessons Learned
- Always verify dmScope before opening to users
```

**When to write here:**
- Facts that should persist indefinitely
- User preferences and context
- Important decisions and rationale
- Patterns and lessons

### memory/YYYY-MM-DD.md — Daily Logs

**Purpose:** Running notes, session context, ephemeral observations

**Format:**
```markdown
# 2026-03-18

## Morning
- Discussed tutorial structure
- Decided on 5-module approach

## Afternoon
- Implemented memory configuration
- User requested more examples

## Tasks
- [ ] Add hybrid search examples
- [x] Document dmScope settings

## Notes
- User seems interested in Coach integration
```

**When to write here:**
- Today's conversations and context
- Temporary notes and reminders
- Work in progress
- Things that might become MEMORY.md entries

### Relationship Between Files

```
Daily Logs (raw input)
     │
     │  Periodic review / agent suggestion
     ▼
MEMORY.md (curated output)
```

The pattern:
1. **Capture** everything in daily logs
2. **Review** periodically (during heartbeats, or manually)
3. **Promote** important items to MEMORY.md
4. **Forget** transient details (they're still searchable, just not auto-loaded)

## Index Storage

### SQLite Database

```
~/.openclaw/memory/main.sqlite

Tables:
  chunks      — text content, file path, line numbers
  embeddings  — vector data (or in vec0 virtual table)
  metadata    — provider, model, chunk params
```

### When Reindexing Occurs

Automatic:
- File content changes (watcher, debounced)
- Provider/model configuration changes
- Chunking parameters change

Manual:
```bash
openclaw memory reindex
```

### Vector Storage Modes

**With sqlite-vec (default when available):**
- Vectors stored in `vec0` virtual table
- Distance queries run in SQLite
- Fast even with large indexes

**Fallback (no sqlite-vec):**
- Vectors stored as blobs
- Loaded into memory for search
- Works but slower for large datasets

## Provider Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Memory Manager                        │
└─────────────────────────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │  Gemini  │    │  OpenAI  │    │  Local   │
    │ Provider │    │ Provider │    │ Provider │
    └──────────┘    └──────────┘    └──────────┘
          │                │                │
          ▼                ▼                ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Gemini   │    │ OpenAI   │    │node-llama│
    │   API    │    │   API    │    │   cpp    │
    └──────────┘    └──────────┘    └──────────┘
```

**Provider selection priority:**
1. Explicit `memorySearch.provider` in config
2. Auto-detect based on available API keys
3. Fallback chain: OpenAI → Gemini → Voyage → Mistral → disabled

## Scope and Security

### What Gets Indexed

By default:
- `MEMORY.md` (workspace root)
- `memory.md` (fallback, lowercase)
- `memory/**/*.md` (all daily logs)

With `extraPaths`:
```json5
{
  memorySearch: {
    extraPaths: ["../team-docs", "/srv/notes"],
  },
}
```

### What's Excluded

- Files outside workspace (unless in extraPaths)
- Non-markdown files (unless multimodal enabled)
- Symlinks (security measure)

### Scope Restrictions

```json5
{
  memorySearch: {
    scope: {
      default: "deny",
      rules: [
        { action: "allow", match: { chatType: "direct" } },
        { action: "deny", match: { keyPrefix: "discord:channel:" } },
      ],
    },
  },
}
```

This prevents memory search in group chats while allowing in DMs.

## Memory Tools

### memory_search

```
Agent: I need to find information about project deadlines

Tool call: memory_search
  query: "project deadline"
  limit: 5

Result:
  - memory/2026-03-15.md:12-18 (score: 0.89)
    "Project X deadline is March 30th..."
  - MEMORY.md:45-50 (score: 0.72)
    "Key deadlines tracked in Notion..."
```

### memory_get

```
Agent: Let me read the full context from that file

Tool call: memory_get
  path: "memory/2026-03-15.md"
  startLine: 10
  lines: 20

Result:
  [Full content of lines 10-30]
```

## Summary

| Layer | Purpose | Storage |
|-------|---------|---------|
| Markdown files | Source of truth, human-readable | Workspace filesystem |
| Auto-load | Immediate context | Session memory |
| Vector index | Fast semantic search | SQLite database |
| Hybrid search | Best of both worlds | Runtime merge |
| Post-processing | Diversity + recency | Runtime filters |

## Next Steps

Learn about the foundation: [Markdown Memory](./02-markdown-memory.md) →
