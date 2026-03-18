# Multi-User Transition

A step-by-step guide for transitioning your OpenClaw deployment from single-user to multi-user safely.

## Your Current Situation

You have:
- A running bot that YOU use (single user)
- A public-facing social account (blog, Telegram bot username)
- You want to open it up to multiple users

The risk: Without proper configuration, new users will see YOUR conversation history.

## Pre-Flight Checklist

Before making changes, verify your current state:

```bash
# 1. Check current dmScope
grep -i "dmScope" ~/.openclaw/openclaw.json
# If missing or "main", you need to migrate

# 2. Check current sessions
openclaw sessions list
# Note: "main" session contains your history

# 3. Backup current config
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup

# 4. Backup session data
cp -r ~/.openclaw/agents/main/sessions ~/.openclaw/agents/main/sessions.backup
```

## Migration Steps

### Step 1: Update Configuration

Edit `~/.openclaw/openclaw.json`:

```json5
{
  // CRITICAL: Add session isolation
  session: {
    dmScope: "per-channel-peer",
  },

  // Tighten access control during transition
  channels: {
    telegram: {
      enabled: true,
      dmPolicy: "pairing",  // Require approval for new users
      
      // Optional: Temporarily restrict to just you
      // allowFrom: ["YOUR_TELEGRAM_USER_ID"],
    },
  },
}
```

### Step 2: Validate Configuration

```bash
openclaw doctor

# Expected output:
# ✓ Configuration valid
# ✓ Session isolation: per-channel-peer
# ✓ Telegram: pairing mode
```

### Step 3: Restart Gateway

```bash
openclaw gateway restart
```

### Step 4: Test Your Own Session

Message your bot and verify:
1. Your conversation works
2. History may be fresh (expected — session key changed)

```bash
# Check new session was created
openclaw sessions list

# Should show something like:
# agent:main:telegram:dm:YOUR_USER_ID
```

### Step 5: Test Isolation (Critical!)

This step confirms users are actually isolated.

**Option A: Use a second Telegram account**
1. Message bot from your main account: "My secret is ALPHA"
2. Message bot from second account: "What secrets do you know?"
3. Second account should NOT see "ALPHA"

**Option B: Use the CLI to simulate**
```bash
# Create a test message as "fake user"
# (This is a thought experiment — verify in logs)
openclaw logs --follow

# In another terminal, message from your main account
# Look for session key in logs — should include your user ID
```

### Step 6: Migrate Your History (Optional)

If you want to preserve your conversation history:

```bash
# Your old session was likely "main" or similar
# Your new session key is: agent:main:telegram:dm:YOUR_USER_ID

# Option 1: Rename the session file
cd ~/.openclaw/agents/main/sessions

# Find your old session UUID
cat sessions.json | jq '."main"'
# Returns: { "uuid": "abc123..." }

# Update sessions.json to map new key to old UUID
# (Manual edit required — backup first!)

# Option 2: Start fresh (recommended)
# Just accept that your history resets
# Old history is still in sessions.backup if needed
```

### Step 7: Open to Users

Once isolation is verified:

```json5
{
  session: {
    dmScope: "per-channel-peer",
  },
  channels: {
    telegram: {
      enabled: true,
      dmPolicy: "pairing",  // Approve users one by one
      // Or for fully open:
      // dmPolicy: "open",
      // allowFrom: ["*"],
    },
  },
}
```

## Post-Migration Verification

### Daily Check (First Week)

```bash
# Check for unexpected session sharing
openclaw sessions list

# Each user should have their own session key:
# agent:main:telegram:dm:user1_id
# agent:main:telegram:dm:user2_id
# ...

# NOT a single "main" session
```

### Monitor Logs

```bash
openclaw logs --follow | grep -E "(session|Created|isolated)"

# Watch for:
# [session] Created: agent:main:telegram:dm:123456789
# [session] Loaded: agent:main:telegram:dm:987654321

# Red flag:
# [session] Loaded: main  ← This means isolation failed!
```

### Test Periodically

Have a friend message your bot and ask:
- "What's my name?" (should not know if first message)
- "What did the last person say?" (should say "I don't know" or similar)

## Rollback Plan

If something goes wrong:

```bash
# 1. Stop gateway
openclaw gateway stop

# 2. Restore config backup
cp ~/.openclaw/openclaw.json.backup ~/.openclaw/openclaw.json

# 3. Restore sessions (if needed)
rm -rf ~/.openclaw/agents/main/sessions
cp -r ~/.openclaw/agents/main/sessions.backup ~/.openclaw/agents/main/sessions

# 4. Restart
openclaw gateway
```

## Common Issues During Migration

### Issue: "My history is gone!"

**Cause:** Session key changed from `main` to `agent:main:telegram:dm:YOUR_ID`

**Solutions:**
1. Accept fresh start (recommended)
2. Manually remap session UUID in `sessions.json`
3. Export old transcript and summarize to memory files

### Issue: "New user sees old messages"

**Cause:** dmScope not applied, or cached session

**Fix:**
```bash
# Verify config
grep -i "dmScope" ~/.openclaw/openclaw.json
# Must show: "per-channel-peer" (or stricter)

# Restart gateway
openclaw gateway restart

# Clear problematic session if needed
openclaw sessions clear "main"
```

### Issue: "Gateway won't start after config change"

**Cause:** Invalid JSON5 syntax

**Fix:**
```bash
# Validate config
openclaw doctor --verbose

# Common issues:
# - Missing comma after dmScope line
# - Typo in dmScope value
# - Unclosed braces
```

## Migration Checklist Summary

- [ ] Backup config: `cp openclaw.json openclaw.json.backup`
- [ ] Backup sessions: `cp -r sessions sessions.backup`
- [ ] Add `session.dmScope: "per-channel-peer"`
- [ ] Set `dmPolicy: "pairing"` during transition
- [ ] Run `openclaw doctor`
- [ ] Restart gateway
- [ ] Test your own session works
- [ ] Test isolation with second account
- [ ] Verify session keys in logs
- [ ] Open access policy when ready
- [ ] Monitor for first week

## Timeline Recommendation

| Day | Action |
|-----|--------|
| Day 1 | Backup, update config, test yourself |
| Day 2-3 | Test with 1-2 trusted users |
| Day 4-7 | Monitor logs, verify isolation |
| Day 7+ | Open to broader audience |

## Next Steps

Now that you've migrated, learn about [Security Patterns](./04-security-patterns.md) for ongoing protection →
