# Telegram Setup

Connect OpenClaw to Telegram — your first channel integration.

## Overview

Telegram integration uses the **Bot API** via long polling (default) or webhooks. You'll need:

1. A Telegram bot token from @BotFather
2. Configuration in `openclaw.json`
3. Pairing approval for the first user

## Step 1: Create Your Bot

### Talk to @BotFather

1. Open Telegram and search for **@BotFather** (verify the checkmark!)
2. Send `/newbot`
3. Follow prompts:
   - Choose a display name (e.g., "My Assistant")
   - Choose a username ending in `bot` (e.g., `my_assistant_bot`)
4. Copy the **bot token** (looks like `123456789:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`)

### Recommended BotFather Settings

```
/setprivacy → Disable (to see all group messages without @mention)
/setjoingroups → Enable (to allow adding to groups)
/setcommands → Set your custom commands
```

## Step 2: Configure OpenClaw

### Minimal Configuration

```json5
// ~/.openclaw/openclaw.json
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123456789:ABC-DEF...",  // Your token here
      dmPolicy: "pairing",                // Require approval for new users
    },
  },
}
```

### Using Environment Variable (Recommended)

```bash
# Set in your shell profile (~/.bashrc, ~/.zshrc)
export TELEGRAM_BOT_TOKEN="123456789:ABC-DEF..."
```

```json5
// openclaw.json — no token needed, auto-resolved from env
{
  channels: {
    telegram: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

## Step 3: Start Gateway and Pair

### Start the Gateway

```bash
openclaw gateway
```

Watch for:
```
[telegram] Bot started: @my_assistant_bot
[telegram] Long polling active
```

### Send Your First Message

1. Open Telegram
2. Search for your bot by username
3. Send any message (e.g., "Hello")

### Approve the Pairing Request

```bash
# List pending pairing requests
openclaw pairing list telegram

# Output:
# CODE      USER            EXPIRES
# ABC123    @yourname       in 58 minutes

# Approve it
openclaw pairing approve telegram ABC123
```

Now your bot will respond!

## Access Control Options

### Option 1: Pairing (Default, Recommended)

New users must be approved via pairing code:

```json5
{
  channels: {
    telegram: {
      dmPolicy: "pairing",
    },
  },
}
```

**Pros:** Secure, audit trail
**Cons:** Manual approval step

### Option 2: Allowlist

Only specific user IDs can message:

```json5
{
  channels: {
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: ["123456789", "987654321"],  // Numeric Telegram user IDs
    },
  },
}
```

**Finding your Telegram user ID:**
```bash
# Method 1: Check gateway logs after messaging
openclaw logs --follow
# Look for: from.id: 123456789

# Method 2: Use Bot API directly
curl "https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates"
```

### Option 3: Open (Use with Caution!)

Anyone can message:

```json5
{
  channels: {
    telegram: {
      dmPolicy: "open",
      allowFrom: ["*"],  // Required for open mode
    },
  },
}
```

**Warning:** Only use this for public bots with proper session isolation!

## Group Chat Configuration

### Basic Group Setup

```json5
{
  channels: {
    telegram: {
      // ... DM settings ...
      
      // Group settings
      groupPolicy: "allowlist",  // "open" | "allowlist" | "disabled"
      groups: {
        "*": {                    // Default for all groups
          requireMention: true,   // Must @mention the bot
        },
      },
    },
  },
}
```

### Per-Group Configuration

```json5
{
  channels: {
    telegram: {
      groups: {
        // Default behavior
        "*": { requireMention: true },
        
        // Specific group (use negative chat ID)
        "-1001234567890": {
          requireMention: false,    // No mention needed
          groupPolicy: "open",      // Anyone in group can trigger
        },
        
        // Another group with allowlist
        "-1009876543210": {
          requireMention: true,
          allowFrom: ["123456789"], // Only this user can trigger
        },
      },
    },
  },
}
```

**Getting group chat ID:**
```bash
# Add bot to group, send a message, check logs
openclaw logs --follow
# Look for: chat.id: -1001234567890
```

### Forum Topics (Supergroups)

For Telegram supergroups with forum topics enabled:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          topics: {
            "1": { agentId: "main" },     // General topic → main agent
            "3": { agentId: "coder" },    // Topic 3 → coder agent
            "5": { 
              agentId: "support",
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

## Streaming and Formatting

### Live Response Streaming

```json5
{
  channels: {
    telegram: {
      // "off" | "partial" | "block" | "progress"
      streaming: "partial",  // Preview message + edits (default)
    },
  },
}
```

### Message Formatting

```json5
{
  channels: {
    telegram: {
      textChunkLimit: 4000,           // Max chars per message
      chunkMode: "newline",           // Split on paragraphs
      linkPreview: true,              // Show link previews
    },
  },
}
```

## Troubleshooting

### Bot doesn't respond

1. **Check gateway is running:**
   ```bash
   openclaw status
   ```

2. **Check pairing status:**
   ```bash
   openclaw pairing list telegram
   ```

3. **Check logs for errors:**
   ```bash
   openclaw logs --follow
   ```

### Bot doesn't see group messages

1. **Privacy mode may be enabled:**
   - Go to @BotFather
   - Send `/setprivacy`
   - Select your bot
   - Choose `Disable`
   - **Remove and re-add bot to group**

2. **Group not in allowlist:**
   ```json5
   groups: {
     "-1001234567890": { requireMention: true },
     // or use "*" for all groups
   }
   ```

### Commands work partially

- Verify your user ID is authorized (pairing or allowlist)
- Check `openclaw logs` for authorization failures

### Network issues

For hosts with unstable connections:
```json5
{
  channels: {
    telegram: {
      proxy: "socks5://user:pass@proxy-host:1080",  // Optional proxy
      network: {
        autoSelectFamily: false,  // Force IPv4 on WSL2
      },
    },
  },
}
```

## Complete Telegram Configuration

```json5
// Full working Telegram setup
{
  channels: {
    telegram: {
      // Core settings
      enabled: true,
      // botToken from TELEGRAM_BOT_TOKEN env var
      
      // Access control
      dmPolicy: "pairing",
      
      // Group behavior
      groupPolicy: "allowlist",
      groups: {
        "*": { 
          requireMention: true,
        },
        // Specific trusted group
        "-1001234567890": {
          requireMention: false,
          groupPolicy: "open",
        },
      },
      
      // Response behavior
      streaming: "partial",
      textChunkLimit: 4000,
      chunkMode: "newline",
      linkPreview: true,
      
      // Reliability
      retry: {
        attempts: 3,
        minDelayMs: 1000,
        maxDelayMs: 10000,
      },
    },
  },
}
```

## Next Steps

Learn the [essential CLI commands](./04-essential-commands.md) for managing your gateway →
