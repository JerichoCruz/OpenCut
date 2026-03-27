# CLAUDE.md

## Overview

Privacy-first, open-source video editor (CapCut alternative). Video processing runs client-side via FFmpeg WASM — no GPU needed on the server.

This is a self-hosted fork. Upstream: `OpenCut-app/OpenCut`.

## Tech Stack

- **Runtime:** Bun 1.2.18
- **Framework:** Next.js 16.1.3 (Turbopack) + React 19
- **Language:** TypeScript 5.8.3
- **Monorepo:** Turbo 2.7.5
- **Styling:** Tailwind CSS 4.2.1 + shadcn/ui (Radix)
- **State:** Zustand 5.0.2
- **ORM:** Drizzle (PostgreSQL 17)
- **Auth:** Better Auth 1.4.15
- **Linting/Formatting:** Biome
- **Media:** FFmpeg.js (WASM), Wavesurfer.js

## Project Structure

```
OpenCut/                          # Monorepo root
├── apps/web/                     # Main Next.js app
│   ├── src/
│   │   ├── app/                  # Next.js App Router (pages, API routes)
│   │   ├── components/           # React components (editor/, ui/, landing/)
│   │   ├── core/                 # EditorCore singleton
│   │   ├── hooks/                # Custom hooks (use-editor.ts is the main one)
│   │   ├── lib/                  # Domain logic (actions/, commands/, timeline/, media/)
│   │   ├── stores/               # Zustand stores (editor, timeline, panel, etc.)
│   │   ├── constants/            # App constants by domain
│   │   ├── services/             # Service layer (API, DB, external)
│   │   ├── types/                # TypeScript type definitions
│   │   └── utils/                # Generic helpers
│   ├── migrations/               # Drizzle DB migrations
│   ├── public/                   # Static assets (logos, ffmpeg WASM)
│   └── Dockerfile                # Multi-stage build (bun:alpine)
├── packages/
│   ├── env/                      # Zod env validation schemas
│   │   └── src/
│   │       ├── web.ts            # Web app env schema (ALL vars validated here)
│   │       └── tools.ts          # Tools env schema
│   └── ui/                       # Shared UI components (icons)
├── docker-compose.yml            # 4 services: db, redis, redis-http, web
├── turbo.json                    # Task orchestration
└── biome.json                    # Linting & formatting config
```

## Lib vs Utils

- `lib/` — domain logic (specific to this app)
- `utils/` — small helper utils (generic, could be copy-pasted into any other app)

## Common Commands

```bash
# Docker (production) — run from repo root
docker compose up -d                          # Start all services
docker compose up -d --build web              # Rebuild web after code changes
docker compose logs -f web                    # Tail web logs
docker compose down                           # Stop (data preserved)
docker compose down -v                        # Stop + destroy volumes

# Local dev
bun install                                   # Install dependencies
docker compose up -d db redis serverless-redis-http  # Start backend services only
bun dev:web                                   # Next.js dev server (Turbopack)
bun build:web                                 # Production build
bun test                                      # Run tests
bun lint:web                                  # Biome lint
bun lint:web:fix                              # Biome lint + auto-fix
bun format:web                                # Biome format
```

## Docker Stack

Four services on an isolated `opencut-network`:

| Service | Image | Port | Notes |
|---------|-------|------|-------|
| db | postgres:17 | 5432 | User/pass: opencut/opencut, volume: `postgres_data` |
| redis | redis:7-alpine | 6379 | Stateless cache, no volume needed |
| serverless-redis-http | hiett/serverless-redis-http | 8079→80 | Redis REST wrapper |
| web | Built from `apps/web/Dockerfile` | 3100→3000 | Next.js standalone |

All services have health checks. Web depends on db + redis-http being healthy.

**Secrets live in `.env` (gitignored).** The `BETTER_AUTH_SECRET` is the critical one — generate with `openssl rand -base64 32`.

## Environment Variables

Validated by Zod in `packages/env/src/web.ts`. **All are required** — missing or malformed values crash the app at runtime with a ZodError.

