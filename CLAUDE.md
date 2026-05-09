# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, Claude generates them via tool calls, and the preview renders live in an iframe sandbox. Works without an API key (falls back to a mock provider returning canned components).

## Commands

```bash
npm run setup       # install deps + prisma generate + prisma migrate dev
npm run dev         # start dev server (Turbopack) at http://localhost:3000
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

Copy `.env` and set `ANTHROPIC_API_KEY`. Without a valid key the app uses a mock provider that returns canned Counter/Form/Card components.

## Architecture

### Data Flow

1. User submits chat → POST `/api/chat/route.ts`
2. Claude streams a response using two tools: `str_replace_editor` (create/edit/insert/view files) and `file_manager` (rename/delete)
3. Tool calls are forwarded to `FileSystemContext` via `onToolCall` → mutates the in-memory `VirtualFileSystem`
4. `PreviewFrame` detects changes and re-renders the iframe
5. `jsx-transformer.ts` uses Babel.standalone to transform JSX → blob URL → injected into the iframe sandbox
6. On finish (authenticated users only): VFS + messages are serialized to Prisma (SQLite)

### Key Abstractions

**`VirtualFileSystem`** (`src/lib/file-system.ts`) — in-memory file tree (`Map<path, FileNode>`). All AI file operations go through this; nothing is written to disk. Serializes to/from JSON for DB persistence.

**`ChatContext`** + **`FileSystemContext`** (`src/lib/contexts/`) — decouple chat state from file state. `ChatContext` drives the AI SDK hook; `FileSystemContext` owns the VFS and exposes `onToolCall` for AI mutations.

**Tool builders** (`src/lib/tools/`) — `buildStrReplaceTool` and `buildFileManagerTool` wrap file ops in Zod schemas and wire them into the AI SDK's tool list.

**System prompt** (`src/lib/prompts/generation.tsx`) — instructs Claude to produce `/App.jsx` as the entry point, use Tailwind, and use `@/` imports.

**Mock provider** (`src/lib/provider.ts`) — `MockLanguageModel` used when `ANTHROPIC_API_KEY` is absent or set to the placeholder.

### Authentication

JWT in an httpOnly cookie (7-day expiry). Server actions in `src/actions/index.ts` handle `signUp`, `signIn`, `signOut`, `getUser`. `middleware.ts` enforces auth on protected routes. Anonymous users can generate but their projects are not persisted to the DB (tracked locally via `anon-work-tracker.ts`).

### Critical Files

| File | Role |
|------|------|
| `src/app/main-content.tsx` | Root layout: resizable panels, chat/preview/code tabs |
| `src/app/api/chat/route.ts` | AI streaming endpoint, tool binding, Prisma save on finish |
| `src/lib/contexts/chat-context.tsx` | Chat state + `useAIChat` hook + tool call handler |
| `src/lib/contexts/file-system-context.tsx` | VFS mutations + selected file state |
| `src/lib/file-system.ts` | `VirtualFileSystem` class |
| `src/lib/transform/jsx-transformer.ts` | Babel JSX transform → blob URL → iframe injection |
| `src/components/preview/PreviewFrame.tsx` | Iframe sandbox, entry point detection |
| `src/lib/prompts/generation.tsx` | System prompt for component generation |
| `src/lib/provider.ts` | Anthropic SDK wrapper + MockLanguageModel |
| `src/lib/auth.ts` | JWT + cookie helpers |
| `src/actions/index.ts` | Server actions for auth + project CRUD |

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` (email + bcrypt password) and `Project` (name, messages JSON, serialized VFS data). Migration files are in `prisma/migrations/`.
