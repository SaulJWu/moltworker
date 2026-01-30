# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Cloudflare Worker that runs [Moltbot](https://molt.bot/) in a Cloudflare Sandbox container. The worker proxies HTTP/WebSocket requests to a Moltbot gateway running inside the container on port 18789, provides an admin UI at `/_admin/`, and handles authentication via Cloudflare Access + device pairing.

**Important:** The CLI tool is still named `clawdbot` internally (upstream hasn't renamed). CLI commands, config paths, and env vars use that name (e.g., `~/.clawdbot/clawdbot.json`, `CLAWDBOT_GATEWAY_TOKEN`).

## Commands

```bash
npm test                    # Run tests (vitest, single run)
npm run test:watch          # Watch mode
npm run test:coverage       # Coverage report
npm run typecheck           # tsc --noEmit
npm run build               # Build worker + React admin UI
npm run deploy              # Build and deploy to Cloudflare
npm run dev                 # Vite dev server (client UI only)
npm run start               # wrangler dev (local worker + sandbox)
npx vitest run src/auth/jwt.test.ts  # Run a single test file
```

## Architecture

```
Browser → Cloudflare Worker (Hono) → Sandbox Container (Node 22 + Moltbot gateway on :18789)
```

- **`src/index.ts`** — Main Hono app. Mounts routes, handles HTTP/WebSocket proxying to the container, manages sandbox lifecycle. Contains the catch-all proxy handler.
- **`src/auth/`** — Cloudflare Access JWT verification and Hono middleware. Routes are either public, CF Access-protected, or gateway-token-protected.
- **`src/gateway/`** — Container management: process lifecycle (`process.ts`), env var mapping (`env.ts`), R2 bucket mounting (`r2.ts`), backup sync (`sync.ts`).
- **`src/routes/`** — Route handlers: `api.ts` (device management, gateway restart), `admin-ui.ts` (static files), `debug.ts` (process inspection), `public.ts` (health checks), `cdp.ts` (Chrome DevTools Protocol WebSocket shim).
- **`src/client/`** — React admin UI built with Vite (base path `/_admin/`).
- **`Dockerfile`** — Container image: `cloudflare/sandbox:0.7.0` + Node.js 22 + clawdbot CLI.
- **`start-moltbot.sh`** — Container startup: restore from R2, configure from env vars, launch gateway.
- **`moltbot.json.template`** — Default gateway config template.

## Testing

Tests use Vitest with Node environment. Test files are colocated (`*.test.ts`). Client code (`src/client/`) is excluded from tests.

## Key Patterns

- **Env var mapping:** Worker env vars are mapped to container env vars in `buildEnvVars()` (`src/gateway/env.ts`). E.g., `MOLTBOT_GATEWAY_TOKEN` → `CLAWDBOT_GATEWAY_TOKEN`.
- **CLI commands from worker:** Always include `--url ws://localhost:18789`. Commands take 10-15 seconds due to WebSocket overhead. Use `waitForProcess()` helper.
- **CLI output parsing:** Success strings use "Approved" (capital A) — use case-insensitive checks.
- **Route structure:** Public routes (no auth) → CF Access-protected routes (`/_admin/`, `/api/*`, `/debug/*`) → catch-all proxy to gateway.
- **R2 sync:** Uses `rsync -r --no-times` (not `-a`) because s3fs doesn't support setting timestamps. Mount point `/data/moltbot` IS the R2 bucket — never `rm -rf` it.

## Gotchas

- `agents.defaults.model` in Moltbot config must be `{ "primary": "model/name" }`, not a string.
- `wrangler dev` has WebSocket proxying issues through sandbox — deploy for full testing.
- Dockerfile has a cache bust comment — bump when changing `moltbot.json.template` or `start-moltbot.sh`.
- Process `proc.status` from sandbox API may not update immediately; verify success by checking expected output instead.

## Adding New Features

- **New API endpoint:** Add handler in `src/routes/api.ts`, types in `src/types.ts`, client API in `src/client/api.ts`, tests.
- **New env var:** Add to `MoltbotEnv` in `src/types.ts`, add to `buildEnvVars()` in `src/gateway/env.ts`, update `.dev.vars.example`.

## Code Style

- TypeScript strict mode. Prefer explicit types for function signatures.
- Keep route handlers thin — extract logic to separate modules.
- Use Hono's context methods (`c.json()`, `c.html()`) for responses.
