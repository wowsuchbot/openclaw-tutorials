# Advanced Search Features

Fine-tune search results with MMR diversity, temporal decay, and result filtering.

## Post-Processing Pipeline

After hybrid search retrieves candidates, OpenClaw applies post-processing:

```
Hybrid Search Results (top 50)
           ↓
    Score Threshold
    (filter low scores)
           ↓
    Temporal Decay
    (boost recent content)
           ↓
    MMR Reranking
    (diversify results)
           ↓
    Final Results (top K)
```

## MMR (Maximal Marginal Relevance)

MMR balances relevance with diversity — avoiding redundant results.

### The Problem MMR Solves

Without MMR:
```
Query: "deployment process"

Results:
1. "Deployment steps for staging" (score: 0.95)
2. "Deployment steps for production" (score: 0.94)  ← Similar to #1
3. "Deployment checklist v2" (score: 0.93)          ← Similar to #1 and #2
4. "Deployment troubleshooting" (score: 0.91)       ← Finally different!
```

With MMR:
```
Query: "deployment process"

Results:
1. "Deployment steps for staging" (score: 0.95)
2. "Deployment troubleshooting" (score: 0.91)       ← Diverse!
3. "CI/CD pipeline overview" (score: 0.88)          ← Different angle
4. "Deployment checklist v2" (score: 0.93)          ← Included but ranked lower
```

### Enabling MMR

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "gemini",
        
        postProcess: {
          mmr: {
            enabled: true,
            lambda: 0.7,  // Balance: relevance vs diversity
          },
        },
      },
    },
  },
}
```

### Lambda Tuning

Lambda (λ) controls the relevance-diversity tradeoff:

| Lambda | Behavior | Use Case |
|--------|----------|----------|
| 1.0 | Pure relevance (no diversity) | When you want most similar |
| 0.7 | Mostly relevant, some diversity | General use (default) |
| 0.5 | Balanced | Exploratory searches |
| 0.3 | Mostly diverse | Brainstorming, research |
| 0.0 | Maximum diversity | Survey all topics |

**MMR Formula:**
```
score = λ * relevance(doc, query) - (1-λ) * max_similarity(doc, selected_docs)
```

### Example Configurations

**Research/exploration (more diversity):**
```json5
{
  postProcess: {
    mmr: {
      enabled: true,
      lambda: 0.5,
    },
  },
}
```

**Precise lookup (less diversity):**
```json5
{
  postProcess: {
    mmr: {
      enabled: true,
      lambda: 0.85,
    },
  },
}
```

## Temporal Decay

Boost recent content — because newer information is often more relevant.

### How It Works

```
Today's notes      → 1.0x multiplier (no decay)
Yesterday's notes  → 0.97x multiplier
Last week's notes  → 0.85x multiplier
Last month's notes → 0.5x multiplier
Last year's notes  → 0.1x multiplier
```

### Configuration

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        postProcess: {
          temporal: {
            enabled: true,
            
            // Half-life in days (default: 30)
            // After this many days, score is halved
            halfLife: 30,
            
            // Minimum multiplier (default: 0.1)
            // Old content never drops below 10% of original score
            floor: 0.1,
          },
        },
      },
    },
  },
}
```

### Half-Life Examples

| Half-Life | 1 week ago | 1 month ago | 1 year ago | Use Case |
|-----------|------------|-------------|------------|----------|
| 7 days | 0.5x | 0.06x | ~0x | Fast-moving projects |
| 30 days | 0.87x | 0.5x | 0.1x | Normal work (default) |
| 90 days | 0.95x | 0.79x | 0.26x | Long-term reference |
| 365 days | 0.99x | 0.94x | 0.5x | Archival content |

### Disabling for Specific Searches

Sometimes you want historical content without decay:

```bash
# CLI override
openclaw memory search "original project requirements" --no-temporal
```

Or in agent configuration for a "historian" agent:

```json5
{
  agents: {
    historian: {
      memorySearch: {
        postProcess: {
          temporal: { enabled: false },
        },
      },
    },
  },
}
```

## Score Threshold

Filter out low-relevance results before they waste context:

```json5
{
  memorySearch: {
    postProcess: {
      threshold: {
        // Minimum score to include (0-1 scale)
        minScore: 0.3,
        
        // Or minimum relative to top result
        minRelative: 0.5,  // Must be at least 50% of top score
      },
    },
  },
}
```

**Why threshold?**
- Prevents irrelevant results from consuming context
- Reduces noise in agent responses
- Saves tokens

## Result Limits

Control how many results to return:

```json5
{
  memorySearch: {
    // Maximum results returned to agent
    topK: 10,
    
    // Candidates to consider before post-processing
    // (should be larger than topK for MMR to work well)
    candidatePool: 50,
  },
}
```

**Guidelines:**
- `topK`: 5-15 for most use cases
- `candidatePool`: 3-5x of `topK` for good MMR diversity

## Chunking Configuration

How documents are split affects search quality:

