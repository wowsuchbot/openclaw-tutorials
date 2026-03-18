---
layout: default
title: Home
---

# OpenClaw Tutorial Series

A comprehensive guide to mastering OpenClaw — from basic gateway setup to advanced multi-agent architectures with sophisticated memory systems.

## Who This Is For

- **Single-user operators** wanting to understand the platform deeply
- **Developers** transitioning from single-user to multi-user deployments
- **Power users** building custom agents, skills, and plugins

---

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

### Social & Collaboration

| Module | Topic | Time | Key Outcome |
|--------|-------|------|-------------|
| [06](./06-group-chats/) | **Group Chat Handling** | 45 min | Multi-user chat etiquette + access control |
| [07](./07-farcaster-setup/) | **Farcaster Setup** | 60 min | Decentralized social integration |

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
        provider: "gemini",
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

### [Module 1: Gateway Fundamentals](./01-gateway-fundamentals/)

Learn to install, configure, and run the OpenClaw gateway with Telegram integration.

**Topics:** Installation, configuration (JSON5), Telegram bot setup, essential CLI commands

### [Module 2: Session Management](./02-session-management/)

Understand session isolation — critical for transitioning from single-user to multi-user deployments.

**Topics:** Session keys, `dmScope` setting, migration patterns, security considerations

### [Module 3: Memory Systems](./03-memory-systems/)

Deep dive into OpenClaw's memory architecture — from markdown files to vector search to hybrid retrieval.

**Topics:** Memory files, vector embeddings, hybrid search, MMR, temporal decay, QMD backend, memory curation workflow

### [Module 4: Sub-Agents](./04-sub-agents/)

Learn to delegate tasks to specialized sub-agents for complex workflows.

**Topics:** `sessions_spawn`, orchestrator patterns, specialized workers

### [Module 5: Skills & Plugins](./05-skills-and-plugins/)

Extend OpenClaw with custom instructions (skills) or new capabilities (plugins).

**Topics:** SKILL.md format, MCP plugins, decision framework, Coach case study

### [Module 6: Group Chat Handling](./06-group-chats/)

Navigate multi-user conversations with grace — when to participate, when to stay silent, and how to enforce access control.

**Topics:** Participation principles, response triggers, access control patterns, multi-agent groups

### [Module 7: Farcaster Setup](./07-farcaster-setup/)

Integrate your agents with Farcaster, the decentralized social protocol, using the Neynar API.

**Topics:** Neynar API setup, posting casts/threads, reading feeds, engagement workflows

---

## Prerequisites

- Node.js 20+ (22+ recommended)
- A Telegram account (for channel setup)
- API key for at least one LLM provider (OpenAI, Anthropic, Google, etc.)
- Optional: API key for embeddings (Gemini, OpenAI, Voyage, or local GGUF model)

## Resources

- [Official OpenClaw Docs](https://docs.openclaw.ai)
- [Configuration Reference](https://docs.openclaw.ai/gateway/configuration-reference)
- [GitHub Repository](https://github.com/wowsuchbot/openclaw-tutorials)

---

*Built with care for the OpenClaw community*
