# OpenClaw Personal Assistant: Comprehensive Setup Guide

## 0. Installation & Bedrock Setup (macOS)

This section documents the full installation process for running OpenClaw locally on macOS with Amazon Bedrock as the AI backend — no Anthropic/OpenAI API keys needed.

### Prerequisites

- macOS (Apple Silicon or Intel)
- Node.js >= 22
- AWS account with Bedrock model access enabled
- AWS CLI v2 configured with credentials

### Step 1: Install Node.js

```bash
node --version
# Requires v22+. If not installed:
brew install node
```

### Step 2: Install AWS CLI v2

```bash
# Download the macOS installer
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "/tmp/AWSCLIV2.pkg"
sudo installer -pkg /tmp/AWSCLIV2.pkg -target /

# Verify
aws --version

# Configure credentials
aws configure
# Enter: Access Key ID, Secret Access Key, Region (e.g. us-west-2), Output format (json)
```

### Step 3: Enable Bedrock Models

Before OpenClaw can use Bedrock, you must enable model access in the AWS Console:

1. Go to [Amazon Bedrock Console](https://console.aws.amazon.com/bedrock/)
2. Navigate to **Model access** in the left sidebar
3. Click **Manage model access**
4. Enable the models you want (e.g., Claude Opus 4.6, Claude Sonnet 4.5, Nova 2 Lite)
5. Wait for access to be granted (usually instant for most models)

### Step 4: Install OpenClaw

```bash
npm install -g openclaw@latest

# Verify
openclaw --version
```

### Step 5: Create OpenClaw Config Directory

```bash
mkdir -p ~/.openclaw
```

### Step 6: Generate Gateway Token

```bash
GATEWAY_TOKEN=$(openssl rand -hex 24)
echo "$GATEWAY_TOKEN" > ~/.openclaw/gateway_token.txt
echo "Your token: $GATEWAY_TOKEN"
```

### Step 7: Create Bedrock Config

Create `~/.openclaw/openclaw.json` with the following content. Replace `us-west-2` with your AWS region:

```json
{
  "gateway": {
    "mode": "local",
    "port": 18789,
    "bind": "loopback",
    "controlUi": {
      "enabled": true,
      "allowInsecureAuth": true
    },
    "auth": {
      "mode": "token",
      "token": "YOUR_GATEWAY_TOKEN_HERE"
    }
  },
  "models": {
    "providers": {
      "amazon-bedrock": {
        "baseUrl": "https://bedrock-runtime.us-west-2.amazonaws.com",
        "api": "bedrock-converse-stream",
        "auth": "aws-sdk",
        "models": [
          {
            "id": "global.anthropic.claude-opus-4-6-v1",
            "name": "Claude Opus 4.6",
            "input": ["text", "image"],
            "contextWindow": 200000,
            "maxTokens": 8192
          },
          {
            "id": "global.anthropic.claude-sonnet-4-5-20250929-v1:0",
            "name": "Claude Sonnet 4.5",
            "input": ["text", "image"],
            "contextWindow": 200000,
            "maxTokens": 8192
          },
          {
            "id": "global.amazon.nova-2-lite-v1:0",
            "name": "Nova 2 Lite",
            "input": ["text", "image"],
            "contextWindow": 300000,
            "maxTokens": 5120
          },
          {
            "id": "us.amazon.nova-pro-v1:0",
            "name": "Nova Pro",
            "input": ["text", "image"],
            "contextWindow": 300000,
            "maxTokens": 5120
          },
          {
            "id": "global.anthropic.claude-haiku-4-5-20251001-v1:0",
            "name": "Claude Haiku 4.5",
            "input": ["text", "image"],
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "amazon-bedrock/global.anthropic.claude-opus-4-6-v1"
      }
    }
  }
}
```

Key config explanation:
- `"auth": "aws-sdk"` — uses your local `~/.aws/credentials` (IAM auth, no plaintext API keys)
- `"bind": "loopback"` — gateway only accessible on localhost (secure)
- `"api": "bedrock-converse-stream"` — uses Bedrock's Converse streaming API
- 5 models configured for different cost/capability tradeoffs

### Step 8: Lock Down Permissions

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
chmod 600 ~/.openclaw/gateway_token.txt
```

### Step 9: Start OpenClaw

```bash
openclaw gateway
```

### Step 10: Access the Web UI

Open in browser:

```
http://localhost:18789/?token=YOUR_GATEWAY_TOKEN_HERE
```

### Step 11: Connect Channels (Optional)

In the Web UI, go to **Channels** to connect messaging platforms:
- **Slack**: Add Slack channel with bot token
- **WhatsApp**: Scan QR code
- **Telegram**: Add bot token from @BotFather

### Why Bedrock Instead of Direct API Keys?

| Direct API Keys | Bedrock (This Setup) |
|----------------|---------------------|
| Plaintext keys in config files | IAM credentials (auto-rotated) |
| Single provider lock-in | 8+ models from multiple providers |
| No audit trail | CloudTrail logs every API call |
| Manual key rotation | AWS handles credential lifecycle |
| Per-provider billing | Unified AWS billing |

### Available Bedrock Models

| Model | ID | Best For | Cost (in/out per 1M tokens) |
|-------|-----|---------|---------------------------|
| Nova 2 Lite | `global.amazon.nova-2-lite-v1:0` | Everyday tasks (cheapest) | $0.30 / $2.50 |
| Nova Pro | `us.amazon.nova-pro-v1:0` | Balanced performance | $0.80 / $3.20 |
| Claude Haiku 4.5 | `global.anthropic.claude-haiku-4-5-20251001-v1:0` | Fast simple tasks | $1.00 / $5.00 |
| Claude Sonnet 4.5 | `global.anthropic.claude-sonnet-4-5-20250929-v1:0` | Daily driver | $3.00 / $15.00 |
| Claude Opus 4.6 | `global.anthropic.claude-opus-4-6-v1` | Complex reasoning | Highest |

### Reference

This setup is based on [aws-samples/sample-OpenClaw-on-AWS-with-Bedrock](https://github.com/aws-samples/sample-OpenClaw-on-AWS-with-Bedrock), adapted for local macOS use instead of EC2 deployment.

---

## 1. Immediate Security Hardening (DO THIS FIRST)

### Critical: CVE-2026-25253 — Remote Code Execution (CVSS 8.8)

Patched in version 2026.1.29+. Current version 2026.2.19-2 is safe. Always keep updated.

Over 40,000+ exposed OpenClaw instances were discovered on the public internet due to misconfigured default bindings. Hundreds had no authentication, exposing API keys, Telegram tokens, and Slack credentials.

### Lock Down Config

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
chmod 600 ~/.openclaw/gateway_token.txt
```

### Current Security Settings (Already Configured)

- `bind: loopback` — gateway only accessible on localhost
- `auth.mode: token` — requires token to access UI
- Bedrock `auth: aws-sdk` — no API keys in plaintext, uses IAM credentials

---

## 2. Architecture Overview

```
User Device (Mac)
  |
  v
Messaging Platform (Slack / WhatsApp / Telegram / Discord)
  |
  v
OpenClaw Gateway (localhost:18789)
  ├── Skills (automation capabilities)
  ├── Agents (AI model routing)
  └── Plugins (channel connectors)
  |
  v
Amazon Bedrock (Claude Opus 4.6 / Sonnet 4.5 / Nova / Haiku)
  via AWS IAM auth (no API keys)
```

---

## 3. Skills Analysis & Recommendations

### Tier 1: Essential (Low Risk, High Value) — Install These

| Skill | What It Does | Security Risk | Why You Need It |
|-------|-------------|---------------|-----------------|
| **weather** | Weather via wttr.in | None — public API | Universal daily utility |
| **apple-notes** | Manage Apple Notes via CLI | Low — local data only | macOS native note automation |
| **apple-reminders** | Manage Reminders app | Low — local data only | Task/reminder automation |
| **summarize** | Document summarization | Low — local processing | Summarize PDFs, articles, docs |
| **openai-whisper** | Local speech-to-text | Low — runs locally, no API key | Privacy-focused transcription |

Enable via Web UI (Settings > Skills) or add to `~/.openclaw/openclaw.json`:

```json
"skills": {
  "entries": {
    "weather": { "enabled": true },
    "apple-notes": { "enabled": true },
    "apple-reminders": { "enabled": true },
    "summarize": { "enabled": true },
    "openai-whisper": { "enabled": true }
  }
}
```

### Tier 2: Productivity (Medium Risk) — Install with Caution

| Skill | What It Does | Security Risk | Mitigation |
|-------|-------------|---------------|------------|
| **slack** | React, send, manage Slack messages | Medium — workspace data access | Use scoped bot tokens, limit channel access |
| **github** | PR review, issue tracking, CI monitoring | Medium — code repo access | Use read-only personal access tokens |
| **openai-image-gen** | Image generation (DALL-E) | Low-Medium — API costs | Set spending limits, monitor usage |
| **notion** | Manage Notion workspaces | Medium — document access | Use integration tokens with limited page access |
| **spotify-player** | Control Spotify playback | Low — entertainment only | Minimal risk |

Important: For Slack, ensure bot token has minimal scopes:

- `chat:write`, `channels:history`, `channels:read` — sufficient for basic use
- Avoid `admin.*` scopes

### Tier 3: Advanced (High Risk) — Expert Users Only

| Skill | What It Does | Security Risk | Recommendation |
|-------|-------------|---------------|----------------|
| **1password** | Credential management via CLI | HIGH — full credential access | Only if you understand 1Password CLI security model |
| **voice-call** | Phone calls via Twilio/Telnyx | HIGH — phone system access | Dedicated environment, spending limits |
| **browser automation** | Full Chrome control via CDP | HIGH — web credential exposure | Run in Docker sandbox only |
| **open-prose** | Multi-agent workflow orchestration | HIGH — system-wide automation | Power users with isolated environments |

### Skills to AVOID

| Skill Type | Why |
|-----------|-----|
| Unverified ClawHub skills from new publishers | 41.7% of popular skills found to contain vulnerabilities |
| Skills requesting admin/root privileges | No legitimate reason for a personal assistant |
| Skills with unrestricted network + credential access | Data exfiltration vector |

---

## 4. ClawHub Marketplace Security

### Known Threats (2026)

- 230+ malicious skills identified by security researchers
- 41.7% of 2,890+ popular skills contain security vulnerabilities
- 12-20% of ClawHub skills confirmed malicious (varies by study)
- Attack vectors include: command injection, credential theft, data exfiltration, backdoor installation
- AMOS macOS infostealer found bundled in some uploads

### Pre-Installation Checklist

1. Verify publisher reputation on ClawHub
2. Review skill permissions and SKILL.md before enabling
3. Check community ratings and comments
4. Test in isolated environment first
5. Never auto-install skills from unknown sources

### Disable ClawHub Auto-Install (Recommended)

```json
{
  "skills": {
    "registry": {
      "enabled": false
    }
  }
}
```

---

## 5. Channel Setup for Personal Assistant

| Channel | Use Case | Setup Method |
|---------|----------|--------------|
| **Slack** (configured) | Work collaboration, team queries | Already connected |
| **WhatsApp** | Mobile-first personal assistant | Scan QR in OpenClaw UI |
| **Telegram** | Quick commands, lightweight messaging | Create bot via @BotFather |
| **Discord** | Community/hobby management | Create bot in Developer Portal |
| **iMessage** (BlueBubbles) | macOS-native messaging | Requires BlueBubbles server |

---

## 6. Security Hardening Checklist

### Network Security

- [x] Gateway bound to loopback (localhost only)
- [x] Token-based authentication enabled
- [ ] For remote access, use SSH tunnel or Tailscale — never expose port 18789

```bash
# SSH tunnel from another machine
ssh -L 18789:localhost:18789 your-mac-user@your-mac-ip
```

### Credential Security

- [x] Using AWS IAM (aws-sdk auth) — no API keys in config files
- [ ] Rotate gateway token periodically:

```bash
NEW_TOKEN=$(openssl rand -hex 24)
echo $NEW_TOKEN
# Update token in ~/.openclaw/openclaw.json
```

### Skills Security

- [ ] Audit all installed skills before enabling: `openclaw skills list`
- [ ] Read SKILL.md files before installation
- [ ] Disable ClawHub auto-install for full control

### DM Security

- [ ] Keep default DM pairing mode (requires pairing code for unknown senders)
- [ ] Configure allowlists for known contacts on WhatsApp/Telegram

### Monitoring

```bash
# Check active connections
lsof -i :18789

# Monitor session logs
tail -f ~/.openclaw/agents/*/sessions/*.jsonl

# Check OpenClaw health
openclaw doctor
```

### File Permissions

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
chmod 600 ~/.openclaw/gateway_token.txt
chmod 600 ~/.openclaw/credentials/*
```

---

## 7. Bedrock Model Strategy

| Task Type | Best Model | Cost (per 1M tokens in/out) |
|-----------|-----------|---------------------------|
| Quick questions, casual chat | Nova 2 Lite | $0.30 / $2.50 |
| Balanced daily use | Claude Sonnet 4.5 | $3.00 / $15.00 |
| Fast simple tasks | Claude Haiku 4.5 | $1.00 / $5.00 |
| Cost-optimized bulk work | Nova Pro | $0.80 / $3.20 |
| Complex reasoning, coding | Claude Opus 4.6 | Highest, most capable |

Recommendation: Use Claude Sonnet 4.5 or Nova Pro as default for everyday use. Reserve Opus 4.6 for complex tasks. This saves 60-80% on Bedrock costs.

---

## 8. Skills Loading Priority

OpenClaw loads skills in this order (highest priority first):

1. **Workspace skills** (`<workspace>/skills/`) — project-specific overrides
2. **Managed/local skills** (`~/.openclaw/skills/`) — user-installed
3. **Bundled skills** — shipped with OpenClaw

---

## 9. Risk Assessment Matrix

| Risk Category | Severity | Likelihood | Priority |
|---------------|----------|------------|----------|
| CVE-2026-25253 RCE | Critical | High | P0 — Patch immediately |
| Exposed instances (default binding) | High | High | P0 — Use loopback only |
| Malicious ClawHub skills | High | Medium | P1 — Audit before install |
| Credential exposure in config | High | Medium | P1 — Use IAM, file permissions |
| Default config weaknesses | Medium | High | P2 — Harden settings |

---

## 10. Quick Start Summary

### Immediate Setup

```bash
# Lock down file permissions
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
chmod 600 ~/.openclaw/gateway_token.txt

# Verify security
openclaw doctor
```

### What You Already Have Working

- OpenClaw v2026.2.19-2 installed globally
- Bedrock integration with 5 models (Opus 4.6, Sonnet 4.5, Haiku 4.5, Nova 2 Lite, Nova Pro)
- AWS IAM authentication (no plaintext API keys)
- Slack channel connected
- Gateway secured with token auth on loopback

### Next Steps

1. Enable Tier 1 skills (weather, apple-notes, apple-reminders, summarize)
2. Set file permissions (chmod commands above)
3. Connect WhatsApp or Telegram for mobile access
4. Consider switching default model to Sonnet 4.5 for cost savings
5. Run `openclaw doctor` to verify setup health
