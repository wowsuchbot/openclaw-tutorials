# Vector Memory Setup

Configure embedding providers for fast semantic search across your memory files.

## Why Vector Search?

Markdown files are great, but they have limitations:
- **Exact match only** — Can't find "deadline" if you wrote "due date"
- **No ranking** — Which result is most relevant?
- **Linear scan** — Slow for large file sets

Vector search solves these:
- **Semantic matching** — Finds conceptually similar content
- **Similarity ranking** — Best matches first
- **Index acceleration** — Fast even with thousands of files

## Provider Options

| Provider | Quality | Cost | Privacy | Setup |
|----------|---------|------|---------|-------|
| **Gemini** | High | Low (~$0.001/1K) | Cloud | API key |
| **OpenAI** | High | Medium (~$0.002/1K) | Cloud | API key |
| **Voyage** | Very High | Higher | Cloud | API key |
| **Local (GGUF)** | Good | Free | Full | Model download |
| **Ollama** | Good | Free | Self-hosted | Ollama server |

## Quick Setup

### Gemini (Recommended for Most Users)

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "gemini",
        model: "gemini-embedding-001",  // or "gemini-embedding-2-preview"
        // API key from GEMINI_API_KEY env var
      },
    },
  },
}
```

```bash
# Set API key
export GEMINI_API_KEY="your-api-key"
```

### OpenAI

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "openai",
        model: "text-embedding-3-small",  // or "text-embedding-3-large"
      },
    },
  },
}
```

```bash
export OPENAI_API_KEY="sk-..."
```

### Local (No API, Full Privacy)

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "local",
        local: {
          // Auto-downloads from HuggingFace
          modelPath: "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf",
        },
      },
    },
  },
}
```

**Setup requirements:**
```bash
# Approve native build
pnpm approve-builds
# Select node-llama-cpp, then rebuild
pnpm rebuild node-llama-cpp
```

## Detailed Provider Configuration

### Gemini

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "gemini",
        
        // Model options:
        // - "gemini-embedding-001" (768 dims, 2048 token limit)
        // - "gemini-embedding-2-preview" (configurable dims, 8192 token limit)
        model: "gemini-embedding-001",
        
        // For embedding-2, configure dimensions
        // outputDimensionality: 3072,  // 768, 1536, or 3072
        
        remote: {
          // Explicit API key (optional if using env var)
          apiKey: "your-api-key",
          
          // Custom base URL (optional)
          baseUrl: null,
          
          // Extra headers (optional)
          headers: {},
        },
      },
    },
  },
}
```

**Gemini Embedding 2 (Preview):**
```json5
{
  memorySearch: {
    provider: "gemini",
    model: "gemini-embedding-2-preview",
    outputDimensionality: 3072,  // Higher = more precise, more storage
  },
}
```

### OpenAI

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        
        // Model options:
        // - "text-embedding-3-small" (1536 dims, lower cost)
        // - "text-embedding-3-large" (3072 dims, higher quality)
        // - "text-embedding-ada-002" (legacy)
        model: "text-embedding-3-small",
        
        remote: {
          apiKey: "sk-...",
          
          // Custom endpoint (for OpenAI-compatible APIs)
          baseUrl: "https://api.example.com/v1/",
          
          headers: {
            "X-Organization": "org-id",
          },
        },
      },
    },
  },
}
```

### Voyage

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "voyage",
        model: "voyage-2",  // or "voyage-large-2"
        remote: {
          apiKey: "your-voyage-key",
        },
      },
    },
  },
}
```

```bash
export VOYAGE_API_KEY="..."
```

### Mistral

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "mistral",
        model: "mistral-embed",
        remote: {
          apiKey: "your-mistral-key",
        },
      },
    },
  },
}
```

```bash
export MISTRAL_API_KEY="..."
```

### Ollama (Self-Hosted)

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "ollama",
        model: "nomic-embed-text",  // or your preferred model
        remote: {
          baseUrl: "http://localhost:11434",
          apiKey: "ollama-local",  // placeholder, often not required
        },
      },
    },
  },
}
```

**Setup:**
```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull embedding model
ollama pull nomic-embed-text

# Verify
ollama list
```

### Local (node-llama-cpp)

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "local",
        local: {
          // HuggingFace URI (auto-downloads)
          modelPath: "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf",
          
          // Or local file path
          // modelPath: "~/.openclaw/models/embedding.gguf",
          
          // Cache directory for downloaded models
          modelCacheDir: "~/.openclaw/models",
        },
        
        // Disable fallback to cloud
        fallback: "none",
      },
    },
  },
}
```

## Fallback Configuration

If your primary provider fails, OpenClaw can fall back:

```json5
{
  memorySearch: {
    provider: "local",
    fallback: "openai",  // Falls back if local fails
    
    // Options: "openai" | "gemini" | "voyage" | "mistral" | "ollama" | "local" | "none"
  },
}
```

**Common pattern:** Local primary, cloud fallback
```json5
{
  memorySearch: {
    provider: "local",
    fallback: "gemini",  // Use cloud if local fails
  },
}
```

## Index Storage

### Default Location

```
~/.openclaw/memory/<agentId>.sqlite

