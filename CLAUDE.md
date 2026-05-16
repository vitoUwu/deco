# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## Project Overview

Deco (`@deco/deco`) is a Git-based Visual CMS and full-stack framework for
TypeScript. It generates content editor UIs from TypeScript interfaces, uses
Preact for server-side JSX rendering, and supports HTMX for client-side
interactivity. Published on JSR as `@deco/deco`.

## Common Commands

```bash
deno task check          # Run all checks: fmt + type-check + test + bench
deno task start          # Alias for `deno task check`
deno task release        # Run release process
```

The `check` task expands to:
`deno fmt && deno check live.ts && deno test --unstable-http -A . && deno bench -A .`

**Running a single test:**

```bash
deno test --unstable-http -A path/to/file.test.ts
```

**Running a single benchmark:**

```bash
deno bench -A path/to/file.bench.ts
```

**Pre-commit hook:** Runs `deno task check` automatically (configured via
`githooks` in deno.json).

**Running a local engine against a deco site:**

```bash
cd path/to/your-deco-site
deno run -A jsr:@deco/deco/scripts/dev ../deco
```

## Architecture

### Block System (core abstraction)

Blocks are serializable, composable typed functions — the fundamental building
blocks of Deco. Each block type lives in `blocks/`:

- **Loaders** (`loader.ts`): Async data fetchers with caching and SWR support
- **Actions** (`action.ts`): Server-side mutations
- **Sections** (`section.ts`): UI components (Preact JSX)
- **Pages** (`page.tsx`): Route handlers
- **Handlers** (`handler.ts`): Custom HTTP endpoints
- **Matchers** (`matcher.ts`): Route matching logic
- **Apps** (`app.ts`): Integration plugins with manifests
- **Flags** (`flag.ts`): Feature toggles

All blocks are registered in a `DecoManifest` (keyed by block type), which the
engine uses for resolution and introspection.

### Engine (`engine/`)

- `engine/core/resolver.ts` — Main resolver: executes resolvables (composable
  async operations) with field-level resolution, middleware, and caching
- `engine/block.ts` — Block type definitions and introspection
- `engine/manifest/` — App manifest handling
- `engine/schema/` — JSON Schema generation from TypeScript types
- `engine/decofile/` — Git-based CMS file provider

### Runtime (`runtime/`)

- `runtime/mod.ts` — Main `Deco` class and HTTP server setup
- `runtime/middleware.ts` — Request middleware pipeline
- `runtime/handler.tsx` — HTTP handler factory
- `runtime/fresh/` — Fresh (Deno web framework) integration
- `runtime/htmx/` — HTMX interactivity support
- `runtime/caches/` — Loader caching system (supports FILE_SYSTEM, CACHE_API,
  REDIS engines)

### Context System (`deco.ts`)

Uses `AsyncLocalStorage` for request-scoped context. `RequestContext` provides
active request info. `DecoContext` is the global context with deployment info,
platform detection, and site details.

### Daemon (`daemon/`)

Background daemon for development: Git operations, AI features, WebSocket
realtime sync.

### Observability (`observability/`)

OpenTelemetry integration for distributed tracing and metrics.

## Key Entry Points

| Export path         | File                   | Purpose                |
| ------------------- | ---------------------- | ---------------------- |
| `@deco/deco`        | `mod.ts`               | Core framework exports |
| `@deco/deco/web`    | `mod.web.ts`           | Browser-only exports   |
| `@deco/deco/htmx`   | `runtime/htmx/mod.ts`  | HTMX support           |
| `@deco/deco/hooks`  | `hooks/mod.ts`         | Preact hooks           |
| `@deco/deco/blocks` | `blocks/mod.ts`        | Block type exports     |
| `@deco/deco/engine` | `engine/mod.ts`        | Engine internals       |
| `@deco/deco/o11y`   | `observability/mod.ts` | Observability          |
| `@deco/deco/daemon` | `daemon/mod.ts`        | Daemon exports         |

## Tech Stack

- **Runtime:** Deno
- **UI:** Preact (JSX with `react-jsx` transform, `jsxImportSource: "preact"`)
- **HTTP:** Hono
- **Web framework:** Fresh (Deno)
- **Client interactivity:** HTMX
- **Observability:** OpenTelemetry
- **State:** `@deco/durable`
- **Git ops:** `simple-git`

## Cache Environment Variables

| Variable                                   | Description                                             |
| ------------------------------------------ | ------------------------------------------------------- |
| `ENABLE_LOADER_CACHE`                      | Enable/disable loader cache                             |
| `WEB_CACHE_ENGINE`                         | Cache engine(s): `"FILE_SYSTEM,CACHE_API"` or `"REDIS"` |
| `CACHE_MAX_AGE_S`                          | Seconds before cache goes stale                         |
| `CACHE_MAX_SIZE`                           | Max file system cache size in bytes                     |
| `FILE_SYSTEM_CACHE_DIRECTORY`              | Directory for file system cache                         |
| `LOADER_CACHE_REDIS_CONNECTION_TIMEOUT_MS` | Redis connection timeout                                |
