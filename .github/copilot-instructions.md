# GitHub Copilot Instructions for Gemini CLI

This document provides essential guidance for AI coding agents working on the
Gemini CLI codebase. It covers architecture, workflows, conventions, and
integration points to maximize productivity and code quality.

## Project Overview

Gemini CLI is an open-source AI agent that brings the power of Google's Gemini
directly into the terminal. It's built with TypeScript, Node.js, React (via
Ink), and follows functional programming patterns.

**Key Repository Information:**

- **Main branch:** `main`
- **License:** Apache 2.0
- **Node.js requirement:** `>=20.0.0` (development requires `~20.19.0`)
- **Package manager:** npm with workspaces
- **Build system:** Custom scripts + esbuild
- **Testing framework:** Vitest
- **UI framework:** Ink (React for CLI)

## Architecture Overview

### Core Components

The project is organized as a monorepo with distinct packages:

1. **`packages/cli`** - User-facing CLI interface
   - Handles user input and output
   - Terminal UI rendering with Ink (React-based)
   - History management
   - Theme and customization
   - Configuration settings

2. **`packages/core`** - Backend logic
   - Gemini API client and communication
   - Prompt construction and management
   - Tool registration and execution
   - State management for conversations
   - Server-side configuration

3. **`packages/core/src/tools/`** - Tool system
   - File system operations (`read-file.ts`, `write-file.ts`, `edit.ts`,
     `smart-edit.ts`)
   - Shell execution (`shell.ts`)
   - Web fetching and search (`web-fetch.ts`, `web-search.ts`)
   - MCP (Model Context Protocol) integration (`mcp-client.ts`, `mcp-tool.ts`)
   - Advanced search (`grep.ts`, `ripGrep.ts`, `glob.ts`)
   - Memory and todos (`memoryTool.ts`, `write-todos.ts`)

4. **`packages/a2a-server`** - A2A server implementation (Experimental)

5. **`packages/test-utils`** - Shared testing utilities

6. **`packages/vscode-ide-companion`** - VS Code extension

### Interaction Flow

```
User Input → CLI (packages/cli) → Core (packages/core)
  → Gemini API → Tool Execution (with confirmation) → Response → CLI → User
```

### Key Design Principles

- **Modularity:** Clear separation between CLI frontend and Core backend
- **Extensibility:** Plugin system via MCP servers and custom extensions
- **Security:** Sandboxing support (macOS Seatbelt, Docker, Podman)
- **User experience:** Rich terminal UI with confirmation for sensitive
  operations

## Development Workflow

### Building and Testing

**Always run the complete preflight check before submitting changes:**

```bash
npm run preflight
```

This single command:

- Cleans the workspace
- Installs dependencies
- Formats code with Prettier
- Lints all code (including TypeScript, integration tests, and scripts)
- Builds all packages
- Runs type checking
- Executes all unit and script tests

### Individual Commands

While `npm run preflight` is comprehensive, you can run steps individually
during development:

```bash
npm run build          # Build all packages
npm run test           # Run unit tests
npm run typecheck      # TypeScript type checking
npm run lint           # ESLint all code
npm run lint:fix       # Auto-fix linting issues + format
npm run format         # Format with Prettier
npm start              # Run CLI from source
```

### Integration Tests

```bash
npm run test:e2e                              # E2E tests without sandbox
npm run test:integration:sandbox:docker       # E2E with Docker sandbox
npm run test:integration:sandbox:podman       # E2E with Podman sandbox
```

### Debugging

**VS Code:** Press `F5` to launch with debugger attached, or use:

```bash
npm run debug          # Attach debugger
```

**React DevTools** (for Ink UI debugging):

```bash
DEV=true npm start
# Then in another terminal:
npx react-devtools@4.28.5
```

**Sandbox debugging:**

```bash
DEBUG=1 gemini
```

## TypeScript/JavaScript Conventions

