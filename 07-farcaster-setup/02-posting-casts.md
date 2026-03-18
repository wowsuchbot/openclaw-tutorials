# Posting Casts

Learn to post casts, build threads, embed content, and reply to conversations.

## Basic Posting

### Simple Text Cast

```bash
./skills/farcaster-skill/scripts/fc_cast.sh --text "Hello Farcaster!"
```

**Constraints:**
- Max 320 characters
- No hashtags (Farcaster culture discourages them)
- Write conversationally, not keyword-stuffed

### Post to a Channel

```bash
./skills/farcaster-skill/scripts/fc_cast.sh \
  --text "Exploring onchain generative art" \
  --channel "cryptoart"
```

Popular channels: `base`, `cryptoart`, `dev`, `frames`, `memes`

## Embeds

### URL Embed (Link Preview)

```bash
./skills/farcaster-skill/scripts/fc_cast.sh \
  --text "Check out this project" \
  --embed "https://example.com/cool-thing"
```

The URL automatically generates an OpenGraph preview card.

### Image Embed

URLs ending in image extensions auto-display as images:

```bash
./skills/farcaster-skill/scripts/fc_cast.sh \
  --text "New artwork" \
  --embed "https://example.com/art.png"
```

## Quote-Casts

A quote-cast embeds another cast with your commentary. Different from a recast (which just re-shares without comment).

### When to Quote-Cast

**Do quote-cast:**
- Adding meaningful context to someone's point
- Amplifying with your own perspective
- Respectfully engaging with a disagreement

**Don't quote-cast:**
- Just saying "this" or "+1" (use like/recast instead)
- Starting drama or calling someone out
- When a simple reply would suffice

### How to Quote-Cast

```bash
./skills/farcaster-skill/scripts/fc_cast.sh \
  --text "This thread on FIP-8 is worth reading" \
  --embed-cast "0xabc123..." \
  --embed-cast-fid 3
```

**Required parameters:**
- `--embed-cast`: The hash of the cast to quote
- `--embed-cast-fid`: The FID of the cast's author

### Quote-Cast Wrapper

For convenience, use the wrapper script:

```bash
./skills/farcaster-skill/scripts/quote_cast.sh "Great analysis here" "0xabc123..."
```

This handles the FID lookup automatically.

## Replies

Replies are casts with a `--parent` reference.

### Basic Reply

```bash
./skills/farcaster-skill/scripts/fc_cast.sh \
  --text "Great point! I'd add that..." \
  --parent "0xabc123..."
```

### Critical Reply Rules

1. **No username prefix** - Just write the reply text
   - WRONG: `"@alice: Here's what I think..."`
   - CORRECT: `"Here's what I think..."`

2. **Get context first** - Always fetch the conversation before replying
   ```bash
   ./scripts/fc_get_conversation.sh "0xabc123..." --parent-casts
   ```

3. **One at a time** - Process replies sequentially, not in batch

4. **Complete text** - Generate full reply before posting (no truncation)

### Reply Workflow

```bash
# 1. Get conversation context
CONTEXT=$(./scripts/fc_get_conversation.sh "$CAST_HASH" --parent-casts)

# 2. Analyze context, formulate response
# (your agent logic here)

# 3. Post reply
./scripts/fc_cast.sh --text "$REPLY_TEXT" --parent "$CAST_HASH"
```

## Threads

Build multi-cast threads with `fc_thread.sh`.

### Simple Thread

```bash
./skills/farcaster-skill/scripts/fc_thread.sh \
  "Good morning! Some thoughts:" \
  --text "First, the market is showing interesting patterns" \
  --text "Second, new art dropped that's worth checking out" \
  --text "Third, a reminder to touch grass today"
```

### Thread in a Channel

```bash
./skills/farcaster-skill/scripts/fc_thread.sh \
  --channel cryptoart \
  "A thread on generative art techniques:" \
  --text "Let's talk about noise functions..." \
  --text "And how they create organic patterns..."
```

### Thread with Embeds

```bash
./scripts/fc_thread.sh \
  "Artworks I'm watching today:" \
  --listing "@artist — Sunset https://example.com/auction"
```

## Deleting Casts

Remove a cast you've posted:

```bash
./skills/farcaster-skill/scripts/fc_delete.sh --hash "0xabc123..."
```

Only works for casts from your signer's FID.

## Style Guidelines

Based on Farcaster culture:

1. **No hashtags** - They're discouraged on Farcaster
2. **Write naturally** - Conversational, not marketing-speak
3. **Tease, don't summarize** - Create curiosity
4. **No keyword stuffing** - Quality over SEO
5. **Share links after posting** - Let others find your cast

## Rate Limiting

Scripts include built-in safeguards:
- Cooldown between posts
- Duplicate detection

If you see "Safeguard abort", wait before retrying or vary your content.

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Username prefix in reply | Looks spammy | Just write reply text |
| Batch posting replies | Rate limits, looks bot-like | One at a time |
| Truncated text | Incomplete thoughts | Generate full text first |
| Wrong script path | Command not found | Use full workspace path |

## Example: Agent Posting Workflow

```bash
#!/bin/bash
# Example: Agent curates and shares content

SKILL_PATH="./skills/farcaster-skill/scripts"

# 1. Find interesting content
RESULTS=$($SKILL_PATH/fc_search.sh --query "generative art" --limit 5)

# 2. Select best cast to highlight
# (agent analysis here)
CAST_HASH="0x..."
CAST_FID="12345"

# 3. Post quote-cast with commentary
$SKILL_PATH/fc_cast.sh \
  --text "This approach to noise functions is fascinating" \
  --embed-cast "$CAST_HASH" \
  --embed-cast-fid "$CAST_FID" \
  --channel "cryptoart"
```

## Next Steps

Learn to read feeds and search content in [03-reading-feeds.md](./03-reading-feeds.md).
