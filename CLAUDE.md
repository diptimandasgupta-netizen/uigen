# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, Claude generates them via tool calls, and the preview renders live in an iframe sandbox. Works without an API key (falls back to a mock provider returning canned components).

## Commands

```bash
npm run setup       # install deps + prisma generate + prisma migrate dev
npm run dev         # start dev server (Turbopack) at http://localhost:3000
npm run dev:daemon  # same, runs in background, logs to logs.txt
npm run build       # production build
npm run lint        # ESLint
npm run test        # Vitest (jsdom)
npm run db:reset    # prisma migrate reset --force
```

Run a single test file:
```bash
npx vitest run src/path/to/file.test.ts
```

**Do not run `npm audit fix`** — dependencies are pinned to specific compatible versions; audit fix can break the app.

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. Without a valid key the app uses a mock provider that returns canned Counter/Form/Card components. The real provider uses `claude-haiku-4-5` (`src/lib/provider.ts`).

## Architecture

### Data Flow

1. User submits chat → POST `/api/chat/route.ts`
2. Claude streams a response using two tools: `str_replace_editor` (create/edit/insert/view files) and `file_manager` (rename/delete)
3. The server executes each tool against its own `VirtualFileSystem` instance (reconstructed from the serialized `files` payload sent with every request); simultaneously, the AI SDK client fires `onToolCall` → `handleToolCall` in `FileSystemContext`, which mirrors the same mutations on the client-side VFS for live preview without waiting for the stream to finish
4. `PreviewFrame` detects VFS changes via `refreshTrigger` and re-renders the iframe
5. `jsx-transformer.ts` uses Babel.standalone to transform JSX → blob URL → injected into the iframe sandbox
6. On finish (authenticated users only): server-side VFS + full message history are serialized to Prisma (SQLite)

**Key subtlety:** server and client each maintain independent VFS instances that stay in sync through the stream. The server's copy is authoritative for persistence; the client's copy drives live preview.

### Key Abstractions

**`VirtualFileSystem`** (`src/lib/file-system.ts`) — in-memory file tree (`Map<path, FileNode>`). All AI file operations go through this; nothing is written to disk. Serializes to/from JSON for DB persistence. The full serialized VFS is sent in the `body` of every chat request so the server can reconstruct state from scratch (no server-side session).

**`ChatContext`** + **`FileSystemContext`** (`src/lib/contexts/`) — decouple chat state from file state. `ChatContext` drives the AI SDK hook; `FileSystemContext` owns the VFS and exposes `handleToolCall` for AI mutations.

**Tool builders** (`src/lib/tools/`) — `buildStrReplaceTool` and `buildFileManagerTool` wrap file ops in Zod schemas and wire them into the AI SDK's tool list.
- `str_replace_editor`: content operations (`create`, `str_replace`, `insert`, `view`). The `undo_edit` command is intentionally unimplemented — it returns an error string instead of reverting.
- `file_manager`: filesystem operations (`rename`, `delete`).

**System prompt** (`src/lib/prompts/generation.tsx`) — instructs Claude to produce `/App.jsx` as the entry point, use Tailwind, and use `@/` imports.

**Mock provider** (`src/lib/provider.ts`) — `MockLanguageModel` used when `ANTHROPIC_API_KEY` is absent or set to the placeholder. `maxSteps` is capped at 4 for mock to prevent repetition (40 for real provider).

### The `@/` Import Alias — Two Different Meanings

Within **generated component code** (files living in the VFS), `@/` maps to the VFS root `/`. So a generated file at `/components/Button.jsx` is imported as `@/components/Button`.

Within the **Next.js application source** (`src/`), `@/` maps to `src/` as usual via `tsconfig.json` paths.

Don't confuse the two: when modifying generated components or the system prompt, use the VFS convention (`@/` = `/`).

### Third-Party Imports in Generated Components

`createImportMap()` in `jsx-transformer.ts` auto-resolves any unrecognized npm package name to `https://esm.sh/<package>` in the browser import map. Generated components can import arbitrary npm packages without any installation — they are fetched at runtime in the iframe.

### Tailwind CSS

Uses **Tailwind v4**, which is configured entirely via CSS imports in `src/app/globals.css` (using `@import "tailwindcss"` and `@theme` blocks). There is no `tailwind.config.js`. Theme customization goes in the `@theme inline { }` block in `globals.css`.

### Authentication

JWT in an httpOnly cookie (7-day expiry). Server actions in `src/actions/index.ts` handle `signUp`, `signIn`, `signOut`, `getUser`. `middleware.ts` enforces auth on protected routes. Anonymous users can generate but their projects are not persisted to the DB.

Anonymous work is tracked in `sessionStorage` via `anon-work-tracker.ts` (lost on tab close). This lets the UI prompt unauthenticated users to save their work when they sign up.

### Critical Files

| File | Role |
|------|------|
| `src/app/main-content.tsx` | Root layout: resizable panels, chat/preview/code tabs |
| `src/app/api/chat/route.ts` | AI streaming endpoint, tool binding, Prisma save on finish (`maxDuration = 120` for Vercel) |
| `src/lib/contexts/chat-context.tsx` | Chat state + `useAIChat` hook + `onToolCall` wiring |
| `src/lib/contexts/file-system-context.tsx` | VFS mutations + selected file state + `refreshTrigger` |
| `src/lib/file-system.ts` | `VirtualFileSystem` class |
| `src/lib/transform/jsx-transformer.ts` | Babel JSX transform → blob URL → iframe injection, esm.sh import map |
| `src/components/preview/PreviewFrame.tsx` | Iframe sandbox, entry point detection |
| `src/lib/prompts/generation.tsx` | System prompt for component generation |
| `src/lib/provider.ts` | Anthropic SDK wrapper + MockLanguageModel |
| `src/lib/auth.ts` | JWT + cookie helpers |
| `src/actions/index.ts` | Server actions for auth + project CRUD |

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` (email + bcrypt password) and `Project` (name, messages JSON, serialized VFS data). Migration files are in `prisma/migrations/`.

The Prisma client is generated to a **non-default location**: `src/generated/prisma/` (configured via `output` in `prisma/schema.prisma`). Import it from `@/generated/prisma` or via the singleton in `src/lib/prisma.ts`.

### Tests

Tests live in `__tests__/` subdirectories co-located with source, run under jsdom (`vitest.config.mts`). Coverage exists for: `VirtualFileSystem`, `jsx-transformer`, both contexts, and chat/editor components. No tests for API routes, tool builders, or auth helpers.