```json5
{
  memorySearch: {
    chunking: {
      // Target chunk size in tokens
      chunkSize: 512,
      
      // Overlap between chunks (helps context continuity)
      chunkOverlap: 50,
      
      // Respect document structure
      strategy: "semantic",  // or "fixed", "sentence"
    },
  },
}
```

### Chunking Strategies

| Strategy | Behavior | Best For |
|----------|----------|----------|
| `fixed` | Split at exact token count | Uniform content |
| `sentence` | Split at sentence boundaries | Prose, notes |
| `semantic` | Split at paragraph/section breaks | Structured docs |

### Chunk Size Guidelines

| Content Type | Chunk Size | Overlap | Why |
|--------------|------------|---------|-----|
| Quick notes | 256 | 25 | Small, self-contained |
| Daily logs | 512 | 50 | Medium entries (default) |
| Documentation | 1024 | 100 | Longer explanations |
| Code files | 256 | 50 | Functions/blocks |

## File Filtering

Control which files are indexed:

```json5
{
  memorySearch: {
    // Include patterns
    include: ["*.md", "*.txt", "notes/**/*"],
    
    // Exclude patterns
    exclude: ["**/node_modules/**", "**/.git/**", "**/drafts/**"],
    
    // Additional paths outside workspace
    extraPaths: ["~/shared-notes", "/team/knowledge-base"],
  },
}
```

## Per-Agent Tuning

Different agents may need different search settings:

```json5
{
  agents: {
    // Default for all agents
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "gemini",
        topK: 10,
        postProcess: {
          mmr: { enabled: true, lambda: 0.7 },
          temporal: { enabled: true, halfLife: 30 },
        },
      },
    },
    
    // Research agent: more diversity, less recency bias
    researcher: {
      memorySearch: {
        topK: 20,
        postProcess: {
          mmr: { enabled: true, lambda: 0.5 },
          temporal: { enabled: false },
        },
      },
    },
    
    // Quick assistant: fewer results, strong recency
    assistant: {
      memorySearch: {
        topK: 5,
        postProcess: {
          mmr: { enabled: true, lambda: 0.8 },
          temporal: { enabled: true, halfLife: 7 },
        },
      },
    },
  },
}
```

## Testing and Tuning

### Evaluate Search Quality

```bash
# Run test queries and examine results
openclaw memory search "project deadline" --verbose --explain

# Output shows:
# - Raw scores before post-processing
# - Temporal decay multipliers applied
# - MMR diversity penalties
# - Final reranked scores
```

### A/B Test Settings

```bash
# Compare configurations
openclaw memory search "deployment" --mmr-lambda 0.5
openclaw memory search "deployment" --mmr-lambda 0.8

# Compare with/without temporal decay
openclaw memory search "requirements" --temporal
openclaw memory search "requirements" --no-temporal
```

### Monitor Search Metrics

```bash
# Check search statistics
openclaw memory stats

# Shows:
# - Average query latency
# - Results per query (mean, median)
# - Score distribution
# - Cache hit rate
```

## Complete Configuration

```json5
// Full advanced search setup
{
  agents: {
    defaults: {
      memorySearch: {
        enabled: true,
        provider: "gemini",
        model: "gemini-embedding-001",
        
        // Hybrid search
        hybrid: {
          enabled: true,
          vectorWeight: 0.6,
          keywordWeight: 0.4,
        },
        
        // Result limits
        topK: 10,
        candidatePool: 50,
        
        // Chunking
        chunking: {
          chunkSize: 512,
          chunkOverlap: 50,
          strategy: "semantic",
        },
        
        // Post-processing pipeline
        postProcess: {
          // Score filtering
          threshold: {
            minScore: 0.3,
          },
          
          // Recency boost
          temporal: {
            enabled: true,
            halfLife: 30,
            floor: 0.1,
          },
          
          // Diversity
          mmr: {
            enabled: true,
            lambda: 0.7,
          },
        },
        
        // File filtering
        include: ["*.md", "*.txt"],
        exclude: ["**/drafts/**"],
      },
    },
  },
}
```

## Troubleshooting

### "Results all seem the same"

Enable or tune MMR:
```json5
{
  postProcess: {
    mmr: { enabled: true, lambda: 0.5 },  // Lower lambda = more diversity
  },
}
```

### "Can't find old content"

Temporal decay might be too aggressive:
```json5
{
  postProcess: {
    temporal: {
      halfLife: 90,  // Longer half-life
      floor: 0.3,    // Higher minimum score
    },
  },
}
```

### "Too many irrelevant results"

Raise the threshold:
```json5
{
  postProcess: {
    threshold: { minScore: 0.5 },
  },
}
```

### "Missing context in results"

Increase chunk overlap:
```json5
{
  chunking: {
    chunkOverlap: 100,  // More overlap
  },
}
```

## Next Steps

For even more powerful search capabilities, including reranking and advanced BM25, see the [QMD Backend](./06-qmd-backend.md) →
