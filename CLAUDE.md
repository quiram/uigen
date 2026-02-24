# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup
npm run setup        # installs deps, generates Prisma client, runs migrations

# Development
npm run dev          # starts Next.js with Turbopack at http://localhost:3000

# Build & Production
npm run build
npm start

# Testing
npm test             # run all tests with Vitest
npx vitest run src/path/to/file.test.tsx  # run a single test file

# Linting
npm run lint

# Database
npx prisma migrate dev   # apply schema changes
npm run db:reset         # reset DB to clean state (destructive)
npx prisma generate      # regenerate Prisma client after schema changes
```

Set `ANTHROPIC_API_KEY` in the environment to use the real Claude API. Without it, a mock provider is used that returns static sample components.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components via chat; Claude generates code that populates a virtual file system; the preview renders the result in a sandboxed iframe.

### Request Flow

```
User message → POST /api/chat → Claude (streamText + tools)
                                        ↓
                          str_replace_editor / file_manager tool calls
                                        ↓
                          FileSystemContext updates (client-side)
                                        ↓
                          PreviewFrame re-renders iframe
```

### Key Subsystems

**Virtual File System** (`src/lib/file-system.ts`)
In-memory tree managed by `VirtualFileSystem`. Never writes to disk. Serializes to JSON for DB persistence and API transfer. Mutations happen on the client via `FileSystemContext` (`src/lib/contexts/`).

**AI Integration** (`src/app/api/chat/route.ts`, `src/lib/provider.ts`)
Uses Vercel AI SDK (`ai`) with `@ai-sdk/anthropic`. The backend streams responses via `streamText`, Claude invokes two tools (`str_replace_editor` for file edits, `file_manager` for file CRUD). Tool definitions live in `src/lib/tools/`. On completion, the project (messages + file system) is serialized and saved to SQLite via Prisma.

**Live Preview** (`src/components/preview/PreviewFrame.tsx`)
Transforms JSX/TSX with `@babel/standalone` in the browser, injects an import map pointing React to esm.sh CDN, and renders in a sandboxed iframe. No server-side bundling involved.

**Authentication** (`src/lib/auth.ts`, `src/actions/`)
JWT-based sessions stored in HTTP-only cookies (7-day expiry). Passwords hashed with bcrypt. Server actions (`signUp`, `signIn`, `signOut`, `getUser`) handle the auth lifecycle. Middleware (`src/middleware.ts`) protects routes.

**UI Layout** (`src/app/main-content.tsx`)
Three resizable panels via `react-resizable-panels`: Chat (left, 35%) | Preview+Code tabs (right, 65%). Code tab splits into file tree (30%) and Monaco editor (70%).

### Data Model

```prisma
User  { id, email, password }
Project { id, name, userId, messages JSON, data JSON }
```

`messages` stores the full chat history; `data` stores the serialized virtual file system. Both are stored as JSON strings.

## Code Style

Avoid comments. Code should be self-documenting. Only add comments for complex logic that wouldn't be obvious to someone with the relevant domain knowledge.

### Path Aliases

`@/*` maps to `src/*` throughout the codebase (configured in `tsconfig.json` and Vitest).
