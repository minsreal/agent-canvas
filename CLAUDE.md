# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## What this is

Agent Canvas (`@openhands/agent-canvas`) is a React + TypeScript frontend for running and monitoring OpenHands agents (and other ACP-compatible agents like Claude Code/Codex) across local, remote, and cloud backends. It talks directly to the [OpenHands Agent Server](https://github.com/OpenHands/software-agent-sdk/tree/main/openhands-agent-server/openhands/agent_server) via `@openhands/typescript-client` — there is no separate app backend. It ships both as a standalone app (`npm run build`) and as library entrypoints for embedding in host apps (`npm run build:lib`).

## Commands

```sh
npm run dev              # full local stack: agent-server + automation backend (via uvx) + Vite + ingress proxy
npm run dev:minimal       # agent-server + Vite only, no automation/ingress — http://localhost:3001
npm run dev:mock          # frontend only, against MSW mocks (VITE_MOCK_API=true)
npm run dev:static        # static frontend build + backend, better for remote/tunneled access

npm run test              # vitest run (runs make-i18n first)
npm run test -- __tests__/api/agent-server-config.test.ts   # single test file
npm run test:e2e:mock-llm              # full-stack Playwright against a scripted mock LLM, no credentials needed
npm run test:e2e:mock-llm -- -g "name" # single mock-llm test by name
npm run test:e2e:live                  # real LLM-backed smoke test; needs an API key, see docs/DEVELOPMENT.md
npm run test:coverage

npm run lint               # typecheck + eslint src + prettier --check
npm run lint:fix
npm run typecheck          # react-router typegen && tsc (needs make-i18n to have run at least once)
npm run make-i18n          # regenerates src/i18n/declaration.ts + public/locales/*/openhands.json from translation.json
npm run check-translation-completeness

npm run build               # standalone app (build:app)
npm run build:lib           # library entrypoints (browser/conversation/files/settings/sidebar/terminal/i18n)
```

`npm run typecheck`/`test` assume `src/i18n/declaration.ts` exists — run `npm run make-i18n` first on a fresh checkout if it's missing.

## Architecture

- `src/api/` — service adapters, one directory per feature (`feature-service/feature-service.api.ts` + `feature.types.ts`). All local agent-server calls go through typed `@openhands/typescript-client` classes via `src/api/agent-server-client-options.ts` (never raw axios/fetch — enforced by `src/api/no-direct-agent-server-calls.test.ts`); cloud backend calls go through `callCloudProxy` in `src/api/cloud/proxy.ts`. See `src/api/README.md` and the "API Access Rules" section pulled in from AGENTS.md above.
- `src/hooks/query/` and `src/hooks/mutation/` — TanStack Query wrappers around services. Components must not call services directly.
- `src/stores/` — Zustand stores for conversation/UI state (one store per concern, e.g. `conversation-store.ts`, `agent-store.ts`, `metrics-store.ts`).
- `src/components/` — feature UI (`conversation`, `browser`, `files`, `settings`, `sidebar`, `terminal`, `providers`) plus `shared`/`ui` for cross-cutting primitives.
- `src/routes/` + `src/routes.ts` — React Router route modules and the route tree.
- `src/i18n/` — all user-facing strings are `t(I18nKey.…)` lookups against `translation.json`; `declaration.ts` and `public/locales/*` are generated, never hand-edited.
- `src/mocks/` — MSW handlers used by `dev:mock` and by Vitest/component tests.
- `bin/agent-canvas.mjs` — the published CLI entrypoint; `scripts/dev-*.mjs` are the various local dev-stack launchers (see `docs/DEVELOPMENT.md` for the full matrix and env vars).
- `tests/e2e/mock-llm/` — full-stack Playwright tests against a scripted mock LLM server (production-fidelity: runs the real published binary). `tests/e2e/live/` — real-LLM smoke tests, run separately and never as part of default CI.

For the system-boundary view (what Agent Canvas is/isn't responsible for) and the runtime-services info passed into agent conversations, see `docs/architecture.md`. For the full dev-stack/env-var matrix, see `docs/DEVELOPMENT.md`.

## Conventions

The imported AGENTS.md content above is the authoritative, CI-enforced source for:

- API access rules (typed client usage, `callCloudProxy`)
- The i18n / no-magic-strings rules (`i18next/no-literal-string` is an `error`)
- Testing rules (TDD, AAA, mock the service not the hook)
- The tracking/analytics split between `telemetry.ts` and `useTracking`
- E2E test frameworks and debugging workflow

Read it before touching API calls, adding user-facing strings, or writing tests — violating these fails CI via dedicated guard tests/lint rules, not just review.
