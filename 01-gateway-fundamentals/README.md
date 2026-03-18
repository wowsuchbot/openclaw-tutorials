# Module 1: Gateway Fundamentals

The OpenClaw Gateway is the always-on process that connects your AI agent to messaging channels, manages sessions, and orchestrates tool execution.

## What You'll Learn

1. [Installation](./01-installation.md) — Getting OpenClaw running
2. [Configuration Basics](./02-configuration-basics.md) — Understanding `openclaw.json`
3. [Telegram Setup](./03-telegram-setup.md) — Connecting your first channel
4. [Essential Commands](./04-essential-commands.md) — CLI reference

## Time Required

~30 minutes for all sections

## Prerequisites

- Node.js 20+ installed (`node --version`)
- A Telegram account
- API key for at least one LLM provider

## Key Concepts

### What is the Gateway?

The Gateway is a single, always-on Node.js process that:

- **Routes messages** from channels (Telegram, Discord, etc.) to your agent
- **Manages sessions** — conversation state, history, context
- **Executes tools** — file operations, web search, browser automation
- **Handles authentication** — API keys, OAuth, channel tokens

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Telegram   │────▶│   Gateway   │────▶│  LLM API    │
│  Discord    │◀────│  (port      │◀────│  (OpenAI,   │
│  WhatsApp   │     │   18789)    │     │   Claude)   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Workspace  │
                    │  (files,    │
                    │   memory)   │
                    └─────────────┘
```

### Configuration File

OpenClaw uses **JSON5** format (comments and trailing commas allowed):

```json5
// ~/.openclaw/openclaw.json
{
  // This is a comment!
  gateway: {
    port: 18789,  // trailing comma OK
  },
}
```

### Hot Reload

The Gateway supports **hot reload** for safe changes:

- **Hot-apply** (no restart): model changes, tool policies, memory settings
- **Auto-restart** (restart required): channel credentials, port changes

Send `SIGHUP` to reload:
```bash
kill -HUP $(pgrep -f "openclaw gateway")
```

Or use the CLI:
```bash
openclaw gateway restart
```

## Quick Start

```bash
# Install
npm install -g openclaw

# Initialize configuration
openclaw init

# Start gateway
openclaw gateway

# In another terminal, check status
openclaw status
```

## Next Steps

Start with [Installation](./01-installation.md) →
