# Plugin Basics

Plugins extend agent capabilities with external tools via MCP (Model Context Protocol).

## What is a Plugin?

A plugin is an MCP server that provides:
- **Tools** — New actions the agent can take
- **Resources** — Data the agent can access
- **Prompts** — Pre-defined prompt templates

Unlike skills (instructions in context), plugins are external services that execute code.

## MCP Overview

MCP (Model Context Protocol) is a standard for connecting AI agents to external tools:

```
┌─────────────────────────────────────────────────────────┐
│                      OpenClaw                            │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐               │
│  │ Agent 1 │   │ Agent 2 │   │ Agent 3 │               │
│  └────┬────┘   └────┬────┘   └────┬────┘               │
│       │             │             │                     │
│       └─────────────┼─────────────┘                     │
│                     │                                   │
│              ┌──────┴──────┐                            │
│              │ MCP Router  │                            │
│              └──────┬──────┘                            │
└─────────────────────┼───────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │ GitHub  │   │  Slack  │   │ Custom  │
   │ Server  │   │ Server  │   │ Server  │
   └─────────┘   └─────────┘   └─────────┘
```

## Installing MCP Servers

### From npm

```bash
# Install globally
npm install -g @modelcontextprotocol/server-github

# Or locally in your project
npm install @modelcontextprotocol/server-github
```

### From Source

```bash
# Clone and build
git clone https://github.com/example/mcp-server-custom
cd mcp-server-custom
npm install
npm run build
```

### Docker

```bash
# Pull pre-built image
docker pull ghcr.io/example/mcp-server-custom:latest
```

## Configuring Plugins

### Basic Configuration

```json5
// ~/.openclaw/openclaw.json
{
  mcp: {
    servers: {
      // Server name (used in logs)
      "github": {
        // Command to run the server
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-github"],
        
        // Environment variables
        env: {
          GITHUB_TOKEN: "${GITHUB_TOKEN}",
        },
      },
    },
  },
}
```

### Multiple Servers

```json5
{
  mcp: {
    servers: {
      "github": {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-github"],
        env: { GITHUB_TOKEN: "${GITHUB_TOKEN}" },
      },
      
      "slack": {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-slack"],
        env: { SLACK_TOKEN: "${SLACK_TOKEN}" },
      },
      
      "filesystem": {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"],
      },
    },
  },
}
```

### Docker-Based Server

```json5
{
  mcp: {
    servers: {
      "custom": {
        command: "docker",
        args: [
          "run", "--rm", "-i",
          "-e", "API_KEY=${API_KEY}",
          "ghcr.io/example/mcp-server-custom:latest"
        ],
      },
    },
  },
}
```

### Per-Agent Plugins

```json5
{
  agents: {
    main: {
      // Uses all configured MCP servers
    },
    
    coder: {
      // Only use specific servers
      mcp: {
        servers: ["github", "filesystem"],
      },
    },
    
    researcher: {
      // Different set of servers
      mcp: {
        servers: ["search", "web"],
      },
    },
  },
  
  mcp: {
    servers: {
      github: { /* ... */ },
      filesystem: { /* ... */ },
      search: { /* ... */ },
      web: { /* ... */ },
    },
  },
}
```

## Using Plugin Tools

Once configured, plugin tools appear alongside built-in tools:

```json5
// Agent can call GitHub tools
{
  "tool": "github_create_issue",
  "params": {
    "owner": "example",
    "repo": "myproject",
    "title": "Bug: Login fails",
    "body": "Steps to reproduce..."
  }
}

// Or Slack tools
{
  "tool": "slack_post_message",
  "params": {
    "channel": "#engineering",
    "text": "Deployment complete!"
  }
}
```

## Popular MCP Servers

### Official Servers

| Server | Purpose | Install |
|--------|---------|---------|
| `server-github` | GitHub API | `@modelcontextprotocol/server-github` |
| `server-slack` | Slack API | `@modelcontextprotocol/server-slack` |
| `server-filesystem` | File access | `@modelcontextprotocol/server-filesystem` |
| `server-postgres` | PostgreSQL | `@modelcontextprotocol/server-postgres` |
| `server-sqlite` | SQLite | `@modelcontextprotocol/server-sqlite` |

### Community Servers

| Server | Purpose | Source |
|--------|---------|--------|
| `mcp-server-fetch` | HTTP requests | Community |
| `mcp-server-puppeteer` | Browser automation | Community |
| `mcp-server-docker` | Docker management | Community |

### Finding Servers

```bash
# Search npm
npm search @modelcontextprotocol

# Check MCP registry
https://github.com/modelcontextprotocol/servers
```

## Creating Custom Plugins

### Basic Structure (TypeScript)

```typescript
// src/index.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server({
  name: "my-custom-server",
  version: "1.0.0",
}, {
  capabilities: {
    tools: {},
  },
});

// Define a tool
server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "my_tool",
    description: "Does something useful",
    inputSchema: {
      type: "object",
      properties: {
        input: { type: "string", description: "Input value" },
      },
      required: ["input"],
    },
  }],
}));

// Handle tool calls
server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "my_tool") {
    const input = request.params.arguments.input;
    const result = await doSomething(input);
    return { content: [{ type: "text", text: result }] };
  }
  throw new Error(`Unknown tool: ${request.params.name}`);
});

// Start server
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Package Structure

```
my-mcp-server/
├── package.json
├── tsconfig.json
├── src/
│   └── index.ts
└── dist/
    └── index.js
