# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Also read `AGENTS.md` for operational guidelines (PR workflows, multi-agent safety, release processes, VM ops).

## Project Overview

OpenClaw is a personal AI assistant gateway that connects AI models (Claude, ChatGPT, etc.) to messaging platforms (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Google Chat, Microsoft Teams, Matrix, and more). It runs locally on your device as a single-user, privacy-first system with a WebSocket-based gateway control plane.

## Build, Test, and Lint Commands

```bash
pnpm install                  # install deps (also: bun install)
pnpm build                    # type-check + compile to dist/
pnpm tsgo                     # TypeScript type-check only
pnpm check                    # type-check + lint (oxlint) + format check (oxfmt)
pnpm lint                     # oxlint only (type-aware)
pnpm format:fix               # auto-format with oxfmt
pnpm test                     # run unit tests (vitest)
pnpm test:coverage            # tests with V8 coverage report
pnpm test:e2e                 # end-to-end tests
pnpm test -- src/foo.test.ts  # run a single test file
pnpm openclaw ...             # run CLI in dev mode (uses bun via tsx)
pnpm dev                      # alternative dev mode
```

**Full gate before pushing:** `pnpm build && pnpm check && pnpm test`

**Commits:** use `scripts/committer "<msg>" <file...>` instead of manual `git add`/`git commit`.

## Runtime Requirements

- Node.js **22+** (ESM). Bun supported for dev execution (`bun <file.ts>`).
- TypeScript strict mode, ES2023 target, NodeNext module resolution.
- Linting: Oxlint (`.oxlintrc.json`). Formatting: Oxfmt (`.oxfmtrc.jsonc`).
- Test framework: Vitest with V8 coverage (70% threshold for lines/functions/statements, 55% branches).

## Architecture

```
Messaging Channels (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, ...)
        |
   Gateway Server (WebSocket @ :18789, Hono HTTP)
        |
   +----+----+----+----+
   |         |         |
 Routing   Agent    Control UI
 (bindings) (Pi RPC) (Lit web components)
```

### Entry Points

- **CLI binary:** `src/entry.ts` -> `src/cli/run-main.ts` (normalizes env, validates Node 22+, fast-path routes common commands, builds Commander program)
- **Library exports:** `src/index.ts` (public API for consumers)
- **Command registry:** `src/cli/program/command-registry.ts`

### Key Modules (src/)

| Directory | Purpose |
|-----------|---------|
| `cli/` | CLI wiring, Commander program, progress utilities |
| `commands/` | Individual CLI command implementations |
| `gateway/` | WebSocket server + client, device auth, protocol framing |
| `channels/` | Channel plugin system, registry, adapter interfaces |
| `routing/` | Message routing: bindings, session keys, agent resolution |
| `agents/` | Pi agent integration, auth profiles, tool registration |
| `plugins/` | Plugin discovery, loading, lifecycle (3-layer: discover -> load -> register) |
| `config/` | Zod-based config parsing/validation, YAML/JSON5 IO, file watching |
| `infra/` | Infrastructure: ports, TLS, device auth, network, outbound delivery |
| `media/` | Image/audio/video processing pipeline |
| `terminal/` | Terminal UI: tables, ANSI styling, palette (`src/terminal/palette.ts`) |

### Messaging Flow

1. **Inbound:** Channel adapter receives message -> normalizes to OpenClaw format
2. **Routing:** `src/routing/resolve-route.ts` extracts channel/account/peer, looks up binding rules, resolves to agentId + sessionKey
3. **Agent:** Pi agent processes via gateway, loads session memory, calls tools
4. **Outbound:** `src/infra/outbound/deliver.ts` formats per-channel, chunks if needed, sends via channel adapter

### Channel Plugin System

Channels implement adapter interfaces defined in `src/channels/plugins/types.ts`:
- `ChannelMessagingAdapter` (send/receive)
- `ChannelOutboundAdapter` (format/deliver)
- `ChannelAuthAdapter` (login/logout)
- `ChannelConfigAdapter`, `ChannelSetupAdapter`, `ChannelHeartbeatAdapter`

Core channels: `src/telegram/`, `src/discord/`, `src/slack/`, `src/signal/`, `src/imessage/`, `src/web/`
Extension channels: `extensions/*` (msteams, matrix, zalo, voice-call, etc.)

### Monorepo Layout

- **Root package:** CLI + gateway + library (`src/`)
- **`ui/`:** Lit-based web control UI (separate Vite build)
- **`extensions/*`:** Channel plugins (workspace packages, each with own `package.json`)
- **`packages/*`:** Bot variants (clawdbot, moltbot)
- **`apps/`:** Native apps (macOS SwiftUI, iOS Swift, Android Kotlin)
- **`skills/`:** 55+ built-in automation skills
- **`docs/`:** Mintlify documentation (docs.openclaw.ai)

## Important Patterns

- **Dependency injection:** Use `createDefaultDeps()` and `createOutboundSendDeps()` for pluggable channel wiring. Follow existing patterns when adding CLI commands.
- **CLI progress:** Use `src/cli/progress.ts` (osc-progress + @clack/prompts spinner); don't hand-roll spinners.
- **Terminal output:** Use shared palette (`src/terminal/palette.ts`) and table utilities (`src/terminal/table.ts`); no hardcoded ANSI colors.
- **Tool schemas:** Avoid `Type.Union`/`anyOf`/`oneOf`/`allOf` in tool input schemas. Use `stringEnum`/`optionalStringEnum` for string lists. Avoid raw `format` property names.
- **Plugin deps:** Keep plugin-only deps in the extension `package.json`. Avoid `workspace:*` in `dependencies`; use `devDependencies` or `peerDependencies` for `openclaw`.
- **Config validation:** Zod schemas in `src/config/zod-schema*.ts` with plugin-aware validation in `src/config/validation.ts`.

## Coding Style

- TypeScript ESM, strict typing, avoid `any`.
- Oxlint + Oxfmt; run `pnpm check` before commits.
- Naming: **OpenClaw** in prose/headings; `openclaw` for CLI/package/paths/config keys.
- Keep files under ~500-700 LOC; extract helpers when it improves clarity.
- Tests colocated as `*.test.ts`; e2e as `*.e2e.test.ts`.
- Never update the Carbon dependency. Patched deps (`pnpm.patchedDependencies`) must use exact versions.
