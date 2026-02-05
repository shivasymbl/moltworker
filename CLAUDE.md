# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Moltworker runs [OpenClaw](https://github.com/openclaw/openclaw) (formerly Moltbot) personal AI assistant in a Cloudflare Sandbox container. It provides a fully managed deployment with optional R2 persistence, multi-channel chat support (Telegram, Discord, Slack), and browser automation via CDP.

**Important:** The CLI tool is still named `clawdbot` internally (upstream hasn't renamed), so CLI commands and config paths use that name.

## Commands

```bash
npm test              # Run tests (vitest)
npm run test:watch    # Watch mode
npm run test:coverage # Coverage report
npm run typecheck     # TypeScript check
npm run build         # Build worker + Vite client
npm run deploy        # Build and deploy to Cloudflare
npm run dev           # Vite dev server
npm run start         # wrangler dev (local worker)
```

## Architecture

```
Browser → Cloudflare Worker (index.ts) → Cloudflare Sandbox Container
                                              └── Moltbot Gateway (port 18789)
```

The Worker manages the sandbox lifecycle, proxies HTTP/WebSocket requests, and passes secrets as environment variables. The container runs the Moltbot gateway with its Control UI.

### Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Main Hono app, Worker entry point, route mounting |
| `src/types.ts` | TypeScript types (`MoltbotEnv`, `AppEnv`) |
| `src/config.ts` | Constants (ports, timeouts, paths) |
| `Dockerfile` | Container image: `cloudflare/sandbox` + Node 22 + Moltbot |
| `start-moltbot.sh` | Container startup: configures moltbot from env vars |
| `wrangler.jsonc` | Cloudflare Worker + Container configuration |

### Source Structure

- `src/auth/` - Cloudflare Access JWT verification (jwt.ts, jwks.ts, middleware.ts)
- `src/gateway/` - Container lifecycle (process.ts, env.ts, r2.ts, sync.ts)
- `src/routes/` - HTTP handlers (api.ts, admin-ui.ts, debug.ts, cdp.ts)
- `src/client/` - React admin UI for device pairing (Vite)

## Key Patterns

### CLI Commands in Container

Always include `--url ws://localhost:18789`. Commands take 10-15 seconds due to WebSocket overhead:
```typescript
sandbox.startProcess('clawdbot devices list --json --url ws://localhost:18789')
```

Use `waitForProcess()` helper from `src/gateway/utils.ts`.

### Success Detection

CLI outputs "Approved" (capital A). Use case-insensitive checks:
```typescript
stdout.toLowerCase().includes('approved')
```

### Environment Variables

- `DEV_MODE=true` - Skips CF Access auth AND bypasses device pairing (local dev only)
- `DEBUG_ROUTES=true` - Enables `/debug/*` routes
- See `src/types.ts` for full `MoltbotEnv` interface

### Adding New Environment Variables

1. Add to `MoltbotEnv` interface in `src/types.ts`
2. If passed to container, add to `buildEnvVars()` in `src/gateway/env.ts`
3. Update `.dev.vars.example`
4. Document in README.md secrets table

## Testing

Tests use Vitest with files colocated as `*.test.ts`. Coverage excludes `src/client/`.

```bash
npm test                    # Single run
npm run test:watch          # Watch mode
npm run test:coverage       # HTML + text coverage report
```

## R2 Storage Gotchas

- **rsync**: Use `rsync -r --no-times` (s3fs doesn't support timestamps)
- **Mount check**: Use `mount | grep s3fs`, not SDK error messages
- **Never delete mount directory**: `/data/moltbot` IS the R2 bucket
- **Process status**: Check for expected output files, not `proc.status`

## Docker Image Caching

Bump the cache bust comment when changing `moltbot.json.template` or `start-moltbot.sh`:
```dockerfile
# Build cache bust: 2026-01-26-v10
```

## Debugging

```bash
npx wrangler tail          # Live logs
npx wrangler secret list   # Check secrets
```

Enable `DEBUG_ROUTES=true` and check `/debug/processes` for container state.

## Local Development

```bash
npm install
cp .dev.vars.example .dev.vars
# Edit .dev.vars with ANTHROPIC_API_KEY, DEV_MODE=true
npm run start
```

**Note:** WebSocket proxying through `wrangler dev` has limitations. Deploy to Cloudflare for full functionality.