### Prefer Plain Objects Over Classes

**Always use plain JavaScript objects with TypeScript interfaces/types instead
of classes.**

**Why:**

- Seamless React integration (no hidden state)
- Reduced boilerplate
- Enhanced readability and predictability
- Simplified immutability
- Better JSON serialization

**Do this:**

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

function createUser(name: string, email: string): User {
  return {
    id: generateId(),
    name,
    email,
  };
}
```

**Don't do this:**

```typescript
class User {
  constructor(
    public id: string,
    public name: string,
    public email: string,
  ) {}
}
```

### ES Module Encapsulation

Use `import`/`export` for public/private APIs instead of class visibility
modifiers:

- **Exported:** Public API of the module
- **Not exported:** Private to the module

If you need to test an unexported function, consider extracting it to a separate
module with a well-defined public API.

### Type Safety

**Avoid `any` - Prefer `unknown`:**

```typescript
// Bad
function process(value: any) {
  return value.toString();
}

// Good
function process(value: unknown) {
  if (typeof value === 'string') {
    return value.toUpperCase();
  } else if (typeof value === 'number') {
    return value.toString();
  }
  throw new Error('Unsupported type');
}
```

**Use Type Assertions Sparingly:**

- Only when you have more information than the compiler
- Never to bypass type checking during testing
- If you need type assertions in tests, it's a code smell - refactor instead

**Exhaustive Switch Statements:**

Use the `checkExhaustive` helper (found in `packages/cli/src/utils/checks.ts`)
in the default clause:

```typescript
import { checkExhaustive } from './utils/checks';

function handleAction(action: Action) {
  switch (action.type) {
    case 'create':
      return createHandler(action);
    case 'update':
      return updateHandler(action);
    case 'delete':
      return deleteHandler(action);
    default:
      checkExhaustive(action.type);
  }
}
```

### Functional Programming

**Embrace array operators** (`.map()`, `.filter()`, `.reduce()`, `.slice()`,
`.sort()`) for immutable, declarative transformations:

```typescript
// Prefer this
const activeUsers = users
  .filter((user) => user.active)
  .map((user) => user.name)
  .sort();

// Over this
const activeUsers = [];
for (const user of users) {
  if (user.active) {
    activeUsers.push(user.name);
  }
}
activeUsers.sort();
```

### Style Requirements

- **Flag names:** Use hyphens, not underscores (e.g., `--my-flag`, not
  `--my_flag`)
- **Comments:** Only write high-value comments. Avoid commenting the obvious or
  "talking to users" through comments.
- **Imports:** Follow ESLint restrictions on relative imports between packages

## React Conventions (Ink-based CLI UI)

The CLI uses **Ink**, which renders React components to the terminal.

### Core Principles

1. **Use functional components with Hooks** - No class components
2. **Keep components pure** - No side effects during rendering
3. **Respect one-way data flow** - Props down, events up
4. **Never mutate state directly** - Always use state setters with immutable
   updates
5. **Minimize `useEffect`** - Only for synchronization with external state
6. **Follow Rules of Hooks** - Call hooks unconditionally at top level
7. **Use refs sparingly** - Only for DOM-like interactions, not for reactive
   state
8. **Prefer composition** - Small, reusable components
9. **Optimize for concurrency** - Functional state updates, cleanup functions

### Important: Avoid useEffect When Possible

**Think hard before using `useEffect`:**

- Use it ONLY for synchronization (e.g., with external systems)
- DO NOT call `setState` inside `useEffect` - this degrades performance
- If logic runs in response to user action, put it in an event handler, not
  `useEffect`
- Always include all dependencies in the dependency array
- Always return a cleanup function when subscribing to external resources

### React Compiler Optimization

This project assumes **React Compiler** is enabled:

- DO NOT use `useMemo`, `useCallback`, or `React.memo` manually
- Focus on clear, simple components with direct data flow
- Let the compiler handle optimization

### Network Optimization

- Use parallel data fetching (start multiple requests at once)
- Leverage Suspense for data loading
- Co-locate data fetching with components that need it
- Consider caching and avoiding duplicate requests

### User Experience Patterns

- Show lightweight placeholders (skeleton screens) during loading
- Handle errors gracefully (error boundaries or inline messages)
- Render partial data as it becomes available
- Use Suspense to declare loading states naturally

## Testing Conventions

### Framework: Vitest

All tests use Vitest with these patterns:

**Test Structure:**

- **File location:** Co-located with source files
  - `*.test.ts` for logic tests
  - `*.test.tsx` for React component tests
- **Configuration:** `vitest.config.ts` files define test environments
- **Setup/Teardown:**

  ```typescript
  beforeEach(() => {
    vi.resetAllMocks();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });
  ```

### Mocking with `vi`

**ES Modules:**

```typescript
vi.mock('module-name', async (importOriginal) => {
  const actual = await importOriginal();
  return {
    ...actual,
    specificFunction: vi.fn(),
  };
});
```

**Hoisting:**

```typescript
const myMock = vi.hoisted(() => vi.fn());

