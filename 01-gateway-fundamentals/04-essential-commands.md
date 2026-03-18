# Essential Commands

Command-line reference for managing your OpenClaw gateway.

## Gateway Management

### Starting and Stopping

```bash
# Start in foreground (Ctrl+C to stop)
openclaw gateway

# Start in background (daemon mode)
openclaw gateway --daemon

# Start with specific config
openclaw gateway --config /path/to/openclaw.json

# Stop daemon
openclaw gateway stop

# Restart (hot reload safe changes, restart for others)
openclaw gateway restart
```

### Status and Health

```bash
# Check gateway status
openclaw status

# Output:
# Gateway: running (pid 12345)
# Uptime: 2h 15m
# Channels: telegram (connected), discord (connected)
# Sessions: 12 active

# Detailed status
openclaw status --verbose

# Health check (for monitoring)
curl http://localhost:18789/health
```

### Logs

```bash
# View recent logs
openclaw logs

# Follow logs in real-time
openclaw logs --follow

# Filter by level
openclaw logs --level error

# Show more lines
openclaw logs --lines 100

# Filter by component
openclaw logs --filter telegram
```

## Configuration

### Validation

```bash
# Check for issues
openclaw doctor

# Auto-fix common problems
openclaw doctor --fix

# Verbose output
openclaw doctor --verbose

# Check specific config file
openclaw doctor --config /path/to/openclaw.json
```

### Interactive Setup

```bash
# Full interactive setup
openclaw init

# Configure specific section
openclaw configure --section models
openclaw configure --section telegram
openclaw configure --section web
```

## Channel Management

### Pairing (Telegram, WhatsApp)

```bash
# List pending pairing requests
openclaw pairing list telegram

# Approve a request
openclaw pairing approve telegram ABC123

# Reject a request
openclaw pairing reject telegram ABC123

# List approved users
openclaw pairing approved telegram
```

### Channel Status

```bash
# Check all channels
openclaw channels status

# Check specific channel with probe
openclaw channels status --channel telegram --probe

# List configured channels
openclaw channels list
```

### Sending Messages

```bash
# Send to Telegram user (by ID)
openclaw message send --channel telegram --target 123456789 --message "Hello!"

# Send to Telegram user (by username)
openclaw message send --channel telegram --target @username --message "Hello!"

# Send to group
openclaw message send --channel telegram --target -1001234567890 --message "Hello group!"

# Send with media
openclaw message send --channel telegram --target 123456789 \
  --message "Check this out" \
  --media /path/to/image.png

# Create a poll
openclaw message poll --channel telegram --target 123456789 \
  --poll-question "Ship it?" \
  --poll-option "Yes" \
  --poll-option "No"
```

## Session Management

```bash
# List active sessions
openclaw sessions list

# Show session history
openclaw sessions history <session-key>

# Clear a session (reset conversation)
openclaw sessions clear <session-key>

# Export session transcript
openclaw sessions export <session-key> --output transcript.json
```

## Agent Management

```bash
# List configured agents
openclaw agents list

# Show agent details
openclaw agents info main

# Show agent workspace
openclaw agents workspace main
```

## Memory Operations

```bash
# Search memory
openclaw memory search "project deadline"

# Show memory status
openclaw memory status

# Reindex memory (after manual file changes)
openclaw memory reindex
```

## Cron Jobs

```bash
# List scheduled jobs
openclaw cron list

# Add a job
openclaw cron add --schedule "0 9 * * *" --task "Morning check-in"

# Remove a job
openclaw cron remove <job-id>

# Run a job immediately
openclaw cron run <job-id>

# Show job history
openclaw cron runs <job-id>
```

## Debugging

### Verbose Output

```bash
# Most commands support --verbose
openclaw status --verbose
openclaw doctor --verbose
openclaw channels status --verbose
```

### Debug Mode

```bash
# Start gateway with debug logging
DEBUG=openclaw:* openclaw gateway

# Debug specific component
DEBUG=openclaw:telegram openclaw gateway
DEBUG=openclaw:memory openclaw gateway
```

### WebSocket Testing

```bash
# Connect to gateway WebSocket
websocat ws://localhost:18789/ws

# Send a test message (JSON format)
{"type": "ping"}
```

## Environment Variables

```bash
# Core
OPENCLAW_CONFIG=/path/to/openclaw.json
OPENCLAW_STATE_DIR=/path/to/state  # Default: ~/.openclaw

# LLM Providers
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=...
XAI_API_KEY=...

# Channels
TELEGRAM_BOT_TOKEN=123:abc...
DISCORD_BOT_TOKEN=...

# Embeddings
VOYAGE_API_KEY=...
MISTRAL_API_KEY=...

# Debug
DEBUG=openclaw:*
LOG_LEVEL=debug
```

## Common Workflows

### Daily Operations

```bash
# Morning: Check status
openclaw status
openclaw logs --lines 20

# Monitor throughout day
openclaw logs --follow

# Evening: Check for issues
openclaw doctor
```

### Troubleshooting

```bash
# 1. Check gateway is running
openclaw status

# 2. Check for config issues
openclaw doctor --verbose

# 3. Check channel connectivity
openclaw channels status --probe

# 4. Review recent errors
openclaw logs --level error --lines 50

# 5. Restart if needed
openclaw gateway restart
```

### Adding a New User (Telegram)

```bash
# 1. User sends message to bot

# 2. Check pairing requests
openclaw pairing list telegram

# 3. Approve
openclaw pairing approve telegram ABC123

# 4. Verify
openclaw pairing approved telegram
```

### Configuration Change

```bash
# 1. Edit config
nano ~/.openclaw/openclaw.json

# 2. Validate
openclaw doctor

# 3. Apply (hot reload or restart)
openclaw gateway restart
```

## Quick Reference Card

| Task | Command |
|------|---------|
| Start gateway | `openclaw gateway` |
| Check status | `openclaw status` |
| View logs | `openclaw logs --follow` |
| Validate config | `openclaw doctor` |
| Approve user | `openclaw pairing approve telegram CODE` |
| Send message | `openclaw message send --channel telegram --target ID --message "text"` |
| List sessions | `openclaw sessions list` |
| Search memory | `openclaw memory search "query"` |
| Restart | `openclaw gateway restart` |

## Next Steps

You've completed Gateway Fundamentals! Next, learn about [Session Management](../02-session-management/) — critical for multi-user deployments →
