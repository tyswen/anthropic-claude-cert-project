# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** — an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates code via tool calls, and the result appears instantly in an iframe preview. No components are written to disk — everything runs through a virtual file system.

## Commands

```bash
npm run setup       # First-time init: install deps, prisma generate, run migrations
npm run dev         # Start dev server (Turbopack)
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Vitest unit tests
npm run db:reset    # Drop and re-run all migrations
```

Run a single test file:
```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

## Environment

Requires `.env` with `ANTHROPIC_API_KEY`. Without it, the app falls back to a static mock provider (`src/lib/provider.ts`) — useful for UI development without burning API credits.

## Architecture

**Stack:** Next.js 15 (App Router) · React 19 · TypeScript · Tailwind CSS v4 · Prisma + SQLite · Vercel AI SDK · Monaco Editor · shadcn/ui

### Data Flow

```
User chat input
  → ChatInterface (useAIChat hook, Vercel AI SDK)
  → POST /api/chat
  → Claude API via streamText (or mock provider)
  → Tool calls: str_replace_editor / file_manager
  → VirtualFileSystem (in-memory, no disk writes)
  → FileSystemContext state update
  → PreviewFrame (iframe) re-renders via jsx-transformer
  → Project persisted to SQLite (authenticated users only)
```

### Key Modules

| Path | Role |
|---|---|
| `src/app/api/chat/route.ts` | Streaming AI endpoint; orchestrates tool use and project persistence |
| `src/lib/provider.ts` | Selects Claude vs. mock provider based on env |
| `src/lib/file-system.ts` | `VirtualFileSystem` class — in-memory file store |
| `src/lib/contexts/` | `ChatContext` and `FileSystemContext` — app-wide state |
| `src/lib/prompts/generation.tsx` | System prompt given to Claude for code generation |
| `src/lib/tools/` | `str_replace_editor` and `file_manager` tools Claude calls to create/edit/delete files |
| `src/lib/transform/jsx-transformer.ts` | Converts JSX → HTML for the preview iframe |
| `src/actions/` | Next.js Server Actions for auth and project CRUD |
| `src/middleware.ts` | JWT-based auth middleware |

### UI Layout

Two-panel split: chat on the left (35%), code editor + preview on the right (65%). `src/app/main-content.tsx` owns the layout; `src/components/preview/PreviewFrame.tsx` hosts the iframe.

### Authentication

JWT sessions via `jose`; passwords hashed with `bcrypt`. Session stored in a cookie. Projects support anonymous ownership (`userId` is optional in the schema) — anonymous work is tracked via `src/lib/anon-work-tracker.ts` so it can be claimed after sign-up.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` and `Project`. Both `messages` (chat history) and `data` (virtual file system snapshot) are stored as serialized JSON strings on `Project`.

Generated Prisma client outputs to `src/generated/prisma` (non-standard location — import from there, not `@prisma/client`).

### Testing

Tests live in `__tests__/` subdirectories alongside the components they cover. Vitest runs in a jsdom environment with path aliases matching the tsconfig (`@/*` → `src/*`).