vi.mock('some-module', () => ({
  someFunction: myMock,
}));
```

**Mocking Order:** For critical dependencies (e.g., `os`, `fs`) that affect
module-level constants, place `vi.mock` at the **very top** of the test file,
before all other imports.

**Commonly Mocked Modules:**

- Node.js built-ins: `fs`, `fs/promises`, `os`, `path`, `child_process`
- External SDKs: `@google/genai`, `@modelcontextprotocol/sdk`
- Internal packages: Dependencies from other workspace packages

### React Component Testing (Ink)

```typescript
import { render } from 'ink-testing-library';

test('renders correctly', () => {
  const { lastFrame } = render(<MyComponent />);
  expect(lastFrame()).toContain('expected text');
});
```

- Wrap components in necessary `Context.Provider`s
- Mock custom hooks and complex children with `vi.mock()`

### Asynchronous Testing

```typescript
// Promises
await expect(promise).rejects.toThrow('error message');

// Timers
vi.useFakeTimers();
await vi.advanceTimersByTimeAsync(1000);
await vi.runAllTimersAsync();
```

### General Testing Guidance

- Examine existing tests before adding new ones to conform to established
  patterns
- Pay attention to mocks at the top of test files - they reveal critical
  dependencies
- Test the public API of modules, not internal implementation details

## Tool System and MCP Integration

### Built-in Tools

Tools are located in `packages/core/src/tools/`:

**File Operations:**

- `read-file.ts` - Read single file
- `read-many-files.ts` - Read multiple files
- `write-file.ts` - Write/create files
- `edit.ts` - Edit existing files
- `smart-edit.ts` - Intelligent file editing
- `glob.ts` - File globbing/pattern matching
- `ls.ts` - List directory contents

**Search:**

- `grep.ts` - Basic grep functionality
- `ripGrep.ts` - Fast search with ripgrep

**Shell:**

- `shell.ts` - Execute shell commands (with confirmation)

**Web:**

- `web-fetch.ts` - Fetch URLs
- `web-search.ts` - Google web search

**State Management:**

- `memoryTool.ts` - Save/recall across sessions
- `write-todos.ts` - Manage subtasks

**MCP Integration:**

- `mcp-client.ts` - MCP client implementation
- `mcp-client-manager.ts` - Manage multiple MCP clients
- `mcp-tool.ts` - MCP tool wrapper
- `tool-registry.ts` - Central tool registration

### Tool Execution Flow

1. User provides prompt
2. Core sends available tools to Gemini API
3. Gemini requests tool execution with parameters
4. Core validates and (often) requests user confirmation
5. Tool executes and returns result
6. Result sent back to Gemini for final response

### Security Considerations

- **Confirmation required** for file writes, edits, and shell commands
- **Sandboxing** restricts tool operations:
  - macOS: Seatbelt (`sandbox-exec`)
  - Other platforms: Docker or Podman containers
- **Read operations** typically don't require confirmation
- Tools must be available inside the sandbox environment

## Project Structure

```
gemini-cli/
├── .github/              # GitHub workflows, issue templates, actions
├── docs/                 # All project documentation
│   ├── architecture.md
│   ├── get-started/     # Installation, configuration, authentication
│   ├── cli/             # CLI commands, shortcuts, features
│   ├── tools/           # Tool documentation
│   └── extensions/      # Extension development guides
├── packages/
│   ├── cli/             # Frontend CLI package
│   │   ├── src/
│   │   │   ├── commands/     # Slash commands (/help, /chat, etc.)
│   │   │   ├── ui/           # Ink React components
│   │   │   ├── utils/        # CLI utilities
│   │   │   └── gemini.tsx    # Main CLI entry
│   │   └── package.json
│   ├── core/            # Backend logic package
│   │   ├── src/
│   │   │   ├── tools/        # Tool implementations
│   │   │   ├── mcp/          # MCP integration
│   │   │   ├── agents/       # Agent logic
│   │   │   ├── prompts/      # Prompt construction
│   │   │   └── index.ts      # Core exports
│   │   └── package.json
│   ├── a2a-server/      # A2A server (experimental)
│   ├── test-utils/      # Shared testing utilities
│   └── vscode-ide-companion/  # VS Code extension
├── integration-tests/   # End-to-end tests
├── scripts/             # Build, test, and utility scripts
├── schemas/             # JSON schemas (settings, etc.)
├── GEMINI.md           # AI assistant instructions (this project's context)
├── CONTRIBUTING.md     # Contribution guidelines
├── README.md           # Project overview
└── package.json        # Root workspace configuration
```

## Key Files and Locations

### Configuration

- **Settings schema:** `schemas/settings-schema.json`
- **User settings:** `~/.gemini/settings.json`
- **Project context:** `GEMINI.md` in project root or `.gemini/GEMINI.md`
- **Environment:** `.env` files (project root or `~/.env`)

### Entry Points

- **CLI:** `packages/cli/src/gemini.tsx`
- **Non-interactive CLI:** `packages/cli/src/nonInteractiveCli.ts`
- **Core:** `packages/core/src/index.ts`
- **Bundle:** `bundle/gemini.js` (generated)

### Important Utilities

- **Type checks:** `packages/cli/src/utils/checks.ts` (includes
  `checkExhaustive`)
- **Settings:** `packages/cli/src/utils/settingsUtils.ts`
- **Commands:** `packages/cli/src/utils/commands.ts`
- **Git utils:** `packages/cli/src/utils/gitUtils.ts`

## Sandboxing

### macOS Seatbelt

Profiles located in `packages/cli/src/utils/`:

- `sandbox-macos-permissive-open.sb` (default)
- `sandbox-macos-restrictive-closed.sb`

Configure with `SEATBELT_PROFILE` environment variable.

### Container-Based Sandboxing

- Enable: `GEMINI_SANDBOX=true|docker|podman`
- Build: `npm run build:all` (includes sandbox image)
- Custom config: `.gemini/sandbox.Dockerfile` and `.gemini/sandbox.bashrc`

### Proxied Networking

All sandboxing methods support custom proxy servers:

- Set `GEMINI_SANDBOX_PROXY_COMMAND=<command>`
- Proxy must listen on `:::8877`
- See `docs/examples/proxy-script.md` for examples

## Coding Standards

### Import Restrictions

ESLint enforces restrictions on relative imports between packages. Always use
package imports:

```typescript
// Good
import { SomeThing } from '@google/gemini-cli-core';

