# OpenClaw WhatsApp Integration Guide

## How It Works

OpenClaw connects to WhatsApp using the **Baileys library** (WhatsApp Web protocol). It works exactly like linking WhatsApp Web — you scan a QR code to authenticate, and OpenClaw maintains the session.

- Uses WhatsApp Web API via WebSocket (not WhatsApp Business API)
- Credentials stored locally in `~/.openclaw/credentials/whatsapp/`
- Auto-reconnects if connection drops
- Supports voice messages, media, reactions, and polls

## Prerequisites

- OpenClaw installed (`openclaw --version`)
- Node.js v22+ (**do not use Bun** — causes Baileys crashes)
- WhatsApp account on your phone
- Phone connected to internet

## Step 1: Enable WhatsApp in OpenClaw

### Via Web UI (Easiest)

1. Open `http://localhost:18789/?token=YOUR_TOKEN`
2. Go to **Channels** > **Add Channel** > **WhatsApp**
3. Follow the on-screen instructions

### Via Config File

Add to `~/.openclaw/openclaw.json`:

```json
{
  "plugins": {
    "entries": {
      "slack": {
        "enabled": true
      },
      "whatsapp": {
        "enabled": true
      }
    }
  }
}
```

## Step 2: Link Your WhatsApp Account

```bash
# Generate QR code for linking
openclaw channels login --channel whatsapp
```

A QR code will appear in your terminal.

**On your phone:**

1. Open WhatsApp
2. Go to **Settings** > **Linked Devices**
3. Tap **Link a Device**
4. Point your camera at the QR code in the terminal
5. Wait for "WhatsApp linked successfully" message

**Important:**
- QR codes expire after ~60 seconds — re-run the command if it expires
- WhatsApp allows a maximum of 4 linked devices
- Your phone must stay connected to the internet for the initial link

## Step 3: Start the Gateway

```bash
openclaw gateway
```

WhatsApp should now show as connected. Send yourself a test message.

## Step 4: Choose Your Setup Mode

### Option A: Self-Chat Mode (Use Your Own Number)

Chat with yourself to access the AI assistant. Best for personal use.

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "selfChatMode": true,
      "dmPolicy": "allowlist",
      "allowFrom": ["+61412345678"]
    }
  }
}
```

**Pros:** No extra phone number needed, simple setup
**Cons:** AI messages mix with your personal chat history

### Option B: Dedicated Number (Recommended)

Use a separate phone number for the bot. Share it with others who need access.

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "dmPolicy": "pairing",
      "allowFrom": []
    }
  }
}
```

**Pros:** Clean separation, can share with team, professional
**Cons:** Requires an extra phone number/SIM

## Security Configuration

### DM Policies

| Policy | Behavior | When to Use |
|--------|----------|-------------|
| `pairing` (default) | New contacts must be approved via pairing code | Production — most secure |
| `allowlist` | Only specified phone numbers can message | Known contacts only |
| `open` | Anyone can message the bot | Testing only — NOT recommended |
| `disabled` | All DMs blocked | Groups-only mode |

### Pairing Mode (Recommended)

When someone messages the bot for the first time, they receive a pairing code. You must approve it:

```bash
# List pending pairing requests
openclaw pairing list whatsapp

# Approve a request
openclaw pairing approve whatsapp <CODE>
```

Pairing requests:
- Expire after 1 hour
- Limited to 3 pending requests per channel
- Must be approved manually

### Allowlist Mode

Restrict access to specific phone numbers (E.164 format):

```json
{
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "allowFrom": [
        "+61412345678",
        "+15551234567"
      ]
    }
  }
}
```

### Group Chat Security

```json
{
  "channels": {
    "whatsapp": {
      "groupPolicy": "allowlist",
      "groups": {
        "120363012345678901@g.us": {
          "enabled": true,
          "requireMention": true,
          "groupAllowFrom": ["+61412345678"]
        }
      }
    }
  }
}
```

Key settings:
- **`requireMention: true`** — Bot only responds when @mentioned in groups
- **`groupAllowFrom`** — Only these numbers can trigger the bot in the group
- **`groupPolicy: "allowlist"`** — Only listed groups are allowed

