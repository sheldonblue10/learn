# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

<!-- This is a learn/personal project used for experimentation and learning purposes. -->

## Commands

```bash
# Development
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run tests with Vitest
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset database (destructive)
```

To run a single test file: `npx vitest run <path/to/test>`

## Environment

Requires `ANTHROPIC_API_KEY` in `.env`. Without it, the app falls back to mock AI responses. Database is SQLite managed by Prisma.

## Architecture

**UIGen** is a Next.js 15 app that lets users generate React components via AI chat with a live preview.

### Core data flow

1. User sends a message in chat → `POST /api/chat`
2. Server calls Claude (via `@ai-sdk/anthropic`) with streaming, passing the current virtual file system state
3. Claude uses two tools to modify files: `str_replace_editor` and `file_manager`
4. Tool calls update the virtual file system, which triggers re-render of the preview iframe
5. If the user is authenticated, the project (messages + file system snapshot) is saved to SQLite

### Virtual file system (`src/lib/file-system.ts`)

All component files live in memory — nothing is written to disk. The FS state is serialized to the database for persistence and passed to the AI on each request so Claude has full context of the current codebase.

### AI tools (`src/lib/tools/`)

- `str-replace.ts` — performs targeted string replacements in files (like a minimal editor)
- `file-manager.ts` — creates or deletes files in the virtual FS

These tools are registered with the AI SDK and executed server-side during streaming.

### Preview rendering (`src/components/preview/`)

The `PreviewFrame` renders the virtual FS into an iframe. JSX is transformed to HTML using Babel standalone (`src/lib/transform/jsx-transformer.ts`) entirely in the browser.

### State management

Two React contexts in `src/lib/contexts/`:
- `FileSystemContext` — owns virtual FS state and exposes file CRUD operations
- `ChatContext` — owns message history and streams AI responses

### Authentication

JWT sessions stored in httpOnly cookies (`src/lib/auth.ts`, `src/middleware.ts`). Anonymous users can use the app without logging in; projects are only persisted for authenticated users.

### Layout

`src/app/main-content.tsx` is the top-level layout with three resizable panels (chat / code editor / preview) using `react-resizable-panels`. The Monaco editor is in `src/components/editor/`.
