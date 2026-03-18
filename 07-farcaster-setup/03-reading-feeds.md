# Reading Feeds

Learn to read user feeds, channel content, search casts, and explore Farcaster's data.

## Feed Types

The `fc_feed.sh` script supports multiple feed types:

| Option | Description |
|--------|-------------|
| `--fid FID` | User's casts by FID |
| `--username NAME` | User's casts by username |
| `--channel ID` | Channel feed |
| `--following` | Following feed (requires `--fid`) |
| `--thread HASH` | Thread/replies for a cast |

## User Feeds

### By Username

```bash
./skills/farcaster-skill/scripts/fc_feed.sh --username "vitalik" --limit 10
```

### By FID

```bash
./skills/farcaster-skill/scripts/fc_feed.sh --fid 3 --limit 5
```

### User's Popular Casts

```bash
./skills/farcaster-skill/scripts/fc_feed_user_popular.sh --username "dwr" --limit 10
```

### User's Replies and Recasts

```bash
./skills/farcaster-skill/scripts/fc_feed_user_recasts.sh --fid 3 --limit 10
```

## Channel Feeds

### Read a Channel

```bash
./skills/farcaster-skill/scripts/fc_feed.sh --channel "base" --limit 20
```

### List Channels

```bash
# Search channels
./skills/farcaster-skill/scripts/fc_channels.sh --search "art"

# Get channel details
./scripts/fc_channels.sh --id "cryptoart"

# Trending channels
./scripts/fc_channels.sh --trending --limit 10
```

## Search

### Basic Search

```bash
./skills/farcaster-skill/scripts/fc_search.sh --query "onchain art" --limit 20
```

### Filtered Search

```bash
# By author
./scripts/fc_search.sh --query "generative" --author-fid 3

# By channel
./scripts/fc_search.sh --query "auction" --channel "cryptoart"

# Combined
./scripts/fc_search.sh --query "base" --author-fid 3 --limit 5
```

## Algorithmic Feeds

### "For You" Feed

Personalized algorithmic feed:

```bash
./skills/farcaster-skill/scripts/fc_feed_for_you.sh --fid 874249 --limit 20
```

### Frames-Only Feed

Casts containing Farcaster Frames:

```bash
./skills/farcaster-skill/scripts/fc_feed_frames.sh --limit 10
```

### Filter by Embed URL

Find casts embedding a specific domain:

```bash
./skills/farcaster-skill/scripts/fc_feed_parent_urls.sh \
  --parent-url "github.com" --limit 20
```

## User Lookups

### By Username

```bash
./skills/farcaster-skill/scripts/fc_user.sh --username "alice"
```

### By FID

```bash
./scripts/fc_user.sh --fid 12345
```

### By Wallet Address

```bash
./scripts/fc_user.sh --address "0x1234..."
```

### Bulk Lookup

```bash
./scripts/fc_user.sh --fids "3,194,6131"
```

### Address to FID

```bash
./skills/farcaster-skill/scripts/fc_user_fid.sh --address "0xabc..."
```

### Check Active Status

```bash
./skills/farcaster-skill/scripts/fc_user_active.sh --fid 12345
```

### Users by Location

```bash
./skills/farcaster-skill/scripts/fc_user_location.sh --location "NYC" --limit 20
```

## Pagination

All feed scripts support pagination:

```bash
# First page
RESULT=$(./scripts/fc_feed.sh --channel "base" --limit 20)

# Extract cursor from response
CURSOR=$(echo "$RESULT" | jq -r '.next.cursor')

# Next page
./scripts/fc_feed.sh --channel "base" --limit 20 --cursor "$CURSOR"
```

## Conversation Threads

### Get Full Thread

```bash
./skills/farcaster-skill/scripts/fc_get_conversation.sh "0xabc123..."
```

### With Options

```bash
./scripts/fc_get_conversation.sh "0xabc123..." \
  --reply-depth 3 \
  --parent-casts \
  --viewer-fid 874249 \
  --limit 50
```

Options:
- `--reply-depth N`: How deep to fetch replies (default: 2)
- `--parent-casts`: Include parent casts in chronological order
- `--viewer-fid FID`: Respect mutes/blocks for this viewer
- `--limit N`: Max casts to fetch (1-50)

### Quick Context

Lighter alternative for simple context:

```bash
./skills/farcaster-skill/scripts/fc_reply_context.sh "0xabc123..."
```

## Quote Cast Lookup

See who quoted a specific cast:

```bash
./skills/farcaster-skill/scripts/fc_cast_quotes.sh \
  --identifier "0xabc123..." \
  --type hash \
  --limit 50
```

## Output Processing

All scripts output JSON. Use `jq` to extract what you need:

```bash
# Get just the cast text
./scripts/fc_feed.sh --username "dwr" --limit 5 | \
  jq -r '.casts[].text'

# Get cast hashes
./scripts/fc_feed.sh --channel "base" --limit 10 | \
  jq -r '.casts[].hash'

# Get author usernames
./scripts/fc_search.sh --query "crypto" --limit 10 | \
  jq -r '.casts[].author.username'
```

## Example: Content Discovery Agent

```bash
#!/bin/bash
# Agent discovers and analyzes trending content

SKILL_PATH="./skills/farcaster-skill/scripts"

# 1. Get trending channels
CHANNELS=$($SKILL_PATH/fc_channels.sh --trending --limit 5)

# 2. For each channel, get top casts
for CHANNEL in $(echo "$CHANNELS" | jq -r '.channels[].id'); do
  echo "=== Channel: $CHANNEL ==="
  
  $SKILL_PATH/fc_feed.sh --channel "$CHANNEL" --limit 3 | \
    jq -r '.casts[] | "\(.author.username): \(.text)"'
done

# 3. Search for specific topics
echo "=== Topic: onchain art ==="
$SKILL_PATH/fc_search.sh --query "onchain art" --limit 5 | \
  jq -r '.casts[] | "[\(.hash)] \(.author.username): \(.text)"'
```

## Example: Research Agent

```bash
#!/bin/bash
# Research a specific user before engaging

USERNAME="$1"
SKILL_PATH="./skills/farcaster-skill/scripts"

echo "=== Profile ==="
$SKILL_PATH/fc_user.sh --username "$USERNAME" | \
  jq '{fid: .user.fid, bio: .user.profile.bio.text, followers: .user.follower_count}'

echo "=== Recent Casts ==="
$SKILL_PATH/fc_feed.sh --username "$USERNAME" --limit 5 | \
  jq -r '.casts[] | "- \(.text)"'

echo "=== Popular Casts ==="
$SKILL_PATH/fc_feed_user_popular.sh --username "$USERNAME" --limit 3 | \
  jq -r '.casts[] | "- [\(.reactions.likes_count) likes] \(.text)"'
```

## Rate Limiting Considerations

- Free tier: 1,000 calls/day
- Batch operations count as multiple calls
- Cache results when possible
- Use pagination thoughtfully

## Next Steps

Learn engagement patterns (reactions, replies, conversation flow) in [04-engagement.md](./04-engagement.md).