// Bad (cross-package relative import)
import { SomeThing } from '../../../core/src/something';
```

### Linting and Formatting

- **ESLint:** Enforces code quality and import restrictions
- **Prettier:** Enforces consistent formatting
- **Husky:** Git hooks ensure code quality pre-commit

Configuration files:

- `eslint.config.js`
- `.prettierrc.json`
- `.prettierignore`

### Pre-commit Hook

Optionally set up automatic preflight on commit:

```bash
echo "
if ! npm run preflight; then
  echo 'npm preflight failed. Commit aborted.'
  exit 1
fi
" > .git/hooks/pre-commit && chmod +x .git/hooks/pre-commit
```

## Documentation

### Structure

- **Table of contents:** `docs/sidebar.json`
- **Style guide:** Follow
  [Google Developer Documentation Style Guide](https://developers.google.com/style)
- **Format:** Markdown with consistent formatting via Prettier

### Key Documentation Files

- `docs/architecture.md` - System architecture
- `docs/get-started/` - Getting started guides
- `docs/cli/` - CLI features and commands
- `docs/tools/` - Tool system documentation
- `docs/extensions/` - Extension development
- `CONTRIBUTING.md` - Contribution process
- `GEMINI.md` - AI assistant instructions

### Documentation Best Practices

- Use sentence case for headings
- Write in second person ("you")
- Use present tense
- Include practical examples
- Keep paragraphs short and focused
- Use code blocks with language tags

## Common Pitfalls to Avoid

1. **Using classes instead of plain objects** - Breaks React patterns and adds
   complexity
2. **Using `any` type** - Loses type safety; use `unknown` instead
3. **Mutating state directly** - Always use immutable updates
4. **Overusing `useEffect`** - Think hard before adding effects
5. **Setting state inside `useEffect`** - Performance degradation
6. **Manual memoization** - Let React Compiler handle it
7. **Cross-package relative imports** - Use package imports
8. **Skipping preflight checks** - Always run before committing
9. **Adding unnecessary comments** - Only high-value comments
10. **Testing private implementation details** - Test public APIs

## Quick Reference

### Common Commands

```bash
# Development
npm run preflight           # Full validation suite (always run before PR)
npm start                   # Run CLI from source
npm run debug               # Debug mode
npm run build:all           # Build everything including sandbox

