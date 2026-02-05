# OpenClaw Secrets Management with Doppler

Guide for secure secret handling in OpenClaw deployments using Doppler.

## The Golden Rule

**Never commit secrets. Always inject.**

```
Secrets live in Doppler → Injected at runtime → Referenced via ${VAR_NAME}
```

## Quick Reference

```
+------------------------------------------+
|  SECRETS PATTERN QUICK REFERENCE         |
+------------------------------------------+
|                                          |
|  Secrets → Doppler (never git)           |
|  Config  → ${VAR_NAME} syntax            |
|  Service → doppler run -- node ...       |
|                                          |
|  WRONG: "token": "xoxb-actual-value"     |
|  WRONG: "token": "env:VAR_NAME"          |
|  RIGHT: "token": "${VAR_NAME}"           |
|                                          |
|  Audit: grep -E "xoxb-|sk-ant-" *.json   |
|                                          |
+------------------------------------------+
```

## What Goes Where

| Type | Location | Example |
|------|----------|---------|
| API Keys | Doppler only | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` |
| Slack Tokens | Doppler only | `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` |
| Database Passwords | Doppler only | `NEO4J_PASSWORD` |
| OAuth Credentials | Doppler only | `GOOGLE_CLIENT_SECRET` |
| Config References | openclaw.json | `"${SLACK_BOT_TOKEN}"` |
| Non-sensitive Config | openclaw.json | ports, policies, model names |

## Correct Pattern

### openclaw.json (safe to commit)

```json
{
  "channels": {
    "slack": {
      "enabled": true,
      "botToken": "${SLACK_BOT_TOKEN}",
      "appToken": "${SLACK_APP_TOKEN}",
      "userToken": "${SLACK_USER_TOKEN}"
    }
  },
  "gateway": {
    "auth": {
      "mode": "token",
      "token": "${GATEWAY_AUTH_TOKEN}"
    }
  },
  "talk": {
    "apiKey": "${ELEVENLABS_API_KEY}"
  },
  "tools": {
    "web": {
      "search": {
        "apiKey": "${BRAVE_SEARCH_API_KEY}"
      }
    }
  }
}
```

### Doppler (never in git)

```
ANTHROPIC_API_KEY=sk-ant-api03-...
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
SLACK_USER_TOKEN=xoxp-...
GATEWAY_AUTH_TOKEN=...
ELEVENLABS_API_KEY=sk_...
BRAVE_SEARCH_API_KEY=...
```

### systemd service

```ini
[Service]
WorkingDirectory=/root/.openclaw
ExecStart=/usr/bin/doppler run --project <project> --config prd -- /usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789
```

## Wrong Patterns (Never Do This)

```json
// WRONG: Hardcoded token
"botToken": "xoxb-4647252096448-actual-token-here"

// WRONG: env: prefix (doesn't work in OpenClaw)
"botToken": "env:SLACK_BOT_TOKEN"

// WRONG: Unquoted variable
"botToken": ${SLACK_BOT_TOKEN}

// RIGHT: Quoted variable reference
"botToken": "${SLACK_BOT_TOKEN}"
```

## Setup Guide

### 1. Install Doppler CLI

```bash
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -sLf --retry 3 --tlsv1.2 --proto "=https" 'https://packages.doppler.com/public/cli/gpg.DE2A7741A397C129.key' | sudo gpg --dearmor -o /usr/share/keyrings/doppler-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/doppler-archive-keyring.gpg] https://packages.doppler.com/public/cli/deb/debian any-version main" | sudo tee /etc/apt/sources.list.d/doppler-cli.list
sudo apt-get update && sudo apt-get install -y doppler
```

### 2. Authenticate

```bash
doppler login
```

### 3. Create Project

```bash
doppler projects create jarvis
```

### 4. Add Secrets

```bash
doppler secrets set \
  ANTHROPIC_API_KEY="sk-ant-..." \
  SLACK_BOT_TOKEN="xoxb-..." \
  SLACK_APP_TOKEN="xapp-..." \
  GATEWAY_AUTH_TOKEN="..." \
  --project jarvis --config prd
