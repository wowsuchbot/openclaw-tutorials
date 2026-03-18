# Installation

Get OpenClaw running on your system.

## System Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| Node.js | 20.x | 22.x |
| RAM | 512 MB | 2 GB+ |
| Disk | 500 MB | 2 GB+ |
| OS | Linux, macOS, WSL2 | Linux |

## Installation Methods

### Method 1: npm (Recommended)

```bash
# Install globally
npm install -g openclaw

# Verify installation
openclaw --version
```

### Method 2: From Source

```bash
git clone https://github.com/anomalyco/openclaw.git
cd openclaw
pnpm install
pnpm build

# Link globally
pnpm link --global
```

## First Run

### Initialize Configuration

```bash
openclaw init
```

This creates `~/.openclaw/openclaw.json` with sensible defaults.

**Interactive prompts:**
- LLM provider selection (OpenAI, Anthropic, Google, etc.)
- API key configuration
- Optional: channel setup (Telegram, Discord)

### Configure LLM Provider

At minimum, you need one LLM provider. Add to your config:

```json5
// ~/.openclaw/openclaw.json
{
  models: {
    default: "anthropic/claude-sonnet-4-20250514",
    providers: {
      anthropic: {
        apiKey: "sk-ant-...",  // or use environment variable
      },
    },
  },
}
```

**Using environment variables (recommended for secrets):**

```bash
# ~/.bashrc or ~/.zshrc
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
```

Then in config:
```json5
{
  models: {
    default: "anthropic/claude-sonnet-4-20250514",
    // No apiKey needed — resolved from environment
  },
}
```

### Start the Gateway

```bash
# Foreground (for development)
openclaw gateway

# Background (for production)
openclaw gateway --daemon

# With specific config file
openclaw gateway --config /path/to/openclaw.json
```

### Verify It's Running

```bash
# Check status
openclaw status

# View logs
openclaw logs --follow

# Check health endpoint
curl http://localhost:18789/health
```

## Directory Structure

After first run, OpenClaw creates:

```
~/.openclaw/
├── openclaw.json          # Main configuration
├── workspace/             # Default agent workspace
│   ├── MEMORY.md          # Long-term memory (created on first use)
│   └── memory/            # Daily logs directory
├── agents/                # Per-agent data
│   └── main/
│       ├── sessions/      # Conversation transcripts
│       └── agent/         # Agent-specific overrides
├── logs/                  # Gateway logs
├── credentials/           # OAuth tokens, channel auth
└── memory/                # Vector index storage
    └── main.sqlite        # Embeddings database
```

## Common Installation Issues

### Issue: `EACCES` permission denied

```bash
# Fix npm global permissions
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### Issue: Node version too old

```bash
# Using nvm (recommended)
nvm install 22
nvm use 22
```

### Issue: Gateway won't start

Check logs for the actual error:
```bash
openclaw logs --lines 50
```

Common causes:
- Port 18789 already in use → change `gateway.port`
- Missing API key → check `models.providers`
- Invalid JSON5 syntax → run `openclaw doctor`

## Configuration Validation

Always validate after editing config:

```bash
# Check for issues
openclaw doctor

# Auto-fix common problems
openclaw doctor --fix
```

## Next Steps

Now that OpenClaw is installed, learn about [Configuration Basics](./02-configuration-basics.md) →

---

## Full Installation Example

```bash
# 1. Install
npm install -g openclaw

# 2. Initialize
openclaw init

# 3. Set API key (choose one)
export ANTHROPIC_API_KEY="sk-ant-..."
# or
export OPENAI_API_KEY="sk-..."

# 4. Start gateway
openclaw gateway

# 5. Verify (in another terminal)
openclaw status
# Output: Gateway running on port 18789
```
