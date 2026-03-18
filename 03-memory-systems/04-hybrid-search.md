# Hybrid Search

Combine keyword matching (BM25) with semantic search (vectors) for the best of both worlds.

## Why Hybrid?

Vector search and keyword search have complementary strengths:

| Scenario | Vector Only | Keyword Only | Hybrid |
|----------|-------------|--------------|--------|
| "What's the deadline for Project Alpha?" | Finds "due date" and "Project Alpha" semantically | Misses if you wrote "due date" | Finds both exact matches AND semantic equivalents |
| "Error code E-1042" | May find unrelated errors | Exact match if present | Guarantees exact code + related error context |
| Rare technical terms | May hallucinate similarity | Precise but limited | Best precision + recall |
| Conceptual questions | Excellent | Poor | Excellent |

**Rule of thumb:** Hybrid search is almost always better than either alone.

## How Hybrid Works

```
User Query: "deployment pipeline failures"
           ↓
    ┌──────┴──────┐
    ↓             ↓
  BM25          Vector
  Search        Search
    ↓             ↓
┌─────────┐  ┌─────────┐
│ Results │  │ Results │
│ (exact) │  │(semantic)│
└────┬────┘  └────┬────┘
     │            │
     └─────┬──────┘
           ↓
    Score Fusion
    (RRF or weighted)
           ↓
    Unified Results
    (best of both)
```

## Enabling Hybrid Search

### Basic Setup

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "gemini",
        
        // Enable hybrid search
        hybrid: {
          enabled: true,
        },
      },
    },
  },
}
```

That's it! OpenClaw uses sensible defaults:
- **Fusion method:** RRF (Reciprocal Rank Fusion)
- **Weight balance:** 50/50 vector/keyword
- **BM25 backend:** Built-in SQLite FTS5

### Custom Weighting

Adjust the balance based on your use case:

```json5
{
  memorySearch: {
    hybrid: {
      enabled: true,
      
      // Weight distribution (must sum to 1.0)
      vectorWeight: 0.6,   // 60% semantic
      keywordWeight: 0.4,  // 40% exact match
    },
  },
}
```

**Tuning guidelines:**

| Use Case | Vector Weight | Keyword Weight | Why |
|----------|---------------|----------------|-----|
| General knowledge | 0.6 | 0.4 | Concepts matter more |
| Technical docs | 0.4 | 0.6 | Exact terms matter |
| Code references | 0.3 | 0.7 | Identifiers must match |
| Creative writing | 0.7 | 0.3 | Themes over words |
| Mixed content | 0.5 | 0.5 | Balanced (default) |

## BM25 Configuration

BM25 (Best Match 25) is the keyword search algorithm. Tune it for your content:

```json5
{
  memorySearch: {
    hybrid: {
      enabled: true,
      
      bm25: {
        // Term frequency saturation (default: 1.2)
        // Higher = repeated terms matter more
        k1: 1.2,
        
        // Document length normalization (default: 0.75)
        // Higher = penalize long documents more
        b: 0.75,
      },
    },
  },
}
```

**When to adjust:**

- **Short notes/snippets:** Lower `b` (0.5) — don't penalize brevity
- **Long documents:** Higher `b` (0.9) — normalize for length
- **Repetitive content:** Lower `k1` (0.8) — diminish repeated term impact
- **Diverse vocabulary:** Higher `k1` (1.5) — reward specific term usage

## Score Fusion Methods

### RRF (Reciprocal Rank Fusion) — Default

Combines rankings rather than raw scores. Robust and stable.

```json5
{
  memorySearch: {
    hybrid: {
      enabled: true,
      fusion: "rrf",
      
      rrf: {
        // Constant to prevent division issues (default: 60)
        k: 60,
      },
    },
  },
}
```

**RRF Formula:**
```
score = 1/(k + rank_vector) + 1/(k + rank_keyword)
```

**Why RRF is default:**
- Works without score normalization
- Resistant to outliers
- Consistent across different providers

### Weighted Sum

Direct score combination. More tunable but requires good score normalization.

```json5
{
  memorySearch: {
    hybrid: {
      enabled: true,
      fusion: "weighted",
      
      vectorWeight: 0.6,
      keywordWeight: 0.4,
      
      // Score normalization (important for weighted fusion)
      normalize: true,  // default
    },
  },
}
```

**Weighted Sum Formula:**
```
score = (vectorWeight * normalized_vector_score) + 
        (keywordWeight * normalized_keyword_score)