```

### 5. Configure Workspace

```bash
cd ~/.openclaw
doppler setup --project jarvis --config prd
```

### 6. Update systemd Service

Edit `/root/.config/systemd/user/openclaw-gateway.service`:

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target

[Service]
WorkingDirectory=/root/.openclaw
ExecStart=/usr/bin/doppler run --project jarvis --config prd -- /usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

### 7. Reload and Restart

```bash
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

## Audit Commands

Run these before any deployment:

### Check for hardcoded secrets in config

```bash
# Should return nothing
grep -E "(xoxb-|xoxp-|xapp-|sk-ant-|sk_[a-z0-9])" ~/.openclaw/openclaw.json
```

### Verify Doppler wrapper in service

```bash
# Should show: ExecStart=/usr/bin/doppler run ...
grep "doppler run" ~/.config/systemd/user/openclaw-gateway.service
```

### Verify secrets are accessible

```bash
doppler secrets --only-names
```

### Test gateway is running with Doppler

```bash
systemctl --user status openclaw-gateway
# Should show doppler as parent process
```

## Required Secrets Reference

| Secret | Required | Description |
|--------|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes | Claude API access |
| `SLACK_BOT_TOKEN` | If Slack | Bot user OAuth token (xoxb-...) |
| `SLACK_APP_TOKEN` | If Slack | App-level token for socket mode (xapp-...) |
| `SLACK_USER_TOKEN` | If Slack | User token for enhanced features (xoxp-...) |
| `GATEWAY_AUTH_TOKEN` | Yes | Gateway authentication token |
| `ELEVENLABS_API_KEY` | If TTS | ElevenLabs text-to-speech |
| `BRAVE_SEARCH_API_KEY` | If search | Brave web search API |
| `GEMINI_API_KEY` | If Gemini | Google Gemini API |
| `OPENAI_API_KEY` | If OpenAI | OpenAI API access |
| `NEO4J_PASSWORD` | If graph | Neo4j database password |

## Troubleshooting

### "invalid_auth" from Slack

1. Check syntax: must be `${SLACK_BOT_TOKEN}` not `env:SLACK_BOT_TOKEN`
2. Verify token exists: `doppler secrets get SLACK_BOT_TOKEN --plain`
3. Test token:
   ```bash
   curl -s "https://slack.com/api/auth.test" \
     -H "Authorization: Bearer $(doppler secrets get SLACK_BOT_TOKEN --plain)"
   ```

### Service starts but no env vars

1. Verify doppler.yaml exists in WorkingDirectory
2. Verify systemd uses `doppler run --` in ExecStart
3. Check Doppler login: `doppler whoami`

### Gateway not starting

```bash
# Check logs
journalctl --user -u openclaw-gateway -n 50

# Verify config is valid JSON
cat ~/.openclaw/openclaw.json | jq .
```

## Migration Checklist

When migrating an existing deployment to Doppler:

- [ ] Backup current config: `cp openclaw.json openclaw.json.backup`
- [ ] Install Doppler CLI on server
- [ ] Create Doppler project
- [ ] Extract secrets from config and add to Doppler
- [ ] Update config to use `${VAR_NAME}` syntax
- [ ] Update systemd service with `doppler run --` wrapper
- [ ] Reload systemd: `systemctl --user daemon-reload`
- [ ] Restart gateway: `systemctl --user restart openclaw-gateway`
- [ ] Run audit commands to verify no hardcoded secrets
- [ ] Verify gateway is running: `curl http://localhost:18789`
- [ ] Delete backup after confirming everything works

## Security Best Practices

1. **Never log secrets** - Don't use `echo $SECRET` in scripts
2. **Rotate compromised secrets immediately** - If a secret may have been exposed
3. **Use separate configs per environment** - dev, stg, prd in Doppler
4. **Audit regularly** - Run grep checks before deployments
5. **Limit access** - Only give Doppler access to those who need it
