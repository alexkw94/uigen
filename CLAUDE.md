# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # Install deps + generate Prisma client + run migrations
npm run dev            # Start dev server (Turbopack) at localhost:3000
npm run dev:daemon     # Same but backgrounded, logs to logs.txt
npm run build          # Production build
npm run lint           # ESLint
npm run test           # Run all tests (Vitest)
npx vitest run path/to/file.test.tsx  # Run a single test file
npm run db:reset       # Reset database (destructive)
npx prisma migrate dev # Apply schema changes and regenerate client
npx prisma generate    # Regenerate Prisma client after schema changes
```

`ANTHROPIC_API_KEY` in `.env` is optional — omitting it activates a mock provider that returns static component code.

## Architecture

UIGen is a Next.js 15 (App Router) application where users describe React components in a chat interface, Claude generates them using tool calls, and they render live in a sandboxed iframe — all without writing files to disk.

### Key data flow

1. User sends a message → `POST /api/chat` (streaming via Vercel AI SDK)
2. Claude receives the system prompt (`src/lib/prompts/generation.tsx`) and the current VFS state
3. Claude calls tools (`str_replace_editor`, `file_manager`) to create/edit files
4. Tool results update `FileSystemContext` (in-memory VFS)
5. VFS changes trigger `PreviewFrame` to re-transform and re-render
6. On completion, if authenticated, the project (messages + VFS) is serialized to JSON and saved to SQLite

### Virtual File System (VFS)

`src/lib/file-system.ts` — in-memory Map-based tree. No real disk I/O. Files are serialized as JSON into the `Project.data` column. All Claude-generated files live here. The VFS is exposed to components via `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`).

### Live Preview

`src/components/preview/PreviewFrame.tsx` + `src/lib/transform/jsx-transformer.ts`:
- Babel standalone transforms JSX → ES modules
- Each VFS file becomes a blob URL
- An import map wires `@/` aliases and local imports together
- React/ReactDOM come from esm.sh CDN; unknown third-party packages also fall back to esm.sh
- Tailwind CSS is loaded from CDN (`cdn.tailwindcss.com`)
- The whole thing renders in a sandboxed `<iframe>`

### AI tools

Defined in `src/lib/tools/`:
- `str-replace.ts` — `view | create | str_replace | insert | undo_edit` on VFS files
- `file-manager.ts` — `rename | delete` on VFS files

These are passed to `streamText()` in `src/app/api/chat/route.ts`. Claude must always produce `/App.jsx` as the entry point.

### Authentication

JWT sessions (`src/lib/auth.ts`) stored in an `auth-token` httpOnly cookie (7-day expiry). Server Actions in `src/actions/` handle sign-up/sign-in/sign-out. Middleware (`src/middleware.ts`) verifies the JWT on protected routes.

Anonymous users can generate components; their work is tracked in sessionStorage (`src/lib/anon-work-tracker.ts`) and migrated into a new project on sign-up/sign-in.

### Database

SQLite via Prisma. Schema in `prisma/schema.prisma`. Two models: `User` (email + bcrypt password) and `Project` (name, optional userId, `messages` JSON string, `data` JSON string for VFS). Prisma client is generated to `src/generated/prisma/`.

### Provider

`src/lib/provider.ts` exports the AI provider. Uses `claude-haiku-4-5` via `@ai-sdk/anthropic`. Falls back to a mock that returns static multi-step component code when `ANTHROPIC_API_KEY` is absent. Max 40 tool-use steps (4 for mock), 120s API timeout.
