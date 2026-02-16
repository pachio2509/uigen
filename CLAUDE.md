# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI (via Anthropic API) to generate React components in real-time, displays them in a live preview using a virtual file system, and supports user authentication with project persistence.

## Development Commands

### Core Commands
```bash
npm run setup         # Install dependencies, generate Prisma client, run migrations (FIRST TIME SETUP)
npm run dev          # Start development server with Turbopack (http://localhost:3000)
npm run dev:daemon   # Start dev server in background, writing logs to logs.txt
npm run build        # Create production build
npm run start        # Start production server
npm run lint         # Run ESLint
npm run test         # Run Vitest tests
npm run db:reset     # Reset database (force)
```

### Database Commands
```bash
npx prisma generate          # Generate Prisma client
npx prisma migrate dev       # Run migrations in development
npx prisma studio            # Open Prisma Studio GUI
```

### Running Tests
```bash
npm run test                 # Run all tests
npm run test -- --watch      # Run tests in watch mode
npm run test -- <file-path>  # Run specific test file
```

## Architecture

### Tech Stack
- **Framework**: Next.js 15 (App Router)
- **React**: 19.0.0
- **TypeScript**: v5
- **Styling**: Tailwind CSS v4
- **Database**: Prisma with SQLite (`prisma/dev.db`)
- **AI**: Anthropic Claude (Haiku 4.5) via Vercel AI SDK
- **Auth**: JWT with jose library
- **Code Editor**: Monaco Editor
- **Testing**: Vitest with jsdom

### Key Architectural Concepts

#### Virtual File System
The application uses an in-memory virtual file system (`VirtualFileSystem` class in `src/lib/file-system.ts`) that mimics a real filesystem but **never writes files to disk**. This enables:
- AI-generated code to be created, edited, and previewed without file I/O
- Serialization to database for project persistence
- Real-time updates in the preview iframe

**Critical Methods:**
- `createFile(path, content)` - Create file with auto-parent-directory creation
- `updateFile(path, content)` - Update file content
- `serialize()` / `deserializeFromNodes()` - Save/load from database
- `viewFile(path)` - View file with line numbers (used by AI tools)
- `replaceInFile(path, oldStr, newStr)` - String replacement (AI tool)

#### AI Tool Integration
The AI uses two custom tools to manipulate the virtual filesystem:

1. **str_replace_editor** (`src/lib/tools/str-replace.ts`)
   - Commands: `create`, `str_replace`, `insert`, `view`
   - Allows Claude to create files, replace strings, insert lines, and view files
   - Returns formatted output with line numbers

2. **file_manager** (`src/lib/tools/file-manager.ts`)
   - Commands: `rename`, `delete`, `list`
   - File/directory management operations

#### Live Preview System
The preview system (`src/components/preview/PreviewFrame.tsx` and `src/lib/transform/jsx-transformer.ts`) works through:

1. **JSX Transformation**: Uses `@babel/standalone` to transform JSX/TSX to JavaScript in the browser
2. **Import Map Generation**: Creates ES Module import maps with blob URLs for each transformed file
3. **Path Alias Resolution**: Supports `@/` aliases (maps to `/` root)
4. **CSS Handling**: Collects CSS imports and injects them as `<style>` tags
5. **Error Display**: Shows syntax errors inline in the preview iframe
6. **Entry Point Detection**: Automatically finds `/App.jsx`, `/App.tsx`, or first `.jsx/.tsx` file

The preview runs in an iframe with `srcdoc` containing the generated HTML with import maps.

#### Mock Provider
When `ANTHROPIC_API_KEY` is not set, a `MockLanguageModel` (`src/lib/provider.ts`) simulates AI responses with predefined component templates (counter, form, card). This allows the app to run without an API key for demos.

#### Authentication Flow
- JWT-based sessions (`src/lib/auth.ts`) using `jose` library
- Cookie-based tokens (`auth-token`) with 7-day expiry
- Middleware protection (`src/middleware.ts`) for project routes
- Anonymous user support (projects saved without userId)
- Authenticated users can save/load projects from SQLite database

#### Database Schema
```prisma
User {
  id: String (cuid)
  email: String (unique)
  password: String (bcrypt hashed)
  projects: Project[]
}

Project {
  id: String (cuid)
  name: String
  userId: String? (optional - supports anonymous)
  messages: String (JSON array)
  data: String (JSON - serialized VirtualFileSystem)
  user: User?
}
```

Projects store:
- `messages`: Full chat history (for resuming conversations)
- `data`: Serialized file system state (all generated files)

#### React Context Architecture
1. **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`)
   - Manages VirtualFileSystem instance
   - Handles file CRUD operations
   - Auto-selects first file or `/App.jsx` for editor
   - Processes AI tool calls to update files in real-time

2. **ChatContext** (`src/lib/contexts/chat-context.tsx`)
   - Manages chat messages and streaming
   - Integrates with Vercel AI SDK's `useChat`
   - Handles project saving after AI responses

### Directory Structure
- `src/app/` - Next.js App Router pages and API routes
  - `api/chat/route.ts` - Chat API endpoint with AI streaming
  - `[projectId]/page.tsx` - Project-specific page (authenticated)
- `src/components/` - React components
  - `ui/` - shadcn/ui components (Radix UI primitives)
  - `chat/` - Chat interface, message list, markdown renderer
  - `editor/` - Monaco code editor, file tree
  - `preview/` - Preview iframe component
  - `auth/` - Sign in/up forms, auth dialog
- `src/lib/` - Core utilities
  - `file-system.ts` - VirtualFileSystem class
  - `transform/jsx-transformer.ts` - Babel transformation and import map generation
  - `tools/` - AI tool definitions (str-replace, file-manager)
  - `contexts/` - React contexts
  - `prompts/` - AI system prompts
  - `auth.ts` - JWT session management
  - `provider.ts` - AI model provider (real or mock)
  - `prisma.ts` - Prisma client singleton
- `src/actions/` - Server actions
  - `create-project.ts`, `get-project.ts`, `get-projects.ts`
- `src/hooks/` - Custom React hooks
- `prisma/` - Database schema and migrations

### Path Aliases
- `@/*` maps to `./src/*` (configured in `tsconfig.json`)
- Used throughout the codebase for cleaner imports

### Node Compatibility
The project uses `node-compat.cjs` (required via `NODE_OPTIONS`) to polyfill Node.js modules for browser-based Babel transformation.

### Environment Variables
- `ANTHROPIC_API_KEY` - Optional. If not set, uses mock provider with static responses
- `JWT_SECRET` - Used for session signing (defaults to `"development-secret-key"`)
- Database URL is hardcoded in `prisma/schema.prisma` as `file:./dev.db`

### Testing
- Uses Vitest with React Testing Library
- Tests located in `__tests__/` directories next to source files
- Example: `src/components/chat/__tests__/ChatInterface.test.tsx`
- jsdom environment for DOM testing
- `vite-tsconfig-paths` plugin for path alias support in tests

### Important Patterns

**Turbopack**: Development server uses `--turbopack` flag for faster builds

**Server-Only Code**: Auth functions use `"server-only"` package to prevent client-side bundling

**Anonymous Work Tracking**: `src/lib/anon-work-tracker.ts` tracks anonymous user projects in localStorage

**Markdown Rendering**: Chat uses `react-markdown` with `@tailwindcss/typography` for formatted AI responses

**File System Updates**: The FileSystemContext listens to AI tool calls and updates the VFS in real-time, triggering re-renders of the preview and editor
