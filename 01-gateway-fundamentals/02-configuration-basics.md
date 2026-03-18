# Configuration Basics

Understanding and customizing `openclaw.json` — the heart of your OpenClaw setup.

## Configuration File Location

Default: `~/.openclaw/openclaw.json`

Override with:
```bash
openclaw gateway --config /custom/path/openclaw.json
```

## JSON5 Format

OpenClaw uses **JSON5** — a superset of JSON that allows:

```json5
{
  // Comments are allowed!
  gateway: {
    port: 18789,  // Trailing commas OK
  },
  
  /* Multi-line
     comments too */
  models: {
    default: "anthropic/claude-sonnet-4-20250514",
  },
}
```

**Why JSON5?**
- Comments document your config
- Trailing commas prevent diff noise
- Unquoted keys are cleaner to read

## Configuration Structure

```json5
{
  // ═══════════════════════════════════════════════════════════════════
  // GATEWAY — Core process settings
  // ═══════════════════════════════════════════════════════════════════
  gateway: {
    port: 18789,              // WebSocket + HTTP port
    host: "127.0.0.1",        // Bind address (localhost only by default)
  },

  // ═══════════════════════════════════════════════════════════════════
  // MODELS — LLM provider configuration
  // ═══════════════════════════════════════════════════════════════════
  models: {
    default: "anthropic/claude-sonnet-4-20250514",  // Primary model
    providers: {
      anthropic: {
        apiKey: "sk-ant-...",   // Or use ANTHROPIC_API_KEY env var
      },
      openai: {
        apiKey: "sk-...",       // Or use OPENAI_API_KEY env var
      },
      google: {
        apiKey: "...",          // Or use GEMINI_API_KEY env var
      },
    },
  },

  // ═══════════════════════════════════════════════════════════════════
  // SESSION — Session isolation and management
  // ═══════════════════════════════════════════════════════════════════
  session: {
    // CRITICAL for multi-user! See Module 2 for details
    dmScope: "per-channel-peer",  // Isolate by channel + sender
  },

  // ═══════════════════════════════════════════════════════════════════
  // CHANNELS — Messaging platform connections
  // ═══════════════════════════════════════════════════════════════════
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc...",
      dmPolicy: "pairing",      // Require approval for new users
    },
  },

  // ═══════════════════════════════════════════════════════════════════
  // AGENTS — Agent configuration
  // ═══════════════════════════════════════════════════════════════════
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",  // Agent working directory
      memorySearch: {
        enabled: true,
        provider: "gemini",     // Embedding provider
      },
    },
    list: [
      // Additional agents (see Module 4)
    ],
  },

  // ═══════════════════════════════════════════════════════════════════
  // TOOLS — Tool access control
  // ═══════════════════════════════════════════════════════════════════
  tools: {
    profile: "coding",          // Base tool set
    deny: ["browser"],          // Explicitly deny specific tools
  },
}
```

## Key Configuration Sections

### Models

Configure which LLM to use and how to authenticate:

```json5
models: {
  // Format: "provider/model-name"
  default: "anthropic/claude-sonnet-4-20250514",
  
  // Fallback chain (if primary fails)
  fallbacks: [
    "openai/gpt-4o",
    "google/gemini-2.0-flash",
  ],
  
  providers: {
    anthropic: {
      // API key can be inline or from environment
      apiKey: "sk-ant-...",
      // Or reference env var explicitly:
      // apiKey: { env: "ANTHROPIC_API_KEY" },
    },
  },
}
```

**Supported providers:**
- `anthropic` — Claude models
- `openai` — GPT models
- `google` — Gemini models
- `xai` — Grok models
- Custom OpenAI-compatible endpoints

### Channels

Connect to messaging platforms:

```json5
channels: {
  telegram: {
    enabled: true,
    botToken: "123456:ABC-DEF...",
    
    // Access control
    dmPolicy: "pairing",        // "pairing" | "allowlist" | "open" | "disabled"
    allowFrom: ["123456789"],   // Numeric user IDs (for allowlist mode)
    
    // Group behavior
    groups: {
      "*": { requireMention: true },  // Default: require @mention
    },
  },
}
```

### Agents

Configure agent behavior and resources:

```json5
agents: {
  defaults: {
    // Where the agent reads/writes files
    workspace: "~/.openclaw/workspace",
    
    // System prompt additions
    systemPrompt: "You are a helpful assistant.",
    
    // Memory search configuration
    memorySearch: {
      enabled: true,
      provider: "gemini",
    },
    
    // Concurrent request limit
    maxConcurrent: 3,
  },
  
  // Additional named agents
  list: [
    {
      id: "coder",
      workspace: "~/.openclaw/workspace-coder",
      systemPrompt: "You are a coding assistant.",
    },
  ],
}
```

### Tools

Control which tools agents can use:

```json5
tools: {
  // Preset profiles: "minimal" | "coding" | "messaging" | "full"
  profile: "coding",
  
  // Fine-grained control
  allow: ["group:fs", "group:web"],   // Explicitly allow
  deny: ["browser"],                   // Explicitly deny
  
  // Tool groups available:
  // - group:runtime (exec, bash, process)
  // - group:fs (read, write, edit, apply_patch)
  // - group:sessions (list, history, send, spawn)
  // - group:memory (memory_search, memory_get)
  // - group:web (web_search, web_fetch)
  // - group:ui (browser, canvas)
}
```

## Hot Reload Behavior

OpenClaw supports **hybrid hot reload**:

| Change Type | Behavior |
|-------------|----------|
| Model selection | Hot-apply (no restart) |
| Tool policies | Hot-apply |
| Memory settings | Hot-apply |
| Channel credentials | Auto-restart |
| Gateway port | Auto-restart |
| New agents | Auto-restart |

**Trigger reload:**
```bash
# Send SIGHUP
kill -HUP $(pgrep -f "openclaw gateway")

# Or use CLI
openclaw gateway restart
```

## Environment Variables

Secrets should use environment variables:

```bash
# LLM providers
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
export GEMINI_API_KEY="..."

# Channels
export TELEGRAM_BOT_TOKEN="123:abc..."

# Embeddings
export VOYAGE_API_KEY="..."
```

Reference in config (optional — auto-resolved by default):
```json5
{
  models: {
    providers: {
      anthropic: {
        apiKey: { env: "ANTHROPIC_API_KEY" },
      },
    },
  },
}
```

## Validation

Always validate your config:

```bash
# Check for issues
openclaw doctor

# Auto-fix common problems
openclaw doctor --fix

# Verbose validation output
openclaw doctor --verbose
```

**Common validation errors:**
- Invalid JSON5 syntax
- Missing required fields
- Unknown provider names
- Invalid enum values

## Complete Example

```json5
// ~/.openclaw/openclaw.json
// Full working configuration for Telegram + memory search
{
  gateway: {
    port: 18789,
  },

  models: {
    default: "anthropic/claude-sonnet-4-20250514",
    fallbacks: ["openai/gpt-4o"],
  },

  // CRITICAL: Isolate sessions for multi-user safety
  session: {
    dmScope: "per-channel-peer",
  },

  channels: {
    telegram: {
      enabled: true,
      // Token from environment: TELEGRAM_BOT_TOKEN
      dmPolicy: "pairing",
      groups: {
        "*": { requireMention: true },
      },
    },
  },

  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      memorySearch: {
        enabled: true,
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

  tools: {
    profile: "coding",
  },
}
```

## Next Steps

With configuration understood, let's [set up Telegram](./03-telegram-setup.md) →