```

## Practical Examples

### Example 1: Developer Knowledge Base

Lots of code snippets, error messages, exact identifiers:

```json5
{
  memorySearch: {
    provider: "gemini",
    hybrid: {
      enabled: true,
      vectorWeight: 0.4,
      keywordWeight: 0.6,
      
      bm25: {
        k1: 1.5,  // Reward specific technical terms
        b: 0.5,   // Don't penalize short code snippets
      },
    },
  },
}
```

### Example 2: Personal Journal

Reflective writing, emotional content, themes over keywords:

```json5
{
  memorySearch: {
    provider: "gemini",
    hybrid: {
      enabled: true,
      vectorWeight: 0.7,
      keywordWeight: 0.3,
      
      bm25: {
        k1: 1.0,  // Standard term frequency
        b: 0.75,  // Standard length normalization
      },
    },
  },
}
```

### Example 3: Project Documentation

Mix of specifications, meeting notes, and decisions:

```json5
{
  memorySearch: {
    provider: "openai",
    model: "text-embedding-3-large",
    
    hybrid: {
      enabled: true,
      vectorWeight: 0.5,
      keywordWeight: 0.5,
      fusion: "rrf",
    },
  },
}
```

## Testing Hybrid Search

### Compare Search Modes

```bash
# Vector only
openclaw memory search "deployment issues" --mode vector

# Keyword only  
openclaw memory search "deployment issues" --mode keyword

# Hybrid (default when enabled)
openclaw memory search "deployment issues" --mode hybrid
```

### Analyze Results

```bash
# Show scores and which backend found each result
openclaw memory search "error E-1042" --verbose

# Output:
# 1. [0.92] memory/2024-01-15.md:42 (keyword: 0.95, vector: 0.88)
#    "Investigated error E-1042, found root cause in auth module"
# 2. [0.78] MEMORY.md:156 (keyword: 0.60, vector: 0.89)
#    "Common authentication errors and their solutions"
```

### A/B Testing Weights

```bash
# Test different weight configurations
openclaw memory search "project deadline" --vector-weight 0.3
openclaw memory search "project deadline" --vector-weight 0.7
```

## When to Disable Hybrid

Sometimes pure vector or pure keyword is better:

### Use Vector Only When:
- Content is highly conceptual
- Exact terminology varies widely
- You're doing similarity/clustering analysis

```json5
{
  memorySearch: {
    provider: "gemini",
    hybrid: { enabled: false },
    // Pure vector search
  },
}
```

### Use Keyword Only When:
- Searching for exact codes, IDs, or identifiers
- Content is highly structured
- You need deterministic, explainable results

```json5
{
  memorySearch: {
    provider: "gemini",
    hybrid: { enabled: true, vectorWeight: 0, keywordWeight: 1 },
    // Pure keyword search with vector infrastructure
  },
}
```

## Performance Considerations

### Index Size

Hybrid search requires both indexes:
- **FTS5 index** (BM25): ~10-20% of text size
- **Vector index**: ~4KB per chunk (768-dim embeddings)

```bash
# Check index sizes
ls -lh ~/.openclaw/memory/main.sqlite
# Typical: 5-50MB for moderate memory collections
```

### Query Latency

Hybrid adds minimal overhead:
- **Vector only:** ~50-200ms (depends on provider)
- **BM25 only:** ~5-20ms (local SQLite)
- **Hybrid:** ~60-220ms (parallel execution)

Both searches run in parallel, so hybrid ≈ max(vector, keyword) + fusion overhead.

### Memory Usage

The BM25 index is memory-mapped, adding minimal RAM overhead. Vector search memory depends on your provider configuration.

## Troubleshooting

### "Hybrid search not working"

```bash
# Check if enabled
openclaw memory status

# Should show:
# Hybrid search: enabled
# BM25 index: 1,247 terms
# Vector index: 1,247 chunks
```

### "Keyword results dominating"

Your keyword weight may be too high, or BM25 is matching aggressively:

```json5
{
  hybrid: {
    vectorWeight: 0.6,  // Increase
    keywordWeight: 0.4, // Decrease
  },
}
```

### "Results seem random"

Score normalization might be off:

```json5
{
  hybrid: {
    fusion: "rrf",  // Switch to RRF (more stable)
  },
}
```

## Complete Configuration

```json5
// Full hybrid search setup
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "gemini",
        model: "gemini-embedding-001",
        
        hybrid: {
          enabled: true,
          
          // Score fusion
          fusion: "rrf",
          rrf: { k: 60 },
          
          // Weight balance
          vectorWeight: 0.5,
          keywordWeight: 0.5,
          
          // BM25 tuning
          bm25: {
            k1: 1.2,
            b: 0.75,
          },
        },
        
        // Storage
        store: {
          path: "~/.openclaw/memory/{agentId}.sqlite",
          vector: { enabled: true },
        },
      },
    },
  },
}
```

## Next Steps

Hybrid search gets you excellent results. But for even better relevance, learn about post-processing techniques like MMR and temporal decay in [Advanced Search](./05-advanced-search.md) →
