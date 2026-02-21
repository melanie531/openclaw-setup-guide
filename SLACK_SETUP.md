# OpenClaw Slack Integration Guide

## Overview

OpenClaw supports two Slack connection modes:
- **Socket Mode** (Recommended): WebSocket connection, no public endpoints needed
- **HTTP Events API Mode**: Requires public webhook endpoint

## Step 1: Create a Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App** > **From scratch**
3. Name: "OpenClaw Assistant" (or your preferred name)
4. Select your target Slack workspace
5. Click **Create App**

## Step 2: Configure OAuth Scopes

Navigate to **OAuth & Permissions** > **Bot Token Scopes** and add:

### Required Scopes

| Scope | Purpose |
|-------|---------|
| `chat:write` | Send messages as the bot |
| `channels:history` | Read public channel messages |
| `channels:read` | View basic channel info |
| `groups:history` | Read private channel messages |
| `im:history` | Read direct messages |
| `mpim:history` | Read group DMs |
| `app_mentions:read` | Respond when mentioned |
| `users:read` | Read user information |
| `reactions:read` | Read message reactions |
| `reactions:write` | Add/remove reactions |
| `files:read` | Read file information |
| `files:write` | Upload files |

### Optional Scopes

| Scope | Purpose |
|-------|---------|
| `pins:read` | Read pinned messages |
| `pins:write` | Pin/unpin messages |
| `emoji:read` | View custom emojis |
| `commands` | Handle slash commands |
| `assistant:write` | Use assistant features |

## Step 3: Install App to Workspace

1. In **OAuth & Permissions**, click **Install to Workspace**
2. Review permissions and click **Allow**
3. Copy the **Bot User OAuth Token** (starts with `xoxb-`)

**Keep this token secret.**

## Step 4: Enable Socket Mode (Recommended)

### Create App Token

1. Go to **Basic Information** > **App-Level Tokens**
2. Click **Generate Token and Scopes**
3. Name: "OpenClaw Connection"
4. Add scope: `connections:write`
5. Click **Generate**
6. Copy the App-Level Token (starts with `xapp-`)

### Enable Socket Mode

1. Navigate to **Socket Mode** in the sidebar
2. Toggle Socket Mode **On**

### Subscribe to Events

1. Navigate to **Event Subscriptions**
2. Toggle **Enable Events** to On
3. Under **Subscribe to bot events**, add:

| Event | Description |
|-------|-------------|
| `app_mention` | Bot mentioned in channel |
| `message.channels` | Messages in public channels |
| `message.groups` | Messages in private channels |
| `message.im` | Direct messages to bot |
| `message.mpim` | Group DM messages |
| `reaction_added` | Reaction added to message |
| `reaction_removed` | Reaction removed |

4. Click **Save Changes**

## Step 5: Configure OpenClaw

### Option A: Via Web UI

1. Open OpenClaw Web UI at `http://localhost:18789/?token=YOUR_TOKEN`
2. Go to **Channels** > **Add Channel** > **Slack**
3. Enter your Bot Token and App Token
4. Save

### Option B: Via Config File

Add to `~/.openclaw/openclaw.json`:

```json
{
  "plugins": {
    "entries": {
      "slack": {
        "enabled": true
      }
    }
  }
}
```

Set tokens via environment variables (recommended — avoids plaintext in config):

```bash
export SLACK_BOT_TOKEN="xoxb-your-bot-token"
export SLACK_APP_TOKEN="xapp-your-app-token"
```

### Option C: Full Config (Advanced)

```json
{
  "channels": {
    "slack": {
      "enabled": true,
      "mode": "socket",
      "appToken": "xapp-your-app-token",
      "botToken": "xoxb-your-bot-token",
      "config": {
        "groupPolicy": "allowlist",
        "channels": {
          "general": {
            "enabled": true,
            "requireMention": true
          },
          "openclaw-assistant": {
            "enabled": true,
            "requireMention": false
          }
        }
      },
      "dm": {
        "policy": "pairing",
        "allowFrom": []
      }
    }
  }
}
```

## Step 6: Start and Test

```bash
# Start the gateway
openclaw gateway
```

### Test the Connection

1. Invite the bot to a channel:
   ```
   /invite @OpenClaw
   ```
2. Mention the bot:
   ```
   @OpenClaw Hello!
   ```
3. Send a DM to the bot directly

## Security Configuration

### DM Policies

| Policy | Behavior | Recommended For |
|--------|----------|----------------|
| `pairing` | Requires approval code for new contacts | Production (default) |
| `allowlist` | Only specified users can message | Restricted access |
| `open` | Anyone can message | Testing only |
| `disabled` | No DMs accepted | Channel-only use |

### Channel Policies

- **`requireMention: true`** — Bot only responds when @mentioned (recommended for public channels)
- **`groupPolicy: "allowlist"`** — Restrict which channels the bot operates in

### Token Security

- Use environment variables instead of hardcoding tokens in config
- Rotate tokens periodically via Slack API Console
- Never commit tokens to version control
- Monitor Slack audit logs for unauthorized access

## Troubleshooting

### Bot Not Responding

1. Verify tokens are correct (`xoxb-` for bot, `xapp-` for app)
2. Check bot is invited to the channel (`/invite @OpenClaw`)
3. Ensure all required scopes are granted
4. Reinstall app after adding new scopes
5. Check OpenClaw logs: `tail -f /tmp/openclaw-gateway.log`

### Socket Mode Connection Failed

1. Verify App Token has `connections:write` scope
2. Check Socket Mode is toggled On in Slack API Console
3. Restart gateway: `openclaw gateway`

### Permission Errors

1. Go to **OAuth & Permissions** and verify scopes
2. Click **Reinstall to Workspace** after adding scopes
3. Re-copy the new Bot Token (it regenerates on reinstall)

### Rate Limiting

Slack rate limits API calls. If you see rate limit errors:
- Reduce message frequency
- Use threading to consolidate responses
- Check Slack's [rate limit documentation](https://api.slack.com/docs/rate-limits)
