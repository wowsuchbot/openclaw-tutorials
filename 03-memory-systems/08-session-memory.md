# Session Memory

Experimental feature: index conversation transcripts for cross-session recall.

> ⚠️ **Experimental:** Session memory is in active development. APIs and behavior may change.

## What is Session Memory?

Standard memory (MEMORY.md, daily logs) captures curated knowledge. Session memory captures *everything* — the full conversation history.

**Without session memory:**
```
Session 1: "We discussed refactoring the auth module"
Session 2: "What did we discuss about auth?" → Agent has no idea
```

**With session memory:**
```
Session 1: "We discussed refactoring the auth module"
Session 2: "What did we discuss about auth?" 
         → Agent searches transcripts
         → Finds exact conversation
         → "In our Jan 15 session, we discussed moving auth to a separate service..."
```

## Use Cases

### When Session Memory Helps

- **Long-running projects** — Recall decisions from weeks ago
- **Debugging sessions** — "What did we try last time?"
- **Learning journeys** — "Show me when I first asked about X"
- **Accountability** — "Did I commit to doing Y?"

### When Session Memory is Overkill

- **Short interactions** — One-off questions
- **Private conversations** — Sensitive content you don't want indexed
- **High-volume channels** — Group chats with lots of noise

## Enabling Session Memory

### Basic Configuration

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      sessionMemory: {
        enabled: true,
        
        // Where to store transcripts
        store: {
          path: "~/.openclaw/sessions/{agentId}",
        },
      },
    },
  },
}
```

### With Search Integration

```json5
{
  agents: {
    defaults: {
      sessionMemory: {
        enabled: true,
        
        // Index transcripts for search
        index: {
          enabled: true,
          
          // Use same provider as memory search
          provider: "gemini",
          
          // Separate index from memory search
          store: {
            path: "~/.openclaw/sessions/{agentId}.sqlite",
          },
        },
      },
    },
  },
}
```

## How It Works

### Transcript Storage

Each session is saved as a structured file:

```
~/.openclaw/sessions/main/
├── 2024-01-15-abc123.json
├── 2024-01-15-def456.json
├── 2024-01-16-ghi789.json
└── index.sqlite
```

**Transcript format:**
```json
{
  "sessionId": "abc123",
  "agentId": "main",
  "channel": "telegram",
  "peerId": "1231002024",
  "startTime": "2024-01-15T10:30:00Z",
  "endTime": "2024-01-15T11:45:00Z",
  "messages": [
    {
      "role": "user",
      "content": "Let's discuss the auth refactor",
      "timestamp": "2024-01-15T10:30:00Z"
    },
    {
      "role": "assistant",
      "content": "Sure! What aspects are you considering?",
      "timestamp": "2024-01-15T10:30:15Z"
    }
  ]
}
```

### Search Flow

```
Query: "What did we discuss about auth?"
                ↓
    ┌───────────┴───────────┐
    ↓                       ↓
Memory Search         Session Search
(MEMORY.md, logs)     (transcripts)
    ↓                       ↓
┌─────────┐           ┌─────────┐
│Curated  │           │Full     │
│context  │           │convos   │
└────┬────┘           └────┬────┘
     └─────────┬───────────┘
               ↓
        Merged Results
