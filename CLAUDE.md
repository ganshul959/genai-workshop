# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack at http://localhost:3000
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run Vitest tests (jsdom environment)
npx prisma migrate dev   # Run new DB migrations
npm run db:reset     # Reset DB (destructive)
```

Run a single test file:
```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. Without it, the app uses a `MockLanguageModel` that returns static components — useful for development without an API key.

## Architecture

**UIGen** is an AI-powered React component generator. Users describe components in a chat; Claude generates/edits files in a virtual (in-memory) file system; a sandboxed iframe renders the result live.

### Key data flow

1. **Chat** (`/src/app/api/chat/route.ts`) — POST handler receives messages + serialized `FileNode` tree. It reconstructs a `VirtualFileSystem`, calls `streamText` (Vercel AI SDK) with two tools (`str_replace_editor`, `file_manager`), and streams back tool calls + text. On finish, saves to SQLite if the user is authenticated.

2. **VirtualFileSystem** (`/src/lib/file-system.ts`) — in-memory tree of `FileNode` objects, never touches disk. Supports create/read/update/delete/rename with path normalization. Serializes to a plain `Record<string, FileNode>` for API transport and Prisma storage.

3. **FileSystemContext** (`/src/lib/contexts/file-system-context.tsx`) — React context wrapping `VirtualFileSystem`. `handleToolCall` applies AI tool calls (`str_replace_editor` and `file_manager`) to the in-memory FS and triggers re-renders via `refreshTrigger`.

4. **Preview** (`/src/components/preview/PreviewFrame.tsx`) — on every `refreshTrigger`, transforms all `.jsx/.tsx` files with Babel standalone, builds an ES module import map (local files → blob URLs, third-party packages → `esm.sh`), and sets `srcdoc` on a sandboxed iframe. Tailwind CSS is injected via CDN.

5. **AI tools** — `str_replace_editor` (create / str_replace / insert / view) and `file_manager` (rename / delete). Both operate on a server-side `VirtualFileSystem` instance per request; the client mirrors changes via `handleToolCall`.

6. **Provider** (`/src/lib/provider.ts`) — returns `anthropic("claude-haiku-4-5")` if `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that produces static counter/form/card components.

### Auth

JWT-based sessions via `jose`. Stored in an HTTP-only cookie. `bcrypt` for password hashing. `middleware.ts` guards `/api/projects` and `/api/filesystem`. Anonymous users can use the app but projects are not persisted.

### Persistence

Prisma + SQLite (`prisma/dev.db`). Two models: `User` and `Project`. `Project.messages` and `Project.data` are JSON-stringified columns storing the full chat history and serialized `VirtualFileSystem` respectively. Prisma client is generated to `src/generated/prisma/`.

### Testing

Vitest + Testing Library (jsdom). Tests live alongside source in `__tests__/` subdirectories.
