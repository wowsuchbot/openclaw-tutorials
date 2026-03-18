# Skills Overview

Skills are in-context instruction modules that teach agents how to perform specific tasks.

## What is a Skill?

A skill is a markdown file (`SKILL.md`) containing:
- Instructions for a specific task or domain
- Example commands and workflows
- Best practices and guidelines

When an agent needs to perform a task, it loads the relevant skill into context and follows the instructions.

## Skill Structure

### Basic Skill

```
skills/
└── deployment/
    └── SKILL.md
```

```markdown
<!-- skills/deployment/SKILL.md -->
# Deployment Skill

## Overview
This skill covers deploying applications to staging and production.

## Prerequisites
- Docker installed
- Access to container registry
- Kubernetes credentials configured

## Deploy to Staging

1. Build the Docker image:
```bash
docker build -t app:$(git rev-parse --short HEAD) .
```

2. Push to registry:
```bash
docker push registry.example.com/app:$(git rev-parse --short HEAD)
```

3. Update staging deployment:
```bash
kubectl set image deployment/app app=registry.example.com/app:$(git rev-parse --short HEAD) -n staging
```

4. Verify deployment:
```bash
kubectl rollout status deployment/app -n staging
```

## Deploy to Production

**Requires approval before proceeding.**

[Similar steps with production namespace]

## Rollback

If issues arise:
```bash
kubectl rollout undo deployment/app -n <namespace>
```
```

### Skill with Supporting Files

```
skills/
└── data-migration/
    ├── SKILL.md           # Main instructions
    ├── templates/
    │   ├── migration.sql  # SQL template
    │   └── verify.sql     # Verification queries
    └── examples/
        └── user-migration.md  # Example walkthrough
```

## How Skills Are Loaded

### Automatic Loading

Skills in the workspace `skills/` directory are available automatically:

```
~/.openclaw/workspace/
└── skills/
    ├── deployment/SKILL.md    # Available as "deployment"
    ├── review/SKILL.md        # Available as "review"
    └── testing/SKILL.md       # Available as "testing"
```

### Manual Loading

Agent can explicitly load a skill:

```markdown
## Agent Instructions

When asked to deploy:
1. Load the deployment skill: Read skills/deployment/SKILL.md
2. Follow the instructions there
```

### On-Demand Loading

Some skills are loaded only when relevant keywords are detected:

```json5
// In agent configuration
{
  skills: {
    deployment: {
      path: "skills/deployment/SKILL.md",
      keywords: ["deploy", "release", "ship"],
    },
  },
}
```

## Writing Effective Skills

### Clear Structure

```markdown
# Skill Name

## Overview
[1-2 sentences on what this skill does]

## When to Use
[Trigger conditions]

## Prerequisites
[What must be true before starting]

## Steps
[Numbered, actionable steps]

## Examples
[Concrete examples]

## Troubleshooting
[Common issues and fixes]
```

### Actionable Instructions

**Bad:**
```markdown
## Deployment
Deploy the application to the server.
```

**Good:**
```markdown
## Deploy to Staging

1. Verify tests pass:
```bash
npm test
```

2. Build production bundle:
```bash
npm run build
```

3. Deploy to staging:
```bash
./deploy.sh staging
```

4. Verify deployment:
```bash
curl https://staging.example.com/health
```
Expected: `{"status": "ok"}`
```

### Include Error Handling

```markdown
## Troubleshooting

### "Connection refused" error
The deploy server may be down. Check:
```bash
ssh deploy-server "systemctl status deploy-service"
```

### Build fails with memory error
Increase Node memory:
```bash
NODE_OPTIONS="--max-old-space-size=4096" npm run build
```
```

### Provide Context

```markdown
## Background

This deployment system uses blue-green deployment:
- "Blue" is the current production
- "Green" is the new version
- Traffic switches atomically after verification

Understanding this helps when troubleshooting rollbacks.
```

## Skill Organization

### By Domain

```
skills/
├── deployment/          # Infrastructure
├── database/            # Database operations
├── testing/             # Test workflows
├── security/            # Security procedures
└── support/             # Customer support
```

### By Complexity

```
skills/
├── quickstart/          # Simple, common tasks
│   ├── deploy-staging.md
│   └── run-tests.md
├── standard/            # Normal workflows
│   ├── release/SKILL.md
│   └── review/SKILL.md
└── advanced/            # Complex procedures
    ├── migration/SKILL.md
    └── disaster-recovery/SKILL.md
```

### By Agent