Example:
~/.openclaw/memory/main.sqlite
```

### Custom Path

```json5
{
  memorySearch: {
    store: {
      path: "/custom/path/{agentId}-memory.sqlite",
    },
  },
}
```

### sqlite-vec Acceleration

When available, OpenClaw uses sqlite-vec for fast vector queries:

```json5
{
  memorySearch: {
    store: {
      vector: {
        enabled: true,  // default
        extensionPath: "/custom/path/to/sqlite-vec",  // optional override
      },
    },
  },
}
```

**Check if sqlite-vec is working:**
```bash
openclaw memory status
# Should show: "Vector backend: sqlite-vec"
```

## Reindexing

### When Automatic Reindexing Occurs

- File content changes (1.5s debounce)
- Provider/model changes in config
- Chunking parameters change
- Embedding dimensions change

### Manual Reindex

```bash
# Full reindex
openclaw memory reindex

# Check index status
openclaw memory status
```

### Force Fresh Index

```bash
# Delete index and regenerate
rm ~/.openclaw/memory/main.sqlite
openclaw gateway restart
# Index rebuilds on first memory_search
```

## Batch Indexing (Large Datasets)

For initial indexing of large memory collections:

```json5
{
  memorySearch: {
    provider: "openai",  // or "gemini"
    remote: {
      batch: {
        enabled: true,
        concurrency: 2,
        wait: true,  // Wait for batch completion
      },
    },
  },
}
```

**Why batch?**
- Cheaper (OpenAI Batch API has discounted pricing)
- Faster for large datasets
- Better rate limit handling

## Embedding Cache

Cache embeddings to avoid re-computing unchanged content:

```json5
{
  memorySearch: {
    cache: {
      enabled: true,
      maxEntries: 50000,
    },
  },
}
```

## Multimodal Memory (Gemini Only)

Index images and audio alongside text:

```json5
{
  memorySearch: {
    provider: "gemini",
    model: "gemini-embedding-2-preview",
    
    // Additional paths with media files
    extraPaths: ["assets/reference", "voice-notes"],
    
    multimodal: {
      enabled: true,
      modalities: ["image", "audio"],  // or ["all"]
      maxFileBytes: 10000000,  // 10MB
    },
  },
}
```

**Supported formats:**
- Images: jpg, jpeg, png, webp, gif, heic, heif
- Audio: mp3, wav, ogg, opus, m4a, aac, flac

## Verifying Setup

### Check Status

```bash
openclaw memory status

# Expected output:
# Memory Search: enabled
# Provider: gemini (gemini-embedding-001)
# Index: ~/.openclaw/memory/main.sqlite
# Chunks: 1,247
# Last indexed: 2 minutes ago
# Vector backend: sqlite-vec
```

### Test Search

```bash
# Via CLI
openclaw memory search "project deadline"

# Or message your bot
User: "What do you remember about deadlines?"
Agent: *calls memory_search* → finds relevant entries
```

### Check Logs

```bash
openclaw logs --follow | grep -i "memory\|embed"

# Look for:
# [memory] Indexing MEMORY.md (42 chunks)
# [memory] Embedding provider: gemini
# [memory] Index updated: 1,247 chunks
```

## Troubleshooting

### "No embedding provider configured"

```bash
# Check config
grep -A10 "memorySearch" ~/.openclaw/openclaw.json

# Verify API key
echo $GEMINI_API_KEY  # Should not be empty
```

### "Embedding failed"

- Check API key is valid
- Check provider is accessible (network)
- Check rate limits haven't been exceeded

```bash
# Test API directly
curl -X POST "https://generativelanguage.googleapis.com/v1/models/gemini-embedding-001:embedContent?key=$GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"content": {"parts": [{"text": "test"}]}}'
```

### "Index is stale"

```bash
# Force reindex
openclaw memory reindex

# Or restart gateway
openclaw gateway restart
```

### Local provider won't load

```bash
# Check node-llama-cpp is built
npm list node-llama-cpp

# Rebuild if needed
pnpm approve-builds
pnpm rebuild node-llama-cpp
```

## Complete Configuration Example

```json5
// Full memory search setup with Gemini
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      
      memorySearch: {
        enabled: true,
        
        // Primary provider
        provider: "gemini",
        model: "gemini-embedding-001",
        
        // Fallback if Gemini fails
        fallback: "openai",
        
        // Additional directories to index
        extraPaths: ["../shared-notes"],
        
        // Storage
        store: {
          path: "~/.openclaw/memory/{agentId}.sqlite",
          vector: { enabled: true },
        },
        
        // Caching
        cache: {
          enabled: true,
          maxEntries: 50000,
        },
        
        // File watching
        sync: {
          watch: true,
        },
      },
    },
  },
}
```

## Next Steps

Vector search alone is powerful, but combining it with keyword search is even better. Learn about [Hybrid Search](./04-hybrid-search.md) →
