# QMD Backend

Optional sidecar service for advanced BM25, reranking, and high-performance search.

## What is QMD?

QMD (Query Memory Daemon) is an optional Rust-based sidecar that provides:

- **High-performance BM25** — Native implementation, faster than SQLite FTS5
- **Cross-encoder reranking** — Two-stage retrieval for better relevance
- **Advanced tokenization** — Language-aware stemming and normalization
- **Distributed search** — Scale across multiple nodes

**When to use QMD:**
- Large memory collections (10K+ chunks)
- Need for cross-encoder reranking
- Multi-agent deployments sharing search infrastructure
- Maximum search performance requirements

**When NOT needed:**
- Small to medium collections
- Single-agent setups
- Built-in SQLite hybrid search is sufficient

## Architecture

```
Without QMD (default):
┌─────────────────┐
│    OpenClaw     │
│  ┌───────────┐  │
│  │ SQLite    │  │
│  │ FTS5+Vec  │  │
│  └───────────┘  │
└─────────────────┘

With QMD:
┌─────────────────┐     ┌─────────────────┐
│    OpenClaw     │────▶│       QMD       │
│  (embedding     │     │  ┌───────────┐  │
│   + routing)    │     │  │ BM25 Index│  │
└─────────────────┘     │  │ Reranker  │  │
                        │  └───────────┘  │
                        └─────────────────┘
```

## Installation

### Docker (Recommended)

```bash
# Pull QMD image
docker pull ghcr.io/openclaw/qmd:latest

# Run with default settings
docker run -d \
  --name qmd \
  -p 8765:8765 \
  -v ~/.openclaw/qmd-data:/data \
  ghcr.io/openclaw/qmd:latest
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  qmd:
    image: ghcr.io/openclaw/qmd:latest
    ports:
      - "8765:8765"
    volumes:
      - qmd-data:/data
    environment:
      - QMD_LOG_LEVEL=info
      - QMD_RERANKER_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
    restart: unless-stopped

volumes:
  qmd-data:
```

```bash
docker-compose up -d
```

### Binary Installation

```bash
# Download latest release
curl -L https://github.com/openclaw/qmd/releases/latest/download/qmd-linux-amd64 -o qmd
chmod +x qmd

# Run
./qmd serve --port 8765 --data-dir ~/.openclaw/qmd-data
```

## OpenClaw Configuration

### Basic QMD Integration

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "gemini",
        
        // Enable QMD backend
        qmd: {
          enabled: true,
          url: "http://localhost:8765",
        },
      },
    },
  },
}
```

### With Reranking

```json5
{
  memorySearch: {
    provider: "gemini",
    
    qmd: {
      enabled: true,
      url: "http://localhost:8765",
      
      // Enable cross-encoder reranking
      rerank: {
        enabled: true,
        model: "cross-encoder/ms-marco-MiniLM-L-6-v2",
        topK: 20,  // Rerank top 20 candidates
      },
    },
  },
}
```

### Full Configuration

```json5
{
  memorySearch: {
    provider: "gemini",
    model: "gemini-embedding-001",
    
    hybrid: {
      enabled: true,
      vectorWeight: 0.5,
      keywordWeight: 0.5,
    },
    
    qmd: {
      enabled: true,
      url: "http://localhost:8765",
      
      // Connection settings
      timeout: 5000,
      retries: 3,
      
      // BM25 settings (handled by QMD)
      bm25: {
        k1: 1.2,
        b: 0.75,
        avgDocLength: null,  // Auto-computed
      },
      
      // Reranking
      rerank: {
        enabled: true,
        model: "cross-encoder/ms-marco-MiniLM-L-6-v2",
        topK: 20,
        threshold: 0.1,  // Minimum rerank score
      },
      
      // Index sync
      sync: {
        mode: "push",  // "push" | "pull" | "bidirectional"
        interval: 60,  // Seconds between syncs
      },
    },
  },
}
```

## Reranking Deep Dive

### Two-Stage Retrieval

```
Stage 1: Fast Retrieval (BM25 + Vector)
─────────────────────────────────────────
Query → Hybrid Search → Top 100 candidates
                       (fast, approximate)

Stage 2: Precise Reranking (Cross-Encoder)
─────────────────────────────────────────
Top 100 → Cross-Encoder → Top 10 results
          (slow, precise)
```

**Why two stages?**
- Cross-encoders are accurate but slow (~100ms per pair)
- First stage quickly narrows down candidates
- Second stage precisely ranks the best ones

### Reranker Models

| Model | Quality | Speed | Size |
|-------|---------|-------|------|
| `ms-marco-MiniLM-L-6-v2` | Good | Fast | 80MB |
| `ms-marco-TinyBERT-L-2-v2` | Fair | Very Fast | 25MB |
| `ms-marco-MiniLM-L-12-v2` | Better | Medium | 120MB |
| `bge-reranker-base` | Best | Slow | 400MB |

### Configure Reranker

```json5
{
  qmd: {
    rerank: {
      enabled: true,
      
      // Model selection
      model: "cross-encoder/ms-marco-MiniLM-L-6-v2",
      
      // How many candidates to rerank
      topK: 20,
      
      // Minimum score after reranking
      threshold: 0.1,
      
      // Batch size for reranking
      batchSize: 32,
    },
  },
}
```

## QMD Server Configuration

### Environment Variables

```bash
# QMD server settings
export QMD_PORT=8765
export QMD_DATA_DIR=~/.openclaw/qmd-data
export QMD_LOG_LEVEL=info

