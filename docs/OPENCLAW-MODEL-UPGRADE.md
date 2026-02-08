# OpenClaw Model Upgrade Guide

Guide for upgrading Claude models in OpenClaw deployments.

## Upgrading to Claude Opus 4.6

### Prerequisites

OpenClaw version **2026.2.6-3 or later** is required. Earlier versions don't have Opus 4.6 in the model catalog and will crash with `Unknown model: anthropic/claude-opus-4-6`.

```bash
# Check version
openclaw --version

# Update if needed
npm update -g openclaw
```

### Configuration

Edit `~/.openclaw/openclaw.json` to set Opus 4.6 as primary with 4.5 as fallback:

```bash
jq '.agents.defaults.model.primary = "anthropic/claude-opus-4-6" |
    .agents.defaults.model.fallbacks = ["anthropic/claude-opus-4-5"]' \
    ~/.openclaw/openclaw.json > ~/.openclaw/openclaw.json.tmp && \
    mv ~/.openclaw/openclaw.json.tmp ~/.openclaw/openclaw.json
```

**Before:**
```json
"model": {
  "primary": "anthropic/claude-opus-4-5",
  "fallbacks": []
}
```

**After:**
```json
"model": {
  "primary": "anthropic/claude-opus-4-6",
  "fallbacks": ["anthropic/claude-opus-4-5"]
}
```

The fallback ensures graceful degradation if Opus 4.6 has issues.

### Apply Changes

```bash
# Restart gateway (systemd)
systemctl --user restart openclaw-gateway

# Or if using system service
systemctl restart openclaw
```

### Verify

Check logs for successful model activation:

```bash
journalctl --user -u openclaw-gateway -n 20 | grep -E "(model|connected|heartbeat)"
```

Expected output:
```
[heartbeat] started
[gateway] agent model: anthropic/claude-opus-4-6
[slack] socket mode connected
```

### Session Handling

Existing sessions stay on the old model. To use Opus 4.6:
- Start a `/new` session in chat
- Or let the gateway restart force it

## Available Models

Check available models in your OpenClaw version:

```bash
openclaw models list | grep opus
```

Example output:
```
anthropic/claude-opus-4-5    text+image 195k   default,configured,alias:opus
anthropic/claude-opus-4-6    text+image 1M     configured
```

## Opus 4.6 Features

- **1M token context window** (beta) - up from 200k
- **Agent teams** - split tasks across multiple coordinating agents
- **Improved agentic coding** - better planning, longer sustained tasks
- **Enhanced debugging** - catches its own mistakes more reliably
- **Same pricing** - $5/$25 per million tokens

## Troubleshooting

### "Unknown model: anthropic/claude-opus-4-6"

**Cause:** OpenClaw version too old (< 2026.2.6-3)

**Fix:**
```bash
npm update -g openclaw
openclaw --version  # Should be 2026.2.6-3 or later
```

### Model not switching

**Cause:** Existing session using old model

**Fix:** Start a `/new` session or restart the gateway

### Rollback to 4.5

If issues occur:
```bash
jq '.agents.defaults.model.primary = "anthropic/claude-opus-4-5"' \
    ~/.openclaw/openclaw.json > ~/.openclaw/openclaw.json.tmp && \
    mv ~/.openclaw/openclaw.json.tmp ~/.openclaw/openclaw.json

systemctl --user restart openclaw-gateway
```

## Version History

| OpenClaw Version | Opus Support |
|------------------|--------------|
| 2026.2.3-1 | Opus 4.5 only |
| 2026.2.6-3 | Opus 4.5 + 4.6 |

## References

- [Anthropic Opus 4.6 Announcement](https://www.anthropic.com/news/claude-opus-4-6)
- [Claude API Models Overview](https://platform.claude.com/docs/en/about-claude/models/overview)
