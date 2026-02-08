# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI (via Vercel AI SDK) to generate React components in a virtual file system with real-time preview capabilities. The project runs **without** an API key by returning static code instead of using LLM generation.

## Development Commands

### Setup & Installation
```bash
npm run setup                # Install deps, generate Prisma client, run migrations
npm install                  # Install dependencies only
npx prisma generate          # Generate Prisma client only
npx prisma migrate dev       # Run database migrations only
```

### Development
```bash
npm run dev                  # Start dev server with Turbopack
npm run dev:daemon           # Start dev server in background, logs to logs.txt
npm run build                # Production build
npm start                    # Start production server
```

### Testing & Quality
```bash
npm test                     # Run all tests with Vitest
npm run lint                 # Lint with ESLint
npx prisma migrate reset --force  # Reset database
```

Note: All npm scripts use `NODE_OPTIONS='--require ./node-compat.cjs'` for Node.js compatibility.

## High-Level Architecture

### Core Data Flow

1. **User Input** → `ChatInterface` → `/api/chat` route
2. **AI Processing** → Claude API with custom tools (`str_replace_editor`, `file_manager`)
3. **Virtual File System** → `VirtualFileSystem` class manages in-memory file tree
4. **JSX Transformation** → Babel transforms JSX/TSX to JavaScript with blob URLs
5. **Live Preview** → Generated HTML with import maps renders components in iframe

### Virtual File System

The `VirtualFileSystem` class (`src/lib/file-system.ts`) is the heart of the application:
- Maintains an in-memory tree structure using `Map<string, FileNode>`
- Supports create, read, update, delete, and rename operations
- Serializes to/from JSON for persistence in SQLite database
- No files are ever written to disk during component generation

### AI Tool Integration

The chat API route (`src/app/api/chat/route.ts`) provides Claude with two custom tools:

1. **`str_replace_editor`** (`src/lib/tools/str-replace.ts`): Create files, replace strings, insert lines
2. **`file_manager`** (`src/lib/tools/file-manager.ts`): Rename and delete files/folders

Tool calls are intercepted by `ChatContext` and applied to the `VirtualFileSystem` via the `FileSystemContext`.

### Preview System

The preview system (`src/lib/transform/jsx-transformer.ts`) transforms virtual files into executable code:

1. **Transform JSX/TSX**: Uses Babel Standalone to transform React code to JavaScript
2. **Create Import Map**: Generates ES Module import maps with:
   - External dependencies → `https://esm.sh/{package}`
   - Internal files → Blob URLs with transformed code
   - Path aliases → `@/` maps to root directory
3. **Generate HTML**: Creates standalone HTML document with:
   - Tailwind CSS CDN
   - Import map for module resolution
   - Error boundary for runtime errors
   - Syntax error display for transform failures
4. **CSS Handling**: Inlines CSS imports as `<style>` tags

The preview iframe (`src/components/preview/PreviewFrame.tsx`) updates when files change, using `srcdoc` for isolation.

### State Management

Two React contexts provide global state:

- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): Manages VirtualFileSystem and selected file state
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat` hook, syncs with file system

### Database & Persistence

- **Schema**: `User` and `Project` models in SQLite (Prisma)
- **Schema Location**: Database schema is defined in `prisma/schema.prisma` - reference this file to understand the structure of data stored in the database
- **Client Location**: Prisma client output is `src/generated/prisma` (not default `node_modules`)
- **Project Data**: Stored as JSON strings:
  - `messages`: Chat history
  - `data`: Serialized VirtualFileSystem
- **Anonymous Users**: Can work without auth; work saved in sessionStorage via `anon-work-tracker.ts`

### Authentication

Simple JWT-based auth (`src/lib/auth.ts`):
- Passwords hashed with bcrypt
- Sessions stored in httpOnly cookies
- Middleware (`src/middleware.ts`) protects project routes
- Users can work anonymously (no persistence)

## Code Organization

```
src/
├── actions/           # Server actions for project CRUD
├── app/              # Next.js 15 App Router pages and API routes
│   ├── api/chat/     # Main chat endpoint with AI streaming
│   └── [projectId]/  # Dynamic project page
├── components/
│   ├── auth/         # Sign-in/sign-up forms
│   ├── chat/         # Chat interface, message list, markdown renderer
│   ├── editor/       # Code editor (Monaco) and file tree
│   ├── preview/      # Preview iframe component
│   └── ui/           # shadcn/ui components (Radix UI)
├── hooks/            # Custom React hooks
├── lib/
│   ├── contexts/     # React contexts (FileSystem, Chat)
│   ├── tools/        # AI tool definitions
│   ├── transform/    # JSX transformation and preview HTML generation
│   ├── prompts/      # System prompts for AI
│   ├── auth.ts       # JWT authentication utilities
│   ├── file-system.ts # VirtualFileSystem class
│   ├── prisma.ts     # Prisma client singleton
│   └── provider.ts   # AI provider configuration (Anthropic/Mock)
└── generated/prisma/ # Generated Prisma client
```

## Important Implementation Details

### Path Resolution

The system supports multiple import path styles:
- Absolute: `/App.jsx`
- Relative: `./components/Button`
- Aliased: `@/components/Button` (@ maps to root)

All paths in `VirtualFileSystem` are normalized to absolute paths starting with `/`.

### Mock Provider

When `ANTHROPIC_API_KEY` is not set, `getLanguageModel()` returns a mock provider that generates static demo code. The `maxSteps` is reduced to 4 for mock mode to prevent repetition.

### Test Setup

Tests use Vitest with jsdom environment:
- Path aliases configured via `vite-tsconfig-paths`
- React Testing Library for component tests
- Tests colocated with source in `__tests__` directories

### Tailwind CSS v4

This project uses Tailwind CSS v4 (PostCSS-based), not the traditional v3 with `tailwind.config.js`. Configuration is in `postcss.config.mjs`.

## Common Pitfall: Prisma Client Location

Prisma client is generated to `src/generated/prisma`, NOT `node_modules/@prisma/client`. Always import from `@/lib/prisma` which uses the correct path.

## Coding Conventions

- Use comments sparingly. Only comment complex code.
