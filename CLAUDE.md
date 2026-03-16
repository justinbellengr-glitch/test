# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Start dev server (Turbopack) at http://localhost:3000
npm run build       # Production build
npm run lint        # ESLint
npm test            # Run all Vitest tests
npm test -- --run src/path/to/test.ts  # Run a single test file
npm run db:reset    # Drop and recreate the SQLite database
```

Tests use Vitest with jsdom and React Testing Library. Test files live in `__tests__/` subdirectories alongside the code they test.

## Architecture

UIGen is a Next.js 15 App Router app where users describe React components in a chat and see them rendered live.

### Virtual File System

All generated code lives in a **virtual file system** (`src/lib/file-system.ts` — `VirtualFileSystem` class), never written to disk. It's a tree of `FileNode` objects held in a `Map<string, FileNode>`. The VFS serializes to plain JSON for transport/storage and reconstructs via `deserializeFromNodes()`.

For authenticated users, the VFS state plus chat messages are persisted in the Prisma `Project` model as two JSON string columns (`data` and `messages`).

### AI Integration Flow

1. **Chat context** (`src/lib/contexts/chat-context.tsx`) wraps the Vercel AI SDK's `useChat`, sending the serialized VFS + `projectId` in every request body.
2. **API route** (`src/app/api/chat/route.ts`) reconstructs the VFS, calls `streamText` with two tools, and on completion saves the updated state to Prisma (authenticated users only).
3. **Two AI tools** manipulate the VFS:
   - `str_replace_editor` (built in `src/lib/tools/str-replace.ts`): `create`, `str_replace`, `insert` commands
   - `file_manager` (built in `src/lib/tools/file-manager.ts`): `rename`, `delete` commands
4. **Tool call interception**: `FileSystemContext.handleToolCall` (`src/lib/contexts/file-system-context.tsx`) intercepts tool calls on the client side to update the React state that drives the preview.

### Live Preview

`src/lib/transform/jsx-transformer.ts` compiles the virtual files in-browser:
- Transforms JSX/TSX with `@babel/standalone`
- Creates blob URLs for each file
- Builds an ES module import map (local files → blob URLs, third-party packages → `esm.sh`)
- Renders into an `<iframe>` via `PreviewFrame` component
- Tailwind CSS loaded from CDN inside the preview iframe

The generated app must have a root `/App.jsx` as its entrypoint (enforced in the system prompt at `src/lib/prompts/generation.tsx`). Local imports use the `@/` alias (e.g., `@/components/Button`).

### AI Provider

`src/lib/provider.ts` returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set, or a `MockLanguageModel` otherwise. The mock generates static counter/form/card components to allow development without an API key.

### Auth

JWT-based auth using `jose`, stored in an httpOnly cookie. `src/lib/auth.ts` is `server-only`. Session is 7 days. `src/middleware.ts` handles route protection.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` and `Project`. The generated Prisma client outputs to `src/generated/prisma/`.

### Context Tree

```
FileSystemProvider   ← manages VirtualFileSystem instance + selected file
  └── ChatProvider   ← wraps useAIChat, intercepts tool calls into FileSystemProvider
        └── UI components (ChatInterface, PreviewFrame, CodeEditor, FileTree)
```
