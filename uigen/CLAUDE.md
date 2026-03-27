# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server with Turbopack (port 3000)
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run Vitest tests
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

## Architecture

UIGen is an AI-powered React component generator. Users describe UI in chat, Claude generates code via tool calls, and components render live in an iframe — all backed by a virtual (in-memory) file system.

### Layout

`src/app/main-content.tsx` renders a resizable 3-panel layout:
- **Left (35%)**: Chat interface
- **Right (65%)**: Tabs for live Preview and Code editor (file tree + Monaco)

### AI Integration

- Route: `src/app/api/chat/route.ts` — streaming endpoint using Vercel AI SDK
- Model: Claude Haiku (`claude-haiku-4-5`) via `@ai-sdk/anthropic`; falls back to mock provider if no `ANTHROPIC_API_KEY`
- System prompt: `src/lib/prompts/generation.tsx`
- AI uses two tools:
  - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — create/edit files
  - `file_manager` (`src/lib/tools/file-manager.ts`) — rename/delete files
- Prompt caching enabled via `ephemeral` cache control headers

### State Management

Two React contexts wrap the app:
- `ChatProvider` (`src/lib/contexts/chat-context.tsx`) — chat messages and AI SDK streaming state
- `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) — virtual file system operations

### Virtual File System

`src/lib/file-system.ts` implements an in-memory tree (`VirtualFileSystem` class). No disk writes occur — all files live in memory and serialize to JSON for database storage.

### Live Preview

`src/components/preview/PreviewFrame.tsx` renders components in a sandboxed iframe. JSX is transformed client-side using `@babel/standalone` (`src/lib/transform/jsx-transformer.ts`) with import maps for module resolution. Missing files render as placeholder components.

### Auth & Database

- JWT sessions via `jose` + `bcrypt` passwords; 7-day expiry; HttpOnly cookies
- `src/lib/auth.ts` — session management; `src/middleware.ts` — API route protection
- Prisma + SQLite: `User` and `Project` models; messages and file data stored as JSON strings
- Server actions in `src/actions/` handle auth, project creation, and fetching

### Provider Selection

`src/lib/provider.ts` — returns either the real Anthropic client or a mock, depending on whether `ANTHROPIC_API_KEY` is set in `.env`.
