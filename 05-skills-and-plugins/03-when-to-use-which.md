# When to Use Which

Decision framework for choosing between skills, plugins, and hybrid approaches.

## Quick Decision Matrix

| Factor | Skill | Plugin | Either |
|--------|-------|--------|--------|
| Setup complexity | Low | Medium-High | - |
| External integration | ❌ | ✅ | - |
| Structured I/O | Loose | Strict | - |
| Context usage | High | Low | - |
| Reliability | Variable | High | - |
| Flexibility | High | Medium | - |
| Speed | Depends | Fast | - |

## Decision Tree

```
Need to extend agent capabilities?
│
├─ Is there an existing CLI tool?
│  │
│  ├─ Yes, and it works well
│  │  └─ Use SKILL (wrap the CLI)
│  │
│  └─ No, or CLI is awkward
│     │
│     ├─ Is there an API?
│     │  │
│     │  ├─ Yes → Use PLUGIN (MCP server)
│     │  │
│     │  └─ No → Create SKILL with custom scripts
│     │
│     └─ Need structured data exchange?
│        │
│        ├─ Yes → PLUGIN (defined schemas)
│        │
│        └─ No → SKILL (flexible instructions)
│
├─ Will multiple agents use this?
│  │
│  ├─ Yes, with same behavior needed
│  │  └─ PLUGIN (consistent execution)
│  │
│  └─ Yes, but with variations
│     └─ SKILL (agents adapt instructions)
│
└─ Is reliability critical?
   │
   ├─ Yes, must work exactly
   │  └─ PLUGIN (code handles edge cases)
   │
   └─ No, approximate is fine
      └─ SKILL (agent interprets)
```

## When to Choose Skills

### Use Skills When:

**1. Wrapping existing CLI tools**
```markdown
<!-- skills/git/SKILL.md -->
# Git Workflow

## Create Feature Branch
```bash
git checkout main
git pull origin main
git checkout -b feature/$(echo "$FEATURE_NAME" | tr ' ' '-')
```
```

Agent interprets and executes using bash.

**2. Procedures with judgment calls**
```markdown
<!-- skills/code-review/SKILL.md -->
# Code Review

## Review Checklist

1. Check for obvious bugs
2. Look for security issues
3. Assess readability
4. **If unclear, ask for clarification**
5. **If minor issues, approve with suggestions**
6. **If major issues, request changes**
```

Agent applies judgment at each step.

**3. Domain knowledge sharing**
```markdown
<!-- skills/our-api/SKILL.md -->
# Our API Conventions

## Endpoint Naming
- Resources are plural: `/users`, `/posts`
- Actions use verbs: `/posts/123/publish`

## Error Handling
Always return:
```json
{ "error": { "code": "...", "message": "..." } }
```
```

**4. Evolving procedures**
```markdown
<!-- Easy to update -->
# Deployment v2.3

## Changes from v2.2
- Now using blue-green deployment
- Rollback is automatic on health check failure
```

### Skills Strengths

✅ Zero external dependencies
✅ Agent can adapt to variations
✅ Easy to write and update
✅ Human-readable documentation
✅ Works offline

### Skills Weaknesses

❌ Uses context window
❌ Agent may misinterpret
❌ Slower for complex operations
❌ No structured input validation
❌ Less consistent execution

## When to Choose Plugins

### Use Plugins When:

**1. External API integration**
```json5
// GitHub API with authentication
{
  mcp: {
    servers: {
      github: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-github"],
        env: { GITHUB_TOKEN: "${GITHUB_TOKEN}" },
      },
    },
  },
}
```

Plugin handles OAuth, rate limits, pagination.

**2. Structured data exchange**
```typescript
// Plugin defines exact schema
{
  name: "database_query",
  inputSchema: {
    type: "object",
    properties: {
      query: { type: "string" },
      params: { type: "array" },
    },
    required: ["query"],
  },
}
```

No ambiguity in input format.

**3. Complex logic**
```typescript
// Plugin handles complexity
server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "analyze_dependencies") {
    // Parse package.json
    // Build dependency graph
    // Check for vulnerabilities
    // Calculate update impact
    // Return structured report
  }
});
```

Logic is tested code, not instructions.

**4. Performance-critical operations**
```typescript
// Plugin executes directly
const results = await db.query(sql);  // Milliseconds

// vs. Skill via bash
// Agent constructs command → spawns process → parses output
```

### Plugin Strengths

✅ Consistent, reliable execution
✅ Structured input/output
✅ Doesn't use context
✅ Fast execution
✅ Handles complexity in code

### Plugin Weaknesses