```

## Configuration Options

### Retention Policy

Control how long transcripts are kept:

```json5
{
  sessionMemory: {
    enabled: true,
    
    retention: {
      // Keep transcripts for 90 days
      maxAgeDays: 90,
      
      // Or keep last N sessions
      maxSessions: 1000,
      
      // Or keep up to N MB
      maxSizeMB: 500,
      
      // Cleanup schedule
      cleanupInterval: "daily",
    },
  },
}
```

### Filtering

Control what gets indexed:

```json5
{
  sessionMemory: {
    enabled: true,
    
    filter: {
      // Only index these channels
      channels: ["telegram", "discord-dm"],
      
      // Exclude certain peers
      excludePeers: ["group-chat-123"],
      
      // Minimum session length to index
      minMessages: 3,
      
      // Exclude sessions with certain patterns
      excludePatterns: ["password", "secret", "api.key"],
    },
  },
}
```

### Privacy Controls

Sensitive content handling:

```json5
{
  sessionMemory: {
    enabled: true,
    
    privacy: {
      // Redact before indexing
      redact: {
        enabled: true,
        patterns: [
          "\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b",  // emails
          "\\b\\d{3}-\\d{2}-\\d{4}\\b",  // SSN
          "sk-[a-zA-Z0-9]{32,}",  // API keys
        ],
        replacement: "[REDACTED]",
      },
      
      // Don't index marked messages
      skipMarked: true,  // Messages with [PRIVATE] are skipped
    },
  },
}
```

## Searching Sessions

### Via Agent

The agent automatically searches sessions when relevant:

```
User: "What did we decide about the database migration?"

Agent: *searches session memory*

Agent: "In our conversation on January 12th, we decided to:
1. Use pg_dump for the initial export
2. Run migration during low-traffic hours (2-4 AM)
3. Keep the old database as read-only backup for 30 days

You mentioned wanting to test on staging first. Did you complete that?"
```

### Via CLI

```bash
# Search sessions
openclaw sessions search "database migration"

# List recent sessions
openclaw sessions list --limit 10

# View specific session
openclaw sessions view abc123

# Export session
openclaw sessions export abc123 --format markdown
```

### Search Options

```bash
# Filter by date range
openclaw sessions search "auth" --after 2024-01-01 --before 2024-01-15

# Filter by channel
openclaw sessions search "auth" --channel telegram

# Show more context
openclaw sessions search "auth" --context 5  # 5 messages before/after
```

## Integration with Memory Curation

Session memory and curated memory work together:

### Promotion Flow

```
Session transcript
     ↓
Agent identifies important exchange
     ↓
Suggests for curation
     ↓
Human approves
     ↓
Added to MEMORY.md (summarized)
     ↓
Original transcript remains searchable
```

### Configuration

```json5
{
  sessionMemory: {
    enabled: true,
    
    // Connect to curation workflow
    curation: {
      // Suggest notable exchanges
      suggestFromSessions: true,
      
      // Criteria for suggestions
      suggestWhen: {
        // User explicitly asked to remember
        explicitRequest: true,
        
        // Decisions were made
        decisionsDetected: true,
        
        // Long detailed explanations
        substantialExchange: {
          minTurns: 5,
          minTokens: 500,
        },
      },
    },
  },
}
```

## Performance Considerations

### Index Size

Session transcripts can grow large:

| Sessions/day | Retention | Approx Size |
|--------------|-----------|-------------|
| 5 | 30 days | 50-100 MB |
| 5 | 90 days | 150-300 MB |
| 20 | 30 days | 200-400 MB |
| 20 | 90 days | 600 MB - 1 GB |

### Search Latency

Session search adds latency:

```
Memory search only:    50-200ms
With session search:   100-400ms
```

### Optimization

```json5
{
  sessionMemory: {
    index: {
      // Chunk size for indexing
      chunkSize: 256,  // Smaller = more precise, larger index
      
      // Search parameters
      search: {
        topK: 5,  // Fewer results = faster
        threshold: 0.5,  // Higher = fewer, better matches
      },
    },
    
    // Background indexing
    indexing: {
      mode: "background",  // Don't block session end
      batchSize: 10,  // Index 10 sessions at once
    },
  },
}
```

## Multi-Agent Sessions

When using sub-agents, decide how to handle transcripts:

### Separate Indexes (Default)

```json5
{
  agents: {
    main: {
      sessionMemory: { enabled: true },
    },
    coder: {
      sessionMemory: { enabled: true },  // Separate index
    },
  },
}
```

### Shared Index

```json5
{
  agents: {
    defaults: {
      sessionMemory: {
        enabled: true,
        store: {
          // All agents share one index
          path: "~/.openclaw/sessions/shared.sqlite",
        },
      },
    },
  },
}
```

### Selective Sharing

```json5
{
  agents: {
    main: {
      sessionMemory: {
        enabled: true,
        // Main can search all
        searchIndexes: ["main", "coder", "researcher"],
      },
    },
    coder: {
      sessionMemory: {
        enabled: true,
        // Coder only searches own + main
        searchIndexes: ["coder", "main"],
      },
    },
  },
}
```

## Troubleshooting

### "Sessions not being saved"

```bash
# Check if enabled
openclaw config get agents.defaults.sessionMemory

