# Neynar Setup

Neynar is the primary API provider for Farcaster. This guide covers getting your API credentials and configuring them for OpenClaw agents.

## Getting Your API Key

1. **Create a Neynar account** at [neynar.com](https://neynar.com)
2. **Navigate to the dashboard** and create a new app
3. **Copy your API key** from the app settings

The free tier includes:
- 1,000 API calls/day
- Read access to all public data
- Write access (with signer)

## Creating a Signer

A signer authorizes your app to take actions on behalf of a Farcaster account.

### Option 1: Neynar Managed Signer (Recommended)

Use Neynar's hosted signer flow:

```bash
# Request a signer
curl -X POST "https://api.neynar.com/v2/farcaster/signer" \
  -H "api_key: YOUR_API_KEY" \
  -H "Content-Type: application/json"
```

Response includes a `signer_uuid` and approval URL. The user must approve the signer in Warpcast.

### Option 2: Self-Hosted Signer

For full control, generate your own Ed25519 keypair and register it. See Neynar docs for details.

## Environment Configuration

### Basic Setup

Set these environment variables:

```bash
export NEYNAR_API_KEY="your-api-key"
export NEYNAR_SIGNER_UUID="your-signer-uuid"
```

### Persistent Configuration

Add to your shell profile (`~/.bashrc`, `~/.zshrc`):

```bash
# Farcaster / Neynar
export NEYNAR_API_KEY="neynar_abc123..."
export NEYNAR_SIGNER_UUID="signer_xyz789..."
```

### JSON File Method

Store credentials in a JSON file:

```json
{
  "apiKey": "neynar_abc123...",
  "signerUuid": "signer_xyz789..."
}
```

Source them at runtime:

```bash
eval $(jq -r '"export NEYNAR_API_KEY=\(.apiKey)\nexport NEYNAR_SIGNER_UUID=\(.signerUuid)"' /path/to/neynar.json)
```

### Per-Command Override

All scripts accept `--api-key` and `--signer` flags:

```bash
./scripts/fc_cast.sh --text "Hello" \
  --api-key "neynar_abc..." \
  --signer "signer_xyz..."
```

## OpenClaw Gateway Configuration

In your `config.yaml`, add Farcaster credentials:

```yaml
gateway:
  name: my-agent
  
env:
  NEYNAR_API_KEY: ${NEYNAR_API_KEY}
  NEYNAR_SIGNER_UUID: ${NEYNAR_SIGNER_UUID}

skills:
  - path: ./skills/farcaster-skill
```

## Verifying Your Setup

### Test Read Access

```bash
# Look up a user (no signer needed)
./skills/farcaster-skill/scripts/fc_user.sh --username "dwr"
```

Expected output: JSON with user profile data.

### Test Write Access

```bash
# Post a test cast (requires signer)
./skills/farcaster-skill/scripts/fc_cast.sh --text "Testing OpenClaw integration"
```

Expected output: JSON with the new cast's hash.

## Troubleshooting

### "NEYNAR_API_KEY not set"

Environment variable missing. Either:
- Export it: `export NEYNAR_API_KEY="..."`
- Pass via flag: `--api-key "..."`

### "NEYNAR_SIGNER_UUID not set"

Required for write operations. Either:
- Export it: `export NEYNAR_SIGNER_UUID="..."`
- Pass via flag: `--signer "..."`

### "Signer not approved"

The signer hasn't been approved in Warpcast yet. Use the approval URL from signer creation.

### "Rate limited"

Free tier has 1,000 calls/day. Check your usage in the Neynar dashboard or upgrade your plan.

## Security Best Practices

1. **Never commit credentials** to version control
2. **Use environment variables** or secure secret management
3. **Rotate signers** if compromised
4. **Scope access** - create separate signers for different agents

## Your Agent's FID

Your agent needs a Farcaster account with its own FID. Options:

1. **Dedicated bot account** - Create a new Farcaster account for your agent
2. **Your personal account** - Use your own FID (be careful with automated posting)

To find an account's FID:

```bash
./skills/farcaster-skill/scripts/fc_user.sh --username "yourusername"
# Look for "fid" in the response
```

## Next Steps

With credentials configured, proceed to [02-posting-casts.md](./02-posting-casts.md) to start posting.