❌ Requires development effort
❌ External dependencies
❌ Less flexible
❌ Harder to modify
❌ May need maintenance

## Hybrid Approaches

### Skill + Plugin Together

Use plugins for operations, skills for workflows:

```markdown
<!-- skills/release/SKILL.md -->
# Release Workflow

## Steps

1. Check CI status:
   Use `github_list_checks` tool for the release branch

2. If all checks pass, create release:
   Use `github_create_release` tool with:
   - tag: version from package.json
   - body: generated changelog

3. Post to Slack:
   Use `slack_post_message` tool to #releases channel

4. Update internal docs:
   Edit docs/releases.md with release notes
```

Skill orchestrates plugins.

### Plugin Calls from Skills

Reference plugin tools in skill instructions:

```markdown
## Using MCP Tools

When you need to:
- Create GitHub issues → Use `github_create_issue`
- Query database → Use `postgres_query`
- Send notifications → Use `slack_post_message`

These are more reliable than CLI equivalents.
```

### Fallback Patterns

```markdown
## Robust GitHub Operations

### Primary: MCP Plugin
Use `github_create_issue` when available.

### Fallback: CLI
If MCP fails, use:
```bash
gh issue create --title "..." --body "..."
```

### Last Resort: Manual
Provide instructions for the user to create manually.
```

## Migration Patterns

### Skill → Plugin

When a skill becomes critical enough:

**Before (Skill):**
```markdown
# Database Backup

```bash
pg_dump -h $DB_HOST -U $DB_USER $DB_NAME > backup.sql
aws s3 cp backup.sql s3://backups/$(date +%Y%m%d).sql
rm backup.sql
```
```

**After (Plugin):**
```typescript
// Handles errors, retries, logging
server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "database_backup") {
    const backup = await pgDump(config);
    await s3Upload(backup, getBucketPath());
    await cleanup(backup);
    return { content: [{ type: "text", text: "Backup complete" }] };
  }
});
```

### Plugin → Skill

When you need more flexibility:

**Before (Plugin):**
```typescript
// Rigid workflow
tools: [{
  name: "deploy",
  inputSchema: {
    properties: {
      environment: { enum: ["staging", "production"] },
    },
  },
}]
```

**After (Skill):**
```markdown
# Flexible Deployment

## Standard Deployment
Deploy to staging or production as usual.

## Hotfix Deployment
Skip staging, deploy directly to production with:
- Extra monitoring
- Immediate rollback ready

## Canary Deployment
Deploy to 10% of production first...
```

## Performance Considerations

### Context Usage

| Approach | Context Impact |
|----------|----------------|
| Small skill (< 500 tokens) | Minimal |
| Large skill (> 2000 tokens) | Significant |
| Plugin | None |

For agents with limited context, prefer plugins.

### Execution Speed

| Approach | Typical Latency |
|----------|-----------------|
| Plugin (direct) | 10-100ms |
| Skill + bash | 100-500ms |
| Skill + complex logic | 500ms-2s |

For high-frequency operations, prefer plugins.

### Reliability

| Approach | Failure Modes |
|----------|---------------|
| Plugin | Crashes, API errors (detectable) |
| Skill | Misinterpretation, wrong command (subtle) |

For critical operations, prefer plugins.

## Real-World Examples

### Example: GitHub Operations

**Use Plugin:**
- Creating issues (structured data)
- Merging PRs (critical operation)
- Managing releases (API-heavy)

**Use Skill:**
- Git workflow (wraps CLI)
- Code review process (judgment)
- Branch naming conventions (guidelines)

### Example: Database Operations

**Use Plugin:**
- Running queries (structured)
- Backups (critical)
- Migrations (complex logic)

**Use Skill:**
- Query optimization tips (knowledge)
- Schema design guidelines (judgment)
- Troubleshooting slow queries (procedure)

### Example: Communication

**Use Plugin:**
- Sending Slack messages (API)
- Creating tickets (structured)
- Email integration (external)

**Use Skill:**
- Message templates (flexible)
- Escalation procedures (judgment)
- Communication guidelines (knowledge)

## Summary

```
Choose SKILL when:
├─ Wrapping CLI tools
├─ Sharing knowledge/guidelines
├─ Needing flexibility
├─ Quick iteration needed
└─ Agent judgment required

Choose PLUGIN when:
├─ External API integration
├─ Structured data required
├─ Reliability critical
├─ Performance matters
└─ Complex logic involved

Choose BOTH when:
├─ Orchestrating plugin calls
├─ Providing fallbacks
└─ Combining judgment with reliability
```

## Next Steps

See a complete skill implementation in [Coach as Example](./04-coach-as-example.md) →
