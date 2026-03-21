# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (localhost:3000)
npm run dev

# Build
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Reset the database
npm run db:reset
```

The dev server requires `NODE_OPTIONS="--require ./node-compat.cjs"` (handled by the npm script via `cross-env`).

## Environment

Copy `.env` and add `ANTHROPIC_API_KEY` to use the real Claude model. Without it, the app falls back to a `MockLanguageModel` in `src/lib/provider.ts` that returns hardcoded Counter/Card/Form components.

The real model used is `claude-haiku-4-5` (defined in `src/lib/provider.ts`).

## Architecture

### Overview

UIGen is a Next.js 15 App Router app where users describe React components in a chat interface and see them rendered live. All generated files live in an **in-memory virtual file system** — nothing is written to disk.

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` class that manages a tree of `FileNode` objects in a `Map<string, FileNode>`. The file system is serialized to JSON for storage in the database (`Project.data`) and for passing to the AI API route on each request.

`src/lib/contexts/file-system-context.tsx` — React context that wraps `VirtualFileSystem` and exposes it to the UI. Also handles `handleToolCall`, which applies AI tool call results (file creates, edits, renames, deletes) into the VFS and triggers re-renders via a `refreshTrigger` counter.

### AI Generation Pipeline

1. Client sends messages + serialized VFS files to `POST /api/chat` (`src/app/api/chat/route.ts`)
2. Server deserializes the VFS, calls `streamText` (Vercel AI SDK) with two tools: `str_replace_editor` and `file_manager`
3. AI streams back tool calls that create/edit/delete virtual files
4. Client receives the stream via `useChat` (chat context), and each tool call is forwarded to `handleToolCall` in `FileSystemContext`
5. `PreviewFrame` re-renders the iframe whenever `refreshTrigger` increments

### Preview Rendering

`src/components/preview/PreviewFrame.tsx` — renders an `<iframe srcdoc>` sandbox. On each VFS change:
- `src/lib/transform/jsx-transformer.ts` uses `@babel/standalone` to transpile all JSX/TSX files in-browser
- An ES module import map is injected so components can import each other via blob URLs
- Entry point priority: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx`

### Auth & Persistence

- JWT-based sessions via `jose`, stored in an httpOnly cookie (`auth-token`). Logic in `src/lib/auth.ts`.
- `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes.
- Anonymous users can use the app freely; only authenticated users get project persistence.
- Prisma with SQLite (`prisma/dev.db`). Models: `User` and `Project`. `Project.messages` and `Project.data` are JSON strings storing the full chat history and serialized VFS respectively.
- Prisma client is generated into `src/generated/prisma/` (non-standard output path — set in `prisma/schema.prisma`).

### Key Contexts

- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — VFS state and file operations
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK `useChat`, forwards tool calls to FileSystemContext

### UI Components

`src/components/ui/` contains shadcn/ui primitives. Feature components are under `src/components/chat/`, `src/components/editor/`, and `src/components/preview/`.

### Testing

Vitest with jsdom + React Testing Library. Tests co-located under `__tests__/` subdirectories next to the components they test.

## Database

The database schema is defined in `prisma/schema.prisma`. Reference it whenever you need to understand the structure of data stored in the database.

## Code Style

Use comments sparingly. Only comment complex or non-obvious code.