# Check storage path exists
ls -la ~/.openclaw/sessions/

# Check logs for errors
openclaw logs | grep -i session
```

### "Search not finding sessions"

```bash
# Check index status
openclaw sessions status

# Rebuild index
openclaw sessions reindex

# Verify session was indexed
openclaw sessions list --verbose
```

### "Search is slow"

1. Reduce retention period
2. Increase search threshold
3. Reduce topK results
4. Use background indexing

### "Index too large"

```bash
# Check current size
du -sh ~/.openclaw/sessions/

# Cleanup old sessions
openclaw sessions cleanup --older-than 30d

# Or reduce retention in config
```

## Limitations

**Current limitations:**
- No real-time streaming (indexed after session ends)
- No cross-agent session search without explicit configuration
- Embedding cost scales with transcript volume
- No summarization (full transcripts only)

**Planned improvements:**
- Incremental indexing during session
- Automatic summarization for old sessions
- Smarter chunking by conversation topic
- Cost optimization for high-volume use

## Complete Configuration

```json5
// Full session memory setup
{
  agents: {
    defaults: {
      sessionMemory: {
        enabled: true,
        
        // Storage
        store: {
          path: "~/.openclaw/sessions/{agentId}",
        },
        
        // Indexing
        index: {
          enabled: true,
          provider: "gemini",
          chunkSize: 256,
          store: {
            path: "~/.openclaw/sessions/{agentId}.sqlite",
          },
        },
        
        // Retention
        retention: {
          maxAgeDays: 90,
          maxSizeMB: 500,
          cleanupInterval: "daily",
        },
        
        // Filtering
        filter: {
          channels: ["telegram", "discord-dm"],
          minMessages: 3,
        },
        
        // Privacy
        privacy: {
          redact: {
            enabled: true,
            patterns: ["sk-[a-zA-Z0-9]+"],
          },
          skipMarked: true,
        },
        
        // Curation integration
        curation: {
          suggestFromSessions: true,
          suggestWhen: {
            explicitRequest: true,
            decisionsDetected: true,
          },
        },
        
        // Performance
        indexing: {
          mode: "background",
          batchSize: 10,
        },
      },
    },
  },
}
```

## Summary

Session memory extends your agent's recall to full conversation history:

| Feature | Standard Memory | Session Memory |
|---------|-----------------|----------------|
| Content | Curated highlights | Full transcripts |
| Signal | High (human-filtered) | Variable |
| Size | Small (KB-MB) | Large (MB-GB) |
| Latency | Fast | Slower |
| Privacy | Controlled | Needs filtering |
| Use case | Core context | Deep recall |

**Recommendation:** Start with standard memory (MEMORY.md + daily logs). Add session memory when you need to recall specific past conversations.

---

## Module Complete

You've completed the Memory Systems module! You now understand:

1. **Memory Architecture** — How markdown files and vector search work together
2. **Markdown Memory** — MEMORY.md structure and daily logs
3. **Vector Memory** — Embedding providers and semantic search
4. **Hybrid Search** — Combining BM25 and vector search
5. **Advanced Search** — MMR, temporal decay, and tuning
6. **QMD Backend** — Optional high-performance search sidecar
7. **Memory Curation** — Agent-suggested, human-curated workflows
8. **Session Memory** — Experimental transcript indexing

**Next module:** [Sub-Agents](../04-sub-agents/README.md) — Delegation, orchestration, and multi-agent patterns →
