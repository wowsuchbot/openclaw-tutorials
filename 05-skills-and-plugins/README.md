# Module 5: Skills and Plugins

Extend agent capabilities with reusable components: skills for in-context instructions, plugins for external tools.

## What You'll Learn

1. [Skills Overview](./01-skills-overview.md) — In-context instruction modules
2. [Plugin Basics](./02-plugin-basics.md) — External tool integrations
3. [When to Use Which](./03-when-to-use-which.md) — Decision framework
4. [Coach as Example](./04-coach-as-example.md) — Complete skill case study

## Time Required

~45 minutes for all sections

## Skills vs Plugins

Both extend agent capabilities, but differently:

| Aspect | Skills | Plugins |
|--------|--------|---------|
| What | Markdown instructions | External code/services |
| Loaded | Into context at runtime | As callable tools |
| Language | Natural language | Code (JS, Python, etc.) |
| Execution | Agent interprets | Direct execution |
| Location | Workspace files | MCP servers, APIs |

### Skills

In-context instruction modules. Think "runbooks" or "playbooks":

```
skills/
└── deployment/
    └── SKILL.md        # Instructions for deployment workflow
```

```markdown
<!-- SKILL.md -->
# Deployment Skill

## Prerequisites
- All tests passing
- Branch up to date with main

## Steps
1. Create release branch
2. Update version in package.json
3. Generate changelog
4. Create PR
5. Wait for approval
6. Merge and tag
```

Agent reads SKILL.md and follows the instructions using its existing tools.

### Plugins

External services that provide additional tools via MCP (Model Context Protocol):

```json5
// openclaw.json
{
  mcp: {
    servers: {
      "github": {
        command: "mcp-server-github",
        args: ["--token", "${GITHUB_TOKEN}"],
      },
    },
  },
}
```

Agent gains new tools: `github_create_issue`, `github_list_prs`, etc.

## Quick Comparison

### Task: "Create a GitHub issue"

**With Skill (no plugin):**
```markdown
<!-- skills/github/SKILL.md -->
# GitHub Skill

## Create Issue
Use bash with gh CLI:
```bash
gh issue create --title "..." --body "..."
```
```

Agent reads skill, runs `bash` with `gh` command.

**With Plugin:**
```json5
// Direct tool call
{
  "tool": "github_create_issue",
  "params": {
    "title": "...",
    "body": "..."
  }
}
```

Agent calls plugin tool directly.

### Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| Skill | No external deps, flexible | Uses context, may misinterpret |
| Plugin | Precise, fast, no context | Requires setup, less flexible |

## When to Use What

```
Need to extend agent capabilities?
│
├─ Does the task need external integration?
│  ├─ Yes → Plugin (MCP server)
│  └─ No → Probably a skill
│
├─ Is there complex logic involved?
│  ├─ Yes → Plugin (code handles logic)
│  └─ No → Skill works fine
│
├─ Do you need structured I/O?
│  ├─ Yes → Plugin (defined schemas)
│  └─ No → Skill (natural language)
│
└─ Will multiple agents use this?
   ├─ Yes → Consider either, but plugins are more consistent
   └─ No → Skill is simpler
```

## Module Contents

### [Skills Overview](./01-skills-overview.md)
- Skill file structure
- How skills are loaded
- Writing effective skill instructions
- Skill libraries and organization

### [Plugin Basics](./02-plugin-basics.md)
- MCP protocol overview
- Installing MCP servers
- Configuring plugins
- Available plugin ecosystem

### [When to Use Which](./03-when-to-use-which.md)
- Decision framework
- Hybrid approaches
- Migration patterns
- Performance considerations

### [Coach as Example](./04-coach-as-example.md)
- Complete skill implementation
- SQLite database integration
- CLI tool design
- Cron-based scheduling
- GTD-style curation frontend

## Next Steps

Start with [Skills Overview](./01-skills-overview.md) to learn how to create in-context instruction modules →