| Variable | Type | Notes |
|----------|------|-------|
| `NODE_ENV` | enum | `development`, `production`, or `test` |
| `DATABASE_URL` | string | Must start with `postgres://` or `postgresql://` |
| `BETTER_AUTH_SECRET` | string | Session signing key — **keep in .env, not compose** |
| `UPSTASH_REDIS_REST_URL` | url | `http://serverless-redis-http:80` in Docker |
| `UPSTASH_REDIS_REST_TOKEN` | string | Must match `SRH_TOKEN` on redis-http service |
| `NEXT_PUBLIC_SITE_URL` | url | Defaults to `http://localhost:3000` |
| `NEXT_PUBLIC_MARBLE_API_URL` | url | `https://api.marblecms.com` |
| `MARBLE_WORKSPACE_KEY` | string | Placeholder OK |
| `FREESOUND_CLIENT_ID` | string | Placeholder OK |
| `FREESOUND_API_KEY` | string | Placeholder OK |
| `CLOUDFLARE_ACCOUNT_ID` | string | Placeholder OK |
| `R2_ACCESS_KEY_ID` | string | Placeholder OK |
| `R2_SECRET_ACCESS_KEY` | string | Placeholder OK |
| `R2_BUCKET_NAME` | string | Placeholder OK |
| `MODAL_TRANSCRIPTION_URL` | **url** | **Must be a valid URL** — `http://localhost:8000` as placeholder. `your_modal_url_here` will crash. |

### Known Upstream Issue

The `.env.example` and Dockerfile ship `MODAL_TRANSCRIPTION_URL` with invalid placeholders (`your_modal_url_here` and `http://localhost:0`). Both fail `z.url()` validation. This fork patches the Dockerfile to use `http://localhost:8000`. See upstream issues #717, #721.

## Core Editor System

The editor uses a **singleton EditorCore** that manages all editor state through specialized managers.

```
EditorCore (singleton)
├── playback: PlaybackManager
├── timeline: TimelineManager
├── scene: SceneManager
├── project: ProjectManager
├── media: MediaManager
└── renderer: RendererManager
```

### In React Components

**Always use the `useEditor()` hook:**

```typescript
import { useEditor } from '@/hooks/use-editor';

function MyComponent() {
  const editor = useEditor();
  const tracks = editor.timeline.getTracks();

  editor.timeline.addTrack({ type: 'media' });

  return <div>{tracks.length} tracks</div>;
}
```

The hook returns the singleton, subscribes to manager changes, and auto re-renders.

### Outside React

```typescript
import { EditorCore } from "@/core";

const editor = EditorCore.getInstance();
await editor.export({ format: "mp4", quality: "high" });
```

## Actions System

Actions are the trigger layer for user-initiated operations. Single source of truth: `@/lib/actions/definitions.ts`.

**To add a new action:**

1. Define in `@/lib/actions/definitions.ts`:

```typescript
export const ACTIONS = {
  "my-action": {
    description: "What the action does",
    category: "editing",
    defaultShortcuts: ["ctrl+m"],
  },
};
```

2. Add handler in `@/hooks/use-editor-actions.ts`:

```typescript
useActionHandler("my-action", () => { /* implementation */ }, undefined);
```

**In components, use `invokeAction()` for user-triggered operations:**

```typescript
import { invokeAction } from '@/lib/actions';

// Good - uses action system (toasts, validation, shortcuts)
const handleSplit = () => invokeAction("split-selected");

// Avoid - bypasses UX layer
const handleSplit = () => editor.timeline.splitElements({ ... });
```

Direct `editor.xxx()` calls are for internal use (commands, tests, complex multi-step operations).

## Commands System

Commands handle undo/redo. They live in `@/lib/commands/` organized by domain (timeline, media, scene).

Each command extends `Command` from `@/lib/commands/base-command` and implements:

- `execute()` — saves current state, then does the mutation
- `undo()` — restores the saved state

Actions are "what triggered this", commands are "how to do it (and undo it)".

## Database

Drizzle ORM with PostgreSQL. Tables managed by Better Auth:

- **users** — accounts (id, name, email, image)
- **sessions** — session tokens (token, expiry, userId)
- **accounts** — OAuth/provider accounts (providerId, tokens)
- **verifications** — email verification records

All tables have RLS enabled. Migrations live in `apps/web/migrations/`.

## Key Directories

| Path | Purpose |
|------|---------|
| `apps/web/src/app/api/` | API routes (auth, health, sounds) |
| `apps/web/src/app/editor/[project_id]/` | Editor page (dynamic route) |
| `apps/web/src/components/editor/` | Editor UI components |
| `apps/web/src/lib/actions/definitions.ts` | Action definitions (single source of truth) |
| `apps/web/src/lib/commands/` | Undo/redo commands by domain |
| `apps/web/src/stores/` | Zustand stores (editor, timeline, panel, etc.) |
| `packages/env/src/web.ts` | Zod env validation — **check here when adding env vars** |