# Reranker settings
export QMD_RERANKER_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
export QMD_RERANKER_DEVICE=cpu  # or "cuda" for GPU

# Performance tuning
export QMD_MAX_CONNECTIONS=100
export QMD_WORKER_THREADS=4
```

### Config File

```toml
# ~/.openclaw/qmd.toml
[server]
port = 8765
data_dir = "~/.openclaw/qmd-data"
log_level = "info"

[bm25]
k1 = 1.2
b = 0.75

[reranker]
enabled = true
model = "cross-encoder/ms-marco-MiniLM-L-6-v2"
device = "cpu"
batch_size = 32

[performance]
max_connections = 100
worker_threads = 4
cache_size_mb = 512
```

## Index Management

### View Index Status

```bash
# QMD CLI
qmd status

# Output:
# Index: main
#   Documents: 1,247
#   Terms: 45,892
#   Size: 12.4 MB
#   Last updated: 2024-01-15 10:30:00
```

### Reindex

```bash
# Full reindex
qmd reindex --index main

# Incremental update
qmd sync --index main
```

### Multiple Indexes

QMD can maintain separate indexes per agent:

```json5
{
  agents: {
    main: {
      memorySearch: {
        qmd: {
          enabled: true,
          index: "main",  // Uses "main" index
        },
      },
    },
    researcher: {
      memorySearch: {
        qmd: {
          enabled: true,
          index: "researcher",  // Separate index
        },
      },
    },
  },
}
```

## Performance Tuning

### For High Volume

```toml
# qmd.toml
[performance]
worker_threads = 8
cache_size_mb = 1024
max_connections = 500

[bm25]
# Pre-compute document lengths
precompute_doc_lengths = true
```

### For Low Memory

```toml
# qmd.toml
[performance]
worker_threads = 2
cache_size_mb = 128

[reranker]
# Use smaller model
model = "cross-encoder/ms-marco-TinyBERT-L-2-v2"
# Reduce batch size
batch_size = 8
```

### GPU Acceleration

```toml
# qmd.toml
[reranker]
device = "cuda"
batch_size = 64  # Larger batches benefit from GPU
```

```bash
# Docker with GPU
docker run -d \
  --name qmd \
  --gpus all \
  -p 8765:8765 \
  -v ~/.openclaw/qmd-data:/data \
  ghcr.io/openclaw/qmd:latest-cuda
```

## Monitoring

### Health Check

```bash
curl http://localhost:8765/health

# Response:
# {"status": "healthy", "uptime": 3600, "indexes": ["main"]}
```

### Metrics

```bash
curl http://localhost:8765/metrics

# Prometheus-format metrics:
# qmd_queries_total{index="main"} 1234
# qmd_query_latency_seconds{quantile="0.95"} 0.045
# qmd_index_size_bytes{index="main"} 13000000
```

### Logs

```bash
# Docker logs
docker logs -f qmd

# Or with journald
journalctl -u qmd -f
```

## Fallback Behavior

If QMD is unavailable, OpenClaw falls back to built-in search:

```json5
{
  memorySearch: {
    qmd: {
      enabled: true,
      url: "http://localhost:8765",
      
      // Fallback settings
      fallback: {
        enabled: true,  // Use SQLite if QMD unavailable
        warnOnFallback: true,  // Log when falling back
      },
    },
  },
}
```

## Troubleshooting

### "QMD connection refused"

```bash
# Check if QMD is running
curl http://localhost:8765/health

# Check Docker
docker ps | grep qmd
docker logs qmd

# Restart
docker restart qmd
```

### "Reranker model not found"

```bash
# Download model manually
qmd download-model cross-encoder/ms-marco-MiniLM-L-6-v2

# Or in Docker
docker exec qmd qmd download-model cross-encoder/ms-marco-MiniLM-L-6-v2
```

### "Index out of sync"

```bash
# Force resync
qmd sync --force --index main

# Or trigger from OpenClaw
openclaw memory sync --qmd
```

### "Slow reranking"

- Reduce `topK` for reranking
- Use smaller reranker model
- Enable GPU acceleration
- Increase batch size

## When QMD vs Built-in

| Factor | Use Built-in | Use QMD |
|--------|--------------|---------|
| Collection size | < 10K chunks | > 10K chunks |
| Reranking needed | No | Yes |
| Multi-agent | Single | Multiple sharing index |
| Setup complexity | Minimal | Acceptable |
| Search latency needs | Good enough | Sub-50ms critical |
| Resource constraints | Limited | Adequate |

## Complete Setup Example

```yaml
# docker-compose.yml
version: '3.8'
services:
  openclaw:
    image: ghcr.io/openclaw/gateway:latest
    environment:
      - GEMINI_API_KEY=${GEMINI_API_KEY}
    volumes:
      - ./openclaw.json:/root/.openclaw/openclaw.json
      - ./workspace:/root/.openclaw/workspace
    depends_on:
      - qmd

  qmd:
    image: ghcr.io/openclaw/qmd:latest
    ports:
      - "8765:8765"
    volumes:
      - qmd-data:/data
    environment:
      - QMD_RERANKER_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2

volumes:
  qmd-data:
```

```json5
// openclaw.json
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "gemini",
        
        hybrid: { enabled: true },
        
        qmd: {
          enabled: true,
          url: "http://qmd:8765",
          rerank: {
            enabled: true,
            topK: 20,
          },
          fallback: { enabled: true },
        },
      },
    },
  },
}
```

## Next Steps

Now that you have powerful search, learn how to curate what gets remembered in [Memory Curation](./07-memory-curation.md) →
