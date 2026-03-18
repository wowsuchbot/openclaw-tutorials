# Engagement Workflows

Build meaningful interactions on Farcaster through likes, recasts, thoughtful replies, and conversation awareness.

## Reactions

### Like a Cast

```bash
./skills/farcaster-skill/scripts/fc_react.sh --like "0xabc123..."
```

### Recast (Re-share)

```bash
./skills/farcaster-skill/scripts/fc_react.sh --recast "0xabc123..."
```

### Unlike / Un-recast

```bash
./scripts/fc_react.sh --like "0xabc123..." --undo
./scripts/fc_react.sh --recast "0xabc123..." --undo
```

## Reply Workflow

A good reply requires context. Never reply blind.

### Step 1: Get Conversation Context

```bash
# Full conversation with parent casts
./skills/farcaster-skill/scripts/fc_get_conversation.sh "0xabc123..." \
  --parent-casts \
  --reply-depth 2
```

This returns:
- The target cast
- Parent casts (what it's replying to)
- Child casts (existing replies)

### Step 2: Analyze Context

Your agent should understand:
- What's the original topic?
- What point is the cast making?
- What have others already said?
- Does your agent have something valuable to add?

### Step 3: Formulate Response

Key principles:
- Add value, don't just agree
- Be conversational
- No username prefix in the text
- Keep it concise (320 char limit)

### Step 4: Post Reply

```bash
./skills/farcaster-skill/scripts/fc_cast.sh \
  --text "Interesting approach. One thing to consider is..." \
  --parent "0xabc123..."
```

### Complete Example

```bash
#!/bin/bash
SKILL_PATH="./skills/farcaster-skill/scripts"
CAST_HASH="0xabc123..."

# Get context
CONTEXT=$($SKILL_PATH/fc_get_conversation.sh "$CAST_HASH" --parent-casts)

# Extract relevant info for agent analysis
CAST_TEXT=$(echo "$CONTEXT" | jq -r '.conversation.cast.text')
AUTHOR=$(echo "$CONTEXT" | jq -r '.conversation.cast.author.username')
PARENT_CASTS=$(echo "$CONTEXT" | jq -r '.conversation.chronological_parent_casts[]?.text')

# Agent formulates reply (your logic here)
REPLY="Your perspective on noise functions is spot on. The key insight is how they create organic variation at different scales."

# Post reply
$SKILL_PATH/fc_cast.sh --text "$REPLY" --parent "$CAST_HASH"
```

## Understanding Quote-Casts in Responses

When analyzing a conversation, understand the difference:

### Quoted Cast (Embedded)

Found in the `embeds` array:

```json
{
  "embeds": [
    {
      "cast": {
        "hash": "0xdef456...",
        "text": "The original embedded cast text",
        "author": { "username": "alice" }
      }
    }
  ]
}
```

### NOT a Quote (Just Reply Text)

Text in `chronological_parent_casts[].text` is NOT an embedded quote - it's just the parent cast being replied to.

**Common mistake:** Treating reply parent text as a "quoted" cast.

## When to Engage

### React (Like/Recast) When:
- Content is genuinely valuable
- You want to amplify to your followers
- Supporting a creator you appreciate

### Reply When:
- You have something unique to add
- You can answer a question
- You have relevant expertise
- Correcting misinformation (respectfully)

### Stay Silent When:
- Someone already made your point
- Your reply would just be "agreed" or "nice"
- You don't have enough context
- The conversation is personal/private in nature

## Engagement Etiquette for Agents

### Do:
- Add substantive value
- Show genuine understanding of context
- Be conversational and natural
- Respect the flow of conversation

### Don't:
- Spam reactions on everything
- Reply with generic AI-sounding text
- Quote-cast just to get attention
- Batch-reply to multiple casts rapidly
- Use hashtags (Farcaster culture)

## Multi-Step Engagement Flow

```bash
#!/bin/bash
# Agent discovers content, analyzes, and engages appropriately

SKILL_PATH="./skills/farcaster-skill/scripts"
CHANNEL="cryptoart"

# 1. Discover interesting content
FEED=$($SKILL_PATH/fc_feed.sh --channel "$CHANNEL" --limit 10)

# 2. For each cast, decide engagement level
for CAST in $(echo "$FEED" | jq -c '.casts[]'); do
  HASH=$(echo "$CAST" | jq -r '.hash')
  TEXT=$(echo "$CAST" | jq -r '.text')
  LIKES=$(echo "$CAST" | jq -r '.reactions.likes_count')
  
  # Skip already-popular casts
  if [ "$LIKES" -gt 100 ]; then
    continue
  fi
  
  # Get full context before deciding
  CONTEXT=$($SKILL_PATH/fc_get_conversation.sh "$HASH" --parent-casts)
  
  # Agent analysis here - decide action:
  # - LIKE: Good content, nothing to add
  # - REPLY: Can add unique value
  # - QUOTE: Worth amplifying with commentary
  # - SKIP: Not relevant or already discussed
  
  ACTION="like"  # Agent decides
  
  case $ACTION in
    "like")
      $SKILL_PATH/fc_react.sh --like "$HASH"
      ;;
    "reply")
      $SKILL_PATH/fc_cast.sh --text "$REPLY_TEXT" --parent "$HASH"
      ;;
    "quote")
      $SKILL_PATH/fc_cast.sh --text "$QUOTE_TEXT" \
        --embed-cast "$HASH" --embed-cast-fid "$AUTHOR_FID"
      ;;
  esac
  
  # Rate limiting - don't hammer the API
  sleep 2
done
```

## Notification Processing

When your agent receives mentions or replies:

```bash
#!/bin/bash
# Process incoming notifications

SKILL_PATH="./skills/farcaster-skill/scripts"
MY_FID="874249"

# Search for recent mentions (since Neynar doesn't have native notifications endpoint)
MENTIONS=$($SKILL_PATH/fc_search.sh --query "@myagent" --limit 20)

for CAST in $(echo "$MENTIONS" | jq -c '.casts[]'); do
  HASH=$(echo "$CAST" | jq -r '.hash')
  
  # Skip if we've already replied (check your own recent casts)
  # ... implementation depends on your tracking system
  
  # Get full context
  CONTEXT=$($SKILL_PATH/fc_get_conversation.sh "$HASH" --parent-casts)
  
  # Formulate and post reply
  # ... agent logic
  
  # One reply at a time, with delay
  sleep 3
done
```

## Conversation Threading

When replying in a thread, be aware of your position:

```bash
# Check if cast is a reply to something
PARENT=$(echo "$CAST" | jq -r '.parent_hash // empty')

if [ -n "$PARENT" ]; then
  # This is a reply - get full thread context
  THREAD=$($SKILL_PATH/fc_get_conversation.sh "$PARENT" --reply-depth 3)
fi
```

## Access Control Integration

From Module 6, remember access control in group contexts:

```bash
# Check if requester is authorized for actions
AUTHORIZED_FID="4905"  # Owner's FID
REQUESTER_FID="$1"

if [ "$REQUESTER_FID" != "$AUTHORIZED_FID" ]; then
  echo "Conversational reply only - no tool execution"
  # Agent responds conversationally but doesn't take actions
  exit 0
fi

# Authorized - proceed with engagement
```

## Error Handling

```bash
# Wrap engagement in error handling
engage_with_cast() {
  local HASH="$1"
  local ACTION="$2"
  
  case $ACTION in
    "like")
      RESULT=$($SKILL_PATH/fc_react.sh --like "$HASH" 2>&1)
      ;;
    "reply")
      RESULT=$($SKILL_PATH/fc_cast.sh --text "$3" --parent "$HASH" 2>&1)
      ;;
  esac
  
  # Check for common errors
  if echo "$RESULT" | grep -q "Safeguard abort"; then
    echo "Rate limited - waiting..."
    sleep 30
    return 1
  fi
  
  if echo "$RESULT" | grep -q "already liked"; then
    echo "Already engaged with this cast"
    return 0
  fi
  
  return 0
}
```

## Metrics and Logging

Track engagement for analysis:

```bash
# Log engagement for later analysis
log_engagement() {
  local TIMESTAMP=$(date -Iseconds)
  local CAST_HASH="$1"
  local ACTION="$2"
  local RESULT="$3"
  
  echo "$TIMESTAMP,$CAST_HASH,$ACTION,$RESULT" >> engagement_log.csv
}
```

## Summary

Effective Farcaster engagement:

1. **Always get context** before replying
2. **Add value** - don't just agree or react automatically
3. **Respect rate limits** - sequential, not parallel
4. **Match the culture** - conversational, no hashtags
5. **Track your activity** - avoid duplicate engagement

The goal is genuine participation that builds reputation, not volume metrics.
