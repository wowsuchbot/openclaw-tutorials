# Module 7: Farcaster Setup

Integrate your OpenClaw agents with Farcaster, the decentralized social protocol. This module covers the Neynar API, posting casts, reading feeds, and building engagement workflows.

## Prerequisites

- Completed Modules 1-3 (Gateway, Sessions, Memory fundamentals)
- A Farcaster account
- Neynar API access (free tier available)

## What You'll Learn

1. **Neynar Setup** - API keys, signer creation, environment configuration
2. **Posting Casts** - Text, embeds, threads, quote-casts, and replies
3. **Reading Feeds** - User feeds, channels, search, and algorithmic feeds
4. **Engagement Workflows** - Replies, reactions, and conversation context

## Why Farcaster?

Farcaster offers several advantages for AI agents:

- **Decentralized**: Your data isn't locked in a platform
- **Developer-friendly**: Robust APIs via Neynar
- **Crypto-native**: Built-in wallet verification and token-gated features
- **Community**: Active, technical user base receptive to AI experiments

## The Farcaster Skill

OpenClaw provides a comprehensive Farcaster skill with 20+ bash scripts wrapping the Neynar v2 API. Zero npm dependencies - just `curl` and `jq`.

```
skills/farcaster-skill/scripts/
├── fc_cast.sh              # Post casts
├── fc_thread.sh            # Build threads
├── fc_delete.sh            # Delete casts
├── fc_react.sh             # Like/recast
├── fc_feed.sh              # Read feeds
├── fc_search.sh            # Search casts
├── fc_user.sh              # User lookups
├── fc_channels.sh          # Channel operations
├── fc_get_conversation.sh  # Full thread context
└── ... (and more)
```

## Module Structure

| File | Topic |
|------|-------|
| [01-neynar-setup.md](./01-neynar-setup.md) | API keys, signers, environment |
| [02-posting-casts.md](./02-posting-casts.md) | Text, embeds, threads, replies |
| [03-reading-feeds.md](./03-reading-feeds.md) | Feeds, search, channels |
| [04-engagement.md](./04-engagement.md) | Reactions, replies, conversation context |

## Quick Start

```bash
# 1. Set environment variables
export NEYNAR_API_KEY="your-api-key"
export NEYNAR_SIGNER_UUID="your-signer-uuid"

# 2. Post your first cast
./skills/farcaster-skill/scripts/fc_cast.sh --text "Hello from OpenClaw!"

# 3. Read a channel feed
./skills/farcaster-skill/scripts/fc_feed.sh --channel "base" --limit 5
```

## Key Concepts

### Casts vs Posts
Farcaster calls posts "casts". A cast can contain:
- Text (max 320 characters)
- Embeds (URLs, images, other casts)
- Channel assignment
- Parent reference (for replies)

### FID (Farcaster ID)
Every Farcaster user has a unique numeric FID. Your agent needs one to interact.

### Signers
Neynar uses "signers" to authorize actions on behalf of your FID. Think of it as an API session that can post, like, and follow.

### Channels
Topic-based feeds similar to subreddits. Casts can be posted to channels for discovery.

## Common Patterns

### Agent as Curator
```bash
# Search for relevant content
./scripts/fc_search.sh --query "onchain art" --limit 20

# Read artist's recent casts
./scripts/fc_feed.sh --username "artist" --limit 10

# Quote-cast with commentary
./scripts/fc_cast.sh --text "Worth following" \
  --embed-cast "0xabc..." --embed-cast-fid 12345
```

### Agent as Responder
```bash
# Get conversation context before replying
./scripts/fc_get_conversation.sh "0xabc..." --parent-casts

# Reply with full context
./scripts/fc_cast.sh --text "Great point about..." --parent "0xabc..."
```

## Next Steps

Start with [01-neynar-setup.md](./01-neynar-setup.md) to configure your API access.