# Testing
npm run test                # Unit tests
npm run test:e2e            # Integration tests
npm run typecheck           # Type checking

# Code Quality
npm run lint                # Lint all code
npm run lint:fix            # Auto-fix + format
npm run format              # Format with Prettier

# Debugging
F5                          # VS Code debugger
DEV=true npm start          # Enable React DevTools
DEBUG=1 gemini              # Debug sandbox
```

### Environment Variables

```bash
# Authentication
GEMINI_API_KEY              # Gemini API key
GOOGLE_API_KEY              # Vertex AI key
GOOGLE_CLOUD_PROJECT        # GCP project
GOOGLE_GENAI_USE_VERTEXAI   # Use Vertex AI

# Sandboxing
GEMINI_SANDBOX              # Enable sandbox (true|docker|podman)
SEATBELT_PROFILE            # macOS Seatbelt profile
GEMINI_SANDBOX_PROXY_COMMAND # Proxy command
BUILD_SANDBOX               # Rebuild custom sandbox

# Development
DEV                         # Enable React DevTools
DEBUG                       # Debug mode
NODE_ENV                    # Environment (development|production)
```

## Additional Resources

- **Architecture:** `docs/architecture.md`
- **Contributing:** `CONTRIBUTING.md`
- **AI Instructions:** `GEMINI.md` (project-specific AI guidance)
- **Tools Documentation:** `docs/tools/index.md`
- **MCP Integration:** `docs/tools/mcp-server.md`
- **Troubleshooting:** `docs/troubleshooting.md`
- **FAQ:** `docs/faq.md`

---

**Remember:** Always run `npm run preflight` before submitting changes. This
ensures code quality, passes all tests, and maintains project standards.
