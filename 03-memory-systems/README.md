# Module 3: Memory Systems

OpenClaw's memory system is **layered and sophisticated** — from plain markdown files to vector embeddings to hybrid search with diversity and recency tuning.

## What You'll Learn

1. [Memory Architecture](./01-memory-architecture.md) — Overview of all memory layers
2. [Markdown Memory](./02-markdown-memory.md) — `MEMORY.md` and daily logs
3. [Vector Memory Setup](./03-vector-memory-setup.md) — Embeddings configuration
4. [Hybrid Search](./04-hybrid-search.md) — BM25 + vector combination
5. [Advanced Search](./05-advanced-search.md) — MMR, temporal decay, tuning
6. [QMD Backend](./06-qmd-backend.md) — Optional advanced search sidecar
7. [Memory Curation](./07-memory-curation.md) — Agent-suggested, human-curated workflow
8. [Session Memory](./08-session-memory.md) — Experimental transcript indexing

## Time Required

~90 minutes for all sections

## Why Memory Matters

Without memory, every conversation starts fresh. Your agent forgets:
- User preferences
- Past decisions
- Project context
- Lessons learned

With proper memory:
- **Continuity** — Agent remembers across sessions
- **Context** — Relevant information surfaces automatically
- **Learning** — Patterns and insights persist

## The Memory Stack

```
┌─────────────────────────────────────────────────────────┐
│                     MEMORY TOOLS                        │
│            memory_search    memory_get                  │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    HYBRID SEARCH                        │
│         Vector (semantic) + BM25 (keyword)              │
│         + MMR (diversity) + Temporal decay              │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   VECTOR INDEX                          │
│            SQLite + sqlite-vec embeddings               │
│       Provider: Gemini / OpenAI / Voyage / Local        │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  MARKDOWN FILES                         │
│   MEMORY.md (curated)  +  memory/YYYY-MM-DD.md (daily)  │
└─────────────────────────────────────────────────────────┘
```

## Quick Start

### Minimal Memory Setup

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      memorySearch: {
        enabled: true,
        provider: "gemini",  // or "openai", "local"
      },
    },
  },
}
```

### Create Your First Memory

```bash
# Create memory directory
mkdir -p ~/.openclaw/workspace/memory

# Create long-term memory file
cat > ~/.openclaw/workspace/MEMORY.md << 'EOF'
# Long-Term Memory

## Preferences
- Prefer concise answers
- Use code examples when helpful

## Projects
- Working on OpenClaw tutorials

## Important Context
- User is transitioning from single to multi-user deployment
EOF

# Create today's daily log
cat > ~/.openclaw/workspace/memory/$(date +%Y-%m-%d).md << 'EOF'
# Daily Log

## Session Notes
- Started OpenClaw memory configuration
- Testing vector search setup
EOF
```

### Test Memory Search

Message your bot:
```
"What projects am I working on?"
```

The agent should use `memory_search` to find "OpenClaw tutorials" from your `MEMORY.md`.

## Memory Tools

Agents have two memory tools:

| Tool | Purpose | Example |
|------|---------|---------|
| `memory_search` | Semantic search across all memory files | "Find notes about project deadlines" |
| `memory_get` | Read specific file or line range | "Get lines 10-20 of MEMORY.md" |

### memory_search

```json
// Agent tool call
{
  "tool": "memory_search",
  "params": {
    "query": "project deadline",
    "limit": 5
  }
}

// Returns snippets with file path, line range, score
```

### memory_get

```json
// Agent tool call
{
  "tool": "memory_get",
  "params": {
    "path": "memory/2026-03-18.md",
    "startLine": 10,
    "lines": 20
  }
}
```

## Memory File Layout

```
~/.openclaw/workspace/
├── MEMORY.md              # Curated long-term memory
│                          # Loaded in main/private sessions only
│
├── memory/                # Daily logs directory
│   ├── 2026-03-18.md      # Today's log (auto-loaded)
│   ├── 2026-03-17.md      # Yesterday's log (auto-loaded)
│   ├── 2026-03-16.md      # Older logs (searchable)
│   └── ...
│
└── memory.md              # Fallback (lowercase, deprecated)
                           # Only used if MEMORY.md doesn't exist
```

## Configuration Reference

### Basic

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "gemini",
      },
    },
  },
}
```

### Full Configuration

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      
      memorySearch: {
        enabled: true,
        
        // Embedding provider
        provider: "gemini",  // "gemini" | "openai" | "voyage" | "mistral" | "ollama" | "local"
        model: "gemini-embedding-001",
        
        // Remote API settings
        remote: {
          apiKey: "YOUR_API_KEY",  // or from env
          baseUrl: null,           // custom endpoint
        },
        
        // Hybrid search (vector + keyword)
        query: {
          hybrid: {
            enabled: true,
            vectorWeight: 0.7,
            textWeight: 0.3,
            
            // Post-processing
            mmr: {
              enabled: true,
              lambda: 0.7,
            },
            temporalDecay: {
              enabled: true,
              halfLifeDays: 30,
            },
          },
        },
        
        // Additional paths to index
        extraPaths: ["../team-docs", "/srv/notes"],
        
        // Storage
        store: {
          path: "~/.openclaw/memory/{agentId}.sqlite",
        },
      },
    },
  },
}
```

## Module Contents

### [Memory Architecture](./01-memory-architecture.md)
Overview of how all memory components fit together.

### [Markdown Memory](./02-markdown-memory.md)
- `MEMORY.md` format and best practices
- Daily logs (`memory/YYYY-MM-DD.md`)
- When to write what where
- Auto-loading behavior

### [Vector Memory Setup](./03-vector-memory-setup.md)
- Choosing an embedding provider
- API key configuration
- Local vs remote embeddings
- Index storage and reindexing

### [Hybrid Search](./04-hybrid-search.md)
- Why combine vector + keyword search
- Weight configuration
- When keyword beats semantic (and vice versa)

### [Advanced Search](./05-advanced-search.md)
- MMR re-ranking for diversity
- Temporal decay for recency
- Tuning for your use case

### [QMD Backend](./06-qmd-backend.md)
- What is QMD?
- Installation and setup
- BM25 + reranking advantages
- When to use QMD vs built-in

### [Memory Curation](./07-memory-curation.md)
- Agent-suggested, human-curated workflow
- Integration with Coach system
- Workspace-coach frontend
- Periodic reconciliation strategies

### [Session Memory](./08-session-memory.md)
- Indexing conversation transcripts
- Delta-based sync
- Privacy considerations

## Next Steps

Start with [Memory Architecture](./01-memory-architecture.md) to understand the full picture →