```

```json
// package.json
{
  "name": "mcp-server-custom",
  "version": "1.0.0",
  "bin": {
    "mcp-server-custom": "dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  }
}
```

### Testing Your Plugin

```bash
# Build
npm run build

# Test manually (stdio interface)
echo '{"jsonrpc":"2.0","method":"tools/list","id":1}' | node dist/index.js

# Configure in OpenClaw
{
  mcp: {
    servers: {
      "custom": {
        command: "node",
        args: ["/path/to/my-mcp-server/dist/index.js"],
      },
    },
  },
}
```

## Plugin Resources

MCP servers can also provide resources (read-only data):

### Configuration

```json5
{
  mcp: {
    servers: {
      "docs": {
        command: "npx",
        args: ["-y", "mcp-server-docs", "/path/to/docs"],
      },
    },
  },
}
```

### Usage

```json5
// Agent can read resources
{
  "tool": "mcp_read_resource",
  "params": {
    "server": "docs",
    "uri": "docs://api-reference/auth"
  }
}
```

## Plugin Prompts

MCP servers can provide prompt templates:

### Server Definition

```typescript
server.setRequestHandler("prompts/list", async () => ({
  prompts: [{
    name: "code_review",
    description: "Review code for issues",
    arguments: [{
      name: "file",
      description: "File to review",
      required: true,
    }],
  }],
}));

server.setRequestHandler("prompts/get", async (request) => {
  if (request.params.name === "code_review") {
    return {
      messages: [{
        role: "user",
        content: {
          type: "text",
          text: `Review this code for issues:\n\n${request.params.arguments.file}`,
        },
      }],
    };
  }
});
```

### Usage

```json5
// Agent can use prompt templates
{
  "tool": "mcp_get_prompt",
  "params": {
    "server": "review",
    "name": "code_review",
    "arguments": { "file": "src/auth.ts" }
  }
}
```

## Security Considerations

### Environment Variables

Never hardcode secrets:

```json5
// Bad
{
  mcp: {
    servers: {
      github: {
        env: { GITHUB_TOKEN: "ghp_xxxxx" },  // Exposed!
      },
    },
  },
}

// Good
{
  mcp: {
    servers: {
      github: {
        env: { GITHUB_TOKEN: "${GITHUB_TOKEN}" },  // From environment
      },
    },
  },
}
```

### Path Restrictions

Limit filesystem access:

```json5
{
  mcp: {
    servers: {
      filesystem: {
        command: "npx",
        args: [
          "-y", "@modelcontextprotocol/server-filesystem",
          "/allowed/path",  // Only this path is accessible
        ],
      },
    },
  },
}
```

### Sandboxing

Use Docker for untrusted plugins:

```json5
{
  mcp: {
    servers: {
      untrusted: {
        command: "docker",
        args: [
          "run", "--rm", "-i",
          "--network", "none",        // No network
          "--read-only",              // Read-only filesystem
          "--memory", "512m",         // Memory limit
          "mcp-server-untrusted"
        ],
      },
    },
  },
}
```

## Troubleshooting

### "Server failed to start"

```bash
# Test server manually
npx -y @modelcontextprotocol/server-github

# Check for missing env vars
echo $GITHUB_TOKEN

# Check logs
openclaw logs | grep mcp
```

### "Tool not found"

```bash
# List available tools
openclaw tools list

# Check server is configured
openclaw config get mcp.servers
```

### "Permission denied"

- Check API tokens have required scopes
- Verify file paths are accessible
- Check Docker permissions

### "Server timeout"

```json5
{
  mcp: {
    servers: {
      slow: {
        command: "...",
        // Increase timeout
        timeout: 30000,  // 30 seconds
      },
    },
  },
}
```

## Best Practices

### Keep Servers Focused

One server, one integration:

```json5
// Good: Separate servers
{
  mcp: {
    servers: {
      github: { /* GitHub only */ },
      jira: { /* Jira only */ },
    },
  },
}

// Avoid: One server doing everything
{
  mcp: {
    servers: {
      everything: { /* GitHub + Jira + Slack + ... */ },
    },
  },
}
```

### Document Tool Usage

```markdown
<!-- In agent instructions -->

## Available MCP Tools

### GitHub
- `github_create_issue`: Create new issues
- `github_list_prs`: List pull requests
- `github_merge_pr`: Merge a pull request

Use these for GitHub operations instead of `gh` CLI.
```

### Handle Failures Gracefully

```markdown
## MCP Tool Errors

If an MCP tool fails:
1. Check the error message
2. Try alternative approach (CLI, manual)
3. Report to user if critical
```

## Next Steps

Now that you understand both skills and plugins, learn [When to Use Which](./03-when-to-use-which.md) →
