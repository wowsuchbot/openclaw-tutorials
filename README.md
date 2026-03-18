# OpenClaw Tutorial Series

A comprehensive guide to mastering OpenClaw — from basic gateway setup to advanced multi-agent architectures with sophisticated memory systems.

## Who This Is For

- **Single-user operators** wanting to understand the platform deeply
- **Developers** transitioning from single-user to multi-user deployments
- **Power users** building custom agents, skills, and plugins

## Learning Path

### Foundation (Start Here)

| Module | Topic | Time | Key Outcome |
|--------|-------|------|-------------|
| [01](./01-gateway-fundamentals/) | **Gateway Fundamentals** | 30 min | Running gateway with Telegram |
| [02](./02-session-management/) | **Session Management** | 45 min | Multi-user safe configuration |

### Core Systems

| Module | Topic | Time | Key Outcome |
|--------|-------|------|-------------|
| [03](./03-memory-systems/) | **Memory Systems** | 90 min | Hybrid search + curation workflow |
| [04](./04-sub-agents/) | **Sub-Agents** | 45 min | Task delegation patterns |

### Extension

| Module | Topic | Time | Key Outcome |
|--------|-------|------|-------------|
| [05](./05-skills-and-plugins/) | **Skills & Plugins** | 60 min | Custom capabilities |

---

## Quick Reference

### Essential Files

```
~/.openclaw/
├── openclaw.json          # Main configuration (JSON5)
├── workspace/             # Default agent workspace
│   ├── MEMORY.md          # Long-term curated memory
│   └── memory/            # Daily logs (YYYY-MM-DD.md)
├── agents/                # Per-agent directories
│   └── <agentId>/
│       ├── sessions/      # Session transcripts
│       └── agent/         # Agent-specific config
└── memory/                # Vector index storage
    └── <agentId>.sqlite   # Embeddings database
```

### Key Config Patterns

```json5
// ~/.openclaw/openclaw.json
{
  // Gateway basics
  gateway: { port: 18789 },
  
  // CRITICAL for multi-user: isolate sessions by sender
  session: { dmScope: "per-channel-peer" },
  
  // Telegram channel
  channels: {
    telegram: {
      enabled: true,
      botToken: "YOUR_TOKEN",
      dmPolicy: "pairing",
    },
  },
  
  // Memory search with hybrid (vector + keyword)
  agents: {
    defaults: {
      memorySearch: {
        provider: "gemini",  // or "openai", "local"
        query: {
          hybrid: {
            enabled: true,
            vectorWeight: 0.7,
            textWeight: 0.3,
          },
        },
      },
    },
  },
}
```

### CLI Essentials

```bash
# Start gateway
openclaw gateway

# Check status
openclaw status

# Telegram pairing
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>

# View logs
openclaw logs --follow

# Configuration validation
openclaw doctor
openclaw doctor --fix
```

---

## Module Summaries

### Module 1: Gateway Fundamentals

Learn to install, configure, and run the OpenClaw gateway with Telegram integration.

**Topics:**
- Installation and first run
- Configuration file structure (JSON5)
- Hot reload and restart behavior
- Telegram bot setup via BotFather
- Essential CLI commands

### Module 2: Session Management

Understand session isolation — critical for transitioning from single-user to multi-user deployments.

**Topics:**
- Session keys and lifecycle
- The `dmScope` setting (why `main` is unsafe for multi-user)
- Migration patterns: single → multi-user
- Security considerations and context leak prevention

### Module 3: Memory Systems

Deep dive into OpenClaw's memory architecture — from markdown files to vector search to hybrid retrieval.

**Topics:**
- Memory file layout (`MEMORY.md`, daily logs)
- Vector embeddings setup (Gemini, OpenAI, local)
- Hybrid search (BM25 + vector)
- Advanced tuning: MMR (diversity), temporal decay (recency)
- QMD backend for BM25 + reranking
- **Memory curation workflow** (agent-suggested, human-curated)
- Experimental: session transcript indexing

### Module 4: Sub-Agents

Learn to delegate tasks to specialized sub-agents for complex workflows.

**Topics:**
- `sessions_spawn` basics
- One-shot runs vs persistent sessions
- Orchestrator patterns and `maxSpawnDepth`
- Case study: Coach as ephemeral sub-agent

### Module 5: Skills & Plugins

Extend OpenClaw with custom instructions (skills) or new capabilities (plugins).

**Topics:**
- Skills: `SKILL.md` format, prompt injection
- Plugins: TypeScript modules, tool registration
- Decision framework: when to use which
- Case study: Coach skill implementation

---

## Prerequisites

- Node.js 20+ (22+ recommended)
- A Telegram account (for channel setup)
- API key for at least one LLM provider (OpenAI, Anthropic, Google, etc.)
- Optional: API key for embeddings (Gemini, OpenAI, Voyage, or local GGUF model)

## Documentation Resources

- [Official Docs](https://docs.openclaw.ai)
- [Configuration Reference](https://docs.openclaw.ai/gateway/configuration-reference)
- [Tools Reference](https://docs.openclaw.ai/tools)

---

## About This Tutorial Series

These tutorials are designed for a specific learning journey:

1. **From single-user to multi-user** — You're running a personal bot but want to open it up safely
2. **Telegram-focused** — Examples use Telegram, but patterns apply to other channels
3. **Memory-centric** — Emphasis on building effective long-term memory systems
4. **Practical patterns** — Based on real-world implementations (Coach system, curation workflows)

Each module includes:
- Annotated configuration snippets
- Full working examples
- "Why this matters" explanations
- Common mistakes to avoid
- Your specific use case highlighted

---

*Last updated: 2026-03-18*