```
workspace-coder/
└── skills/
    ├── coding-standards/SKILL.md
    └── testing/SKILL.md

workspace-reviewer/
└── skills/
    └── code-review/SKILL.md

workspace-deployer/
└── skills/
    └── deployment/SKILL.md
```

## Skill Libraries

### Shared Skills

For skills used by multiple agents, use a shared location:

```
~/.openclaw/shared-skills/
├── git-workflow/SKILL.md
├── documentation/SKILL.md
└── communication/SKILL.md
```

Reference from agent workspaces:
```json5
{
  agents: {
    coder: {
      workspace: "~/.openclaw/workspace-coder",
      skills: {
        extraPaths: ["~/.openclaw/shared-skills"],
      },
    },
  },
}
```

### Community Skills

Skills can be shared and reused:

```bash
# Clone a skill library
git clone https://github.com/example/openclaw-skills ~/.openclaw/community-skills

# Reference in config
{
  skills: {
    extraPaths: ["~/.openclaw/community-skills"],
  },
}
```

## Skills with Code

### CLI Tool Integration

Skills often wrap CLI tools:

```markdown
# Coach Skill

## Check Priorities
```bash
python3 skills/coach/coach_db.py report priorities --limit 5
```

## Complete Task
```bash
python3 skills/coach/coach_db.py task complete <task_id>
```

## Create Task
```bash
python3 skills/coach/coach_db.py task create --title "..." --priority 1
```
```

### Script References

Skills can reference helper scripts:

```
skills/
└── release/
    ├── SKILL.md
    ├── bump-version.sh
    ├── generate-changelog.py
    └── tag-release.sh
```

```markdown
<!-- SKILL.md -->
# Release Skill

## Steps

1. Bump version:
```bash
./skills/release/bump-version.sh minor
```

2. Generate changelog:
```bash
python3 skills/release/generate-changelog.py
```

3. Create tag:
```bash
./skills/release/tag-release.sh
```
```

## Dynamic Skills

### Template Skills

Use placeholders for reusable patterns:

```markdown
# Deploy {SERVICE_NAME}

## Prerequisites
- {SERVICE_NAME} repository cloned
- Docker installed

## Steps

1. Navigate to service:
```bash
cd ~/repos/{SERVICE_NAME}
```

2. Build:
```bash
docker build -t {SERVICE_NAME}:latest .
```
```

### Conditional Instructions

```markdown
# Environment Setup

## Detect Environment

First, check your environment:
```bash
uname -s
```

## If macOS

```bash
brew install dependencies
```

## If Linux

```bash
apt-get install dependencies
```

## If Windows

Use WSL and follow Linux instructions.
```

## Skill Testing

### Manual Testing

```bash
# Ask agent to use skill
openclaw agent main --task "Deploy to staging using the deployment skill"

# Verify it followed instructions
```

### Checklist Testing

Create a test checklist:

```markdown
<!-- skills/deployment/TEST.md -->
# Deployment Skill Tests

## Test: Deploy to staging
- [ ] Agent reads SKILL.md
- [ ] Agent runs tests first
- [ ] Agent builds correctly
- [ ] Agent deploys to staging
- [ ] Agent verifies deployment

## Test: Rollback
- [ ] Agent identifies rollback need
- [ ] Agent executes rollback command
- [ ] Agent verifies rollback
```

## Best Practices

### Keep Skills Focused

One skill, one purpose:

**Bad:**
```
skills/everything/SKILL.md  # 2000 lines covering all operations
```

**Good:**
```
skills/deploy/SKILL.md      # Just deployment
skills/test/SKILL.md        # Just testing
skills/release/SKILL.md     # Just releases
```

### Version Your Skills

```markdown
<!-- SKILL.md -->
# Deployment Skill

**Version:** 2.1.0
**Last Updated:** 2024-01-15
**Maintainer:** DevOps team

## Changelog
- 2.1.0: Added blue-green deployment
- 2.0.0: Migrated to Kubernetes
- 1.0.0: Initial Docker deployment
```

### Include Escape Hatches

```markdown
## When to Deviate

These instructions cover the common case. Deviate when:
- Emergency hotfix needed (skip staging)
- Infrastructure changes required (coordinate with DevOps)
- Breaking changes (follow migration guide instead)

When deviating, document why in the deployment notes.
```

### Reference External Docs

```markdown
## Additional Resources

- [Kubernetes documentation](https://kubernetes.io/docs/)
- [Company deployment wiki](https://wiki.example.com/deployment)
- [On-call runbook](https://runbook.example.com)
```

## Next Steps

Skills handle instructions. For external tool integration, see [Plugin Basics](./02-plugin-basics.md) →
