# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # Install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack (localhost:3000)
npm run dev:daemon   # Start dev server in background, logs to logs.txt
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all tests with Vitest
npm run db:reset     # Reset database (destructive)
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/some-file.test.ts
```

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates and edits files in a virtual file system that is rendered live in an iframe.

### Request Flow

1. User types in chat → `ChatInterface.tsx` → POST `/api/chat`
2. `/api/chat/route.ts` streams a response from Claude (via `@ai-sdk/anthropic`) using tool calls defined in `src/lib/tools/`
3. Tool calls (`str-replace`, `file-manager`) mutate the virtual file system held in `FileSystemContext`
4. `PreviewFrame.tsx` detects changes, transforms JSX with Babel Standalone, builds an import map, and renders the result in an iframe using esm.sh CDN for React/React-DOM

### Key Abstractions

- **Virtual File System** (`src/lib/file-system.ts`) — in-memory only; serialized to JSON and persisted in the SQLite `Project.files` column via Server Actions
- **LLM Provider** (`src/lib/provider.ts`) — wraps `@ai-sdk/anthropic`; falls back to a mock when `ANTHROPIC_API_KEY` is absent
- **Contexts** (`src/lib/contexts/`) — `FileSystemContext` owns file state; `ChatContext` owns message history and streams chat updates
- **Server Actions** (`src/actions/`) — auth (sign-up/in/out), project CRUD; all database writes go through here
- **LLM Tools** (`src/lib/tools/`) — `str-replace.ts` and `file-manager.ts` define the tool schema Claude uses to edit files

### Data Model (Prisma / SQLite)

```
User  { id, email, passwordHash, createdAt, projects }
Project { id, userId, name, messages (JSON), files (JSON), createdAt, updatedAt }
```

### Auth

JWT sessions via `jose` stored in HTTP-only cookies. `src/middleware.ts` protects routes. Server Actions in `src/actions/index.ts` handle sign-up/in/out with bcrypt.

### Path Aliases

`@/*` maps to `./src/*` (configured in `tsconfig.json`).