## Supported Features

### Message Types

| Type | Supported | Notes |
|------|-----------|-------|
| Text messages | Yes | Auto-splits at 4000 chars |
| Voice messages | Yes | Auto-transcribed to text |
| Images | Yes | JPG, PNG, GIF, WebP |
| Videos | Yes | MP4, MOV |
| Documents | Yes | PDF, DOC, TXT, etc. |
| Stickers | Yes | WhatsApp stickers |
| Location | Yes | GPS coordinates |
| Reactions | Yes | Full emoji support |
| Polls | Yes | Up to 12 options |

### Voice Messages

OpenClaw automatically transcribes voice messages to text, processes them with the AI model, and responds. If you have the `openai-whisper` skill enabled, transcription runs locally (no API key needed).

## Advanced Configuration

### Multiple WhatsApp Accounts

```json
{
  "channels": {
    "whatsapp": {
      "accounts": {
        "personal": {
          "enabled": true,
          "name": "Personal",
          "dmPolicy": "allowlist",
          "allowFrom": ["+61412345678"]
        },
        "work": {
          "enabled": true,
          "name": "Work",
          "dmPolicy": "pairing"
        }
      }
    }
  }
}
```

Link each account separately:

```bash
openclaw channels login --channel whatsapp --account personal
openclaw channels login --channel whatsapp --account work
```

### Media Handling

```json
{
  "channels": {
    "whatsapp": {
      "config": {
        "mediaMaxMb": 10
      }
    }
  }
}
```

## Troubleshooting

### QR Code Won't Appear

```bash
# Restart gateway and try again
openclaw gateway restart
openclaw channels login --channel whatsapp
```

### Connection Keeps Dropping

1. Check your phone's internet connection
2. Open WhatsApp on your phone (keep it active)
3. Check linked devices list — unlink and re-link if needed
4. Clear credentials and re-authenticate:

```bash
openclaw channels logout --channel whatsapp
rm -rf ~/.openclaw/credentials/whatsapp/
openclaw channels login --channel whatsapp
```

### "Not Linked" After Restart

WhatsApp may unlink after extended inactivity:

1. Check phone's **WhatsApp > Linked Devices**
2. If OpenClaw is not listed, re-scan QR code
3. Re-run: `openclaw channels login --channel whatsapp`

### Bot Not Responding to Messages

1. Check DM policy — is the sender approved?
   ```bash
   openclaw pairing list whatsapp
   ```
2. Verify gateway is running: `openclaw gateway status`
3. Check logs for errors: `tail -f /tmp/openclaw-gateway.log`
4. Ensure Bedrock model is accessible: `aws bedrock-runtime invoke-model --help`

### Multiple Device Limit Reached

WhatsApp allows max 4 linked devices. To free a slot:
1. Open WhatsApp on phone
2. Go to **Linked Devices**
3. Tap a device > **Log Out**
4. Re-link OpenClaw

### Crashes on Bun Runtime

OpenClaw with WhatsApp **must** run on Node.js, not Bun:

```bash
# Verify you're using Node
node --version  # Should show v22+

# If using Bun, switch to Node
export OPENCLAW_RUNTIME=node
openclaw gateway
```

## Security Best Practices

1. **Always use pairing mode** (`dmPolicy: "pairing"`) in production
2. **Enable `requireMention`** for all group chats
3. **Review pending pairing requests** weekly
4. **Protect credentials directory**:
   ```bash
   chmod 700 ~/.openclaw/credentials/whatsapp/
   ```
5. **Use a dedicated number** if sharing with others
6. **Monitor logs** for unauthorized message attempts
7. **Keep phone connected** — WhatsApp may unlink after prolonged disconnection

## File Locations

| File | Location |
|------|----------|
| Config | `~/.openclaw/openclaw.json` |
| WhatsApp credentials | `~/.openclaw/credentials/whatsapp/` |
| Gateway logs | `/tmp/openclaw-gateway.log` |
| Session data | `~/.openclaw/agents/*/sessions/` |
