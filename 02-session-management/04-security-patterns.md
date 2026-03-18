# Security Patterns

Best practices for maintaining security in a multi-user OpenClaw deployment.

## Security Principles

1. **Isolation by Default** — Users cannot see each other's data
2. **Least Privilege** — Agents have minimal necessary permissions
3. **Explicit Authorization** — Access requires approval, not just absence of denial
4. **Audit Trail** — Actions are logged for review

## Context Leak Prevention

### Session Isolation (Critical)

Already covered, but worth repeating:

```json5
{
  session: {
    dmScope: "per-channel-peer",  // REQUIRED
  },
}
```

### Memory Isolation

Memory files can contain sensitive information. Control access:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        // Only search in main/private sessions
        scope: {
          default: "deny",
          rules: [
            { action: "allow", match: { chatType: "direct" } },
          ],
        },
      },
    },
  },
}
```

### Tool Visibility

Limit what sessions an agent can see:

```json5
{
  tools: {
    sessions: {
      // "self" — only current session
      // "tree" — current + spawned sub-agents (default)
      // "agent" — all sessions for this agent
      // "all" — all sessions (dangerous!)
      visibility: "tree",
    },
  },
}
```

**In multi-user deployments:**
```json5
{
  tools: {
    sessions: {
      visibility: "self",  // Most restrictive
    },
  },
}
```

## Access Control Patterns

### Pattern 1: Pairing + Allowlist Hybrid

Start with pairing, then lock down:

```json5
{
  channels: {
    telegram: {
      // New users go through pairing
      dmPolicy: "pairing",
      
      // After approval, add to permanent allowlist
      // (optional, for faster reconnection)
      allowFrom: [
        "123456789",  // Approved user 1
        "987654321",  // Approved user 2
      ],
    },
  },
}
```

### Pattern 2: Group Sender Filtering

Control who can trigger the bot in groups:

```json5
{
  channels: {
    telegram: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["123456789"],  // Only this user can trigger in groups
      
      groups: {
        "-1001234567890": {
          // Per-group override
          allowFrom: ["123456789", "555555555"],
        },
      },
    },
  },
}
```

### Pattern 3: Topic-Based Isolation

Route different forum topics to different agents:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          topics: {
            "1": { agentId: "main" },     // General
            "3": { agentId: "support" },  // Support requests
            "5": { agentId: "admin" },    // Admin only
          },
        },
      },
    },
  },
}
```

## Tool Access Control

### Profile-Based Restrictions

```json5
{
  tools: {
    // Base profile
    profile: "coding",  // fs, runtime, sessions, memory
    
    // Explicit denials
    deny: [
      "browser",           // No browser automation
      "exec",              // No shell commands (use bash instead)
      "group:automation",  // No cron/gateway tools
    ],
  },
}
```

### Per-Agent Restrictions

```json5
{
  agents: {
    list: [
      {
        id: "support",
        tools: {
          profile: "messaging",  // Only messaging tools
          deny: ["group:fs"],    // No file access
        },
      },
      {
        id: "coder",
        tools: {
          profile: "coding",
          allow: ["browser"],    // This agent can use browser
        },
      },
    ],
  },
}
```

### Sandbox Mode

Run agents in restricted environments:

```json5
{
  agents: {
    list: [
      {
        id: "untrusted",
        sandbox: {
          enabled: true,
          workspaceAccess: "rw",  // "rw" | "ro" | "none"
          network: "restricted",  // "full" | "restricted" | "none"
          tools: {
            deny: ["exec", "process"],
          },
        },
      },
    ],
  },
}
```

## Exec Approvals

Require human approval for dangerous commands:

```json5
{
  tools: {
    exec: {
      approvals: {
        enabled: true,
        policy: "allowlist",  // "allowlist" | "always" | "never"
        allowlist: [
          "git status",
          "git diff",
          "npm test",
        ],
      },
    },
  },
  
  channels: {
    telegram: {
      execApprovals: {
        enabled: true,
        approvers: ["123456789"],  // Only these users can approve
        target: "dm",              // Send approval requests to DM
      },
    },
  },
}
```

## Memory Security

### Private Memory Files

`MEMORY.md` should only load in private sessions:

```json5
// This is default behavior, but worth understanding:
// MEMORY.md is loaded only in "main session" (direct chat with owner)
// It is NOT loaded in:
// - Group chats
// - Shared sessions
// - Sub-agent runs (unless explicitly passed)
```

### Memory Scope Control

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        scope: {
          default: "deny",
          rules: [
            // Allow in DMs only
            { action: "allow", match: { chatType: "direct" } },
            // Deny in specific channels
            { action: "deny", match: { keyPrefix: "discord:channel:" } },
          ],
        },
      },
    },
  },
}
```

## Audit Logging

### Enable Detailed Logs

```json5
{
  logging: {
    level: "info",  // "debug" for more detail
    
    // Log security-relevant events
    events: {
      sessionCreate: true,
      sessionAccess: true,
      toolExecution: true,
      approvalRequests: true,
    },
  },
}
```

### Monitor Logs

```bash
# Watch for security events
openclaw logs --follow | grep -E "(auth|session|approval|denied)"

# Export logs for review
openclaw logs --since "24h" --output audit.log
```

### What to Look For

```bash
# Suspicious patterns:
# - Same session accessed by different IPs
# - Repeated auth failures
# - Unusual tool usage patterns
# - Cross-session tool calls

grep -E "(denied|unauthorized|failed)" ~/.openclaw/logs/*.log
```

## Security Checklist

### Initial Setup

- [ ] `session.dmScope` set to `per-channel-peer` or stricter
- [ ] `dmPolicy` set to `pairing` or `allowlist`
- [ ] `groupPolicy` set to `allowlist` (not `open`)
- [ ] Tool profile appropriate for use case
- [ ] Exec approvals enabled for production

### Ongoing

- [ ] Review pairing requests before approving
- [ ] Monitor logs weekly for anomalies
- [ ] Update allowlists when users leave
- [ ] Rotate API keys periodically
- [ ] Test isolation after config changes

### Before Opening to Public

- [ ] Test with multiple accounts
- [ ] Verify session isolation in logs
- [ ] Confirm memory scope restrictions
- [ ] Enable exec approvals
- [ ] Set up alerting for failures

## Common Security Mistakes

### Mistake 1: Open Group Policy

```json5
// ❌ DANGEROUS
{
  channels: {
    telegram: {
      groupPolicy: "open",
      groups: { "*": { requireMention: false } },
    },
  },
}

// ✅ SAFER
{
  channels: {
    telegram: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["trusted_user_id"],
      groups: {
        "-specific_group": { requireMention: true },
      },
    },
  },
}
```

### Mistake 2: Visibility: "all"

```json5
// ❌ DANGEROUS — any user can browse all sessions
{
  tools: {
    sessions: { visibility: "all" },
  },
}

// ✅ SAFER
{
  tools: {
    sessions: { visibility: "self" },
  },
}
```

### Mistake 3: Unrestricted Exec

```json5
// ❌ DANGEROUS — agent can run any command
{
  tools: {
    profile: "full",
  },
}

// ✅ SAFER
{
  tools: {
    profile: "coding",
    exec: {
      approvals: { enabled: true, policy: "allowlist" },
    },
  },
}
```

## Emergency Procedures

### Suspected Data Leak

```bash
# 1. Stop gateway immediately
openclaw gateway stop

# 2. Check recent sessions
openclaw sessions list

# 3. Review logs for cross-session access
grep "session" ~/.openclaw/logs/*.log | tail -100

# 4. Identify affected sessions
# 5. Notify affected users if necessary
# 6. Fix configuration
# 7. Restart with tightened security
```

### Compromised API Key

```bash
# 1. Rotate key at provider (OpenAI, Anthropic, etc.)
# 2. Update environment variable
export ANTHROPIC_API_KEY="new-key"

# 3. Restart gateway
openclaw gateway restart

# 4. Review logs for unauthorized usage
```

## Summary

| Area | Recommended Setting |
|------|---------------------|
| Session isolation | `dmScope: "per-channel-peer"` |
| DM access | `dmPolicy: "pairing"` |
| Group access | `groupPolicy: "allowlist"` |
| Session visibility | `visibility: "self"` or `"tree"` |
| Tool profile | `"coding"` or `"messaging"` (not `"full"`) |
| Exec approvals | `enabled: true` |

## Next Steps

You've completed Session Management! Continue to [Memory Systems](../03-memory-systems/) to learn about building effective long-term memory →
