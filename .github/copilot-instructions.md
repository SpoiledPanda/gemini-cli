# Gemini CLI - AI Coding Agent Instructions

This document provides essential guidance for AI coding agents working on the
Gemini CLI codebase. It covers architecture, workflows, conventions, and
integration points to help you be productive quickly.

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Development Workflow](#development-workflow)
4. [Code Conventions](#code-conventions)
5. [Testing Strategy](#testing-strategy)
6. [Key Integration Points](#key-integration-points)
7. [Common Patterns](#common-patterns)
8. [Quick Reference](#quick-reference)

## Project Overview

**Gemini CLI** is an open-source AI agent that brings Google's Gemini directly
into the terminal. It's a Node.js/TypeScript monorepo that provides a
command-line interface for interacting with Gemini models.

### Key Capabilities

- Interactive terminal UI built with React (Ink framework)
- File system operations and shell command execution
- MCP (Model Context Protocol) server integration
- Web fetching and Google Search grounding
- Conversation checkpointing and token caching
- Sandbox execution environments (macOS Seatbelt, Docker, Podman)

### Repository Structure

```
gemini-cli/
├── packages/
│   ├── cli/              # User-facing CLI (React/Ink UI)
│   ├── core/             # Backend logic and Gemini API client
│   ├── a2a-server/       # Agent-to-Agent server (experimental)
│   ├── test-utils/       # Shared testing utilities
│   └── vscode-ide-companion/  # VS Code extension
├── docs/                 # Comprehensive documentation
├── scripts/              # Build, test, and automation scripts
├── integration-tests/    # End-to-end integration tests
└── schemas/              # JSON schemas for configuration
```

## Architecture

### Core Components

1. **CLI Package (`packages/cli/`)** - Frontend
   - User input processing and command handling
   - React-based terminal UI using Ink
   - History management and display rendering
   - Theme and UI customization
   - CLI configuration settings
   - Main entry point: `packages/cli/src/index.ts`

2. **Core Package (`packages/core/`)** - Backend
   - Gemini API client and communication
   - Prompt construction and management
   - Tool registration and execution logic
   - State management for conversations/sessions
   - Server-side configuration
   - Main exports: `packages/core/src/index.ts`

3. **Tools System (`packages/core/src/tools/`)**
   - Modular tools that extend model capabilities
   - File operations: `read-file.ts`, `write-file.ts`, `edit.ts`,
     `smart-edit.ts`
   - Shell execution: `shell.ts`
   - Search and navigation: `grep.ts`, `ripGrep.ts`, `glob.ts`, `ls.ts`
   - Web access: `web-fetch.ts`, `web-search.ts`
   - MCP integration: `mcp-client.ts`, `mcp-client-manager.ts`, `mcp-tool.ts`
   - Memory management: `memoryTool.ts`

### Interaction Flow

1. User types input → CLI package receives it
2. CLI sends request → Core package
3. Core constructs prompt → Sends to Gemini API
4. Gemini responds (answer or tool request)
5. If tool requested → Core executes (with user approval for modifying
   operations)
6. Tool result → Sent back to Gemini API
7. Final response → Core sends to CLI
8. CLI formats and displays → User sees result

### Key Design Principles

- **Modularity**: Clear separation between frontend (CLI) and backend (Core)
- **Extensibility**: Tool system designed for easy addition of new capabilities
- **User Experience**: Rich, interactive terminal experience with React/Ink
- **Security**: Sandbox execution, trusted folders, user approval for
  modifications
- **Type Safety**: Strong TypeScript usage, avoid `any`, prefer `unknown`

## Development Workflow

### Essential Commands

```bash
# Build entire project
npm run build

# Build all including sandbox
npm run build:all

# Start CLI from source
npm start

# Run full preflight check (REQUIRED before submitting)
npm run preflight

# Individual checks
npm run lint              # ESLint
npm run typecheck         # TypeScript type checking
npm run test              # Unit tests (Vitest)
npm run test:e2e          # Integration tests
npm run format            # Prettier formatting
```

### Before Submitting Changes

**ALWAYS run `npm run preflight`** - This runs:

- Build
- Linting
- Type checking
- All tests
- Formatting checks

This is the comprehensive validation that ensures quality gates are met.

### Git Workflow

- Main branch: `main`
- Create feature branches from `main`
- All PRs require passing CI checks
- Follow [Conventional Commits](https://www.conventionalcommits.org/) format

### Development Setup

**Prerequisites:**

- Node.js `>=20.0.0` (development: `~20.19.0` recommended)
- Git

**Initial Setup:**

```bash
git clone https://github.com/google-gemini/gemini-cli.git
cd gemini-cli
npm install
npm run build
```

**Enable Sandboxing (Recommended):**

- Set `GEMINI_SANDBOX=true` in `~/.env`
- Requires: macOS Seatbelt, Docker, or Podman

## Code Conventions

### TypeScript Style

**Prefer Plain Objects over Classes**

- Use TypeScript interfaces/types instead of JavaScript classes
- Better React integration, serialization, and immutability
- Clearer public API definition via ES module exports
- Example:

  ```typescript
  // Good
  interface User {
    id: string;
    name: string;
  }

  function createUser(name: string): User {
    return { id: crypto.randomUUID(), name };
  }

  // Avoid
  class User {
    constructor(public name: string) {}
  }
  ```

**Type Safety**

- **Avoid `any`** - Use `unknown` instead, then narrow the type
- **Avoid type assertions** (`as Type`) - Only use when absolutely necessary
- **Type narrowing**: Use `typeof`, `instanceof`, or type guards
- Example:
  ```typescript
  function processValue(value: unknown) {
    if (typeof value === 'string') {
      // value is safely a string here
      console.log(value.toUpperCase());
    } else if (typeof value === 'number') {
      // value is safely a number here
      console.log(value * 2);
    }
  }
  ```

**Exhaustive Switch Statements**

- Use `checkExhaustive` helper from `packages/cli/src/utils/checks.ts`
- Ensures all enum/union cases are handled
  ```typescript
  switch (value) {
    case 'option1':
      return handleOption1();
    case 'option2':
      return handleOption2();
    default:
      return checkExhaustive(value);
  }
  ```

**Array Operations**

- Prefer functional array methods: `.map()`, `.filter()`, `.reduce()`,
  `.slice()`
- Promotes immutability and readability
- Avoid mutating methods like `.push()`, `.splice()` on state

**ES Module Encapsulation**

- Use `export` for public API, unexported for private
- No need for `private`/`public` keywords
- Enhances testability and reduces coupling

### React Conventions

**Functional Components with Hooks**

- Use `useState`, `useReducer`, `useEffect`, custom hooks
- NO class components
- Example:

  ```typescript
  function MyComponent({ data }: { data: string }) {
    const [count, setCount] = useState(0);

    return <Text>{data} - {count}</Text>;
  }
  ```

**Pure Components**

- No side effects in render function
- Side effects go in `useEffect` or event handlers
- Avoid `useEffect` when possible - prefer event handlers

**Effect Management**

- Include all dependencies in dependency array
- Return cleanup functions
- Don't call `setState` inside `useEffect` (degrades performance)
- Example:
  ```typescript
  useEffect(() => {
    const subscription = subscribe(source);
    return () => subscription.unsubscribe();
  }, [source]);
  ```

**Memoization**

- Don't manually add `useMemo`, `useCallback`, `React.memo`
- React Compiler handles optimization automatically

### Naming Conventions

- **Files**: Kebab-case for most files (`my-component.tsx`, `utils.ts`)
- **Tests**: Co-located with source (e.g., `tool.ts` → `tool.test.ts`)
- **Components**: PascalCase (`MyComponent.tsx`)
- **Flags**: Use hyphens, not underscores (`--my-flag`, not `--my_flag`)
- **Functions**: camelCase (`createUser`, `handleClick`)
- **Types/Interfaces**: PascalCase (`User`, `ApiResponse`)

### Comments Policy

- Only write high-value comments
- Avoid "talking to users" through comments
- Explain "why", not "what" (code should be self-documenting)

## Testing Strategy

### Framework: Vitest

All tests use Vitest (`describe`, `it`, `expect`, `vi`).

### Test Location

- **Unit tests**: Co-located with source files
  - `tool.ts` → `tool.test.ts`
  - `component.tsx` → `component.test.tsx`
- **Integration tests**: `integration-tests/` directory
- **Configuration**: `vitest.config.ts` files define test environments

### Test Structure

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

describe('MyFunction', () => {
  beforeEach(() => {
    vi.resetAllMocks();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('should handle valid input', () => {
    const result = myFunction('valid');
    expect(result).toBe('expected');
  });

  it('should throw on invalid input', () => {
    expect(() => myFunction('invalid')).toThrow('Error message');
  });
});
```

### Mocking with Vitest

**ES Modules**

```typescript
vi.mock('module-name', async (importOriginal) => {
  const actual = await importOriginal();
  return {
    ...actual,
    myFunction: vi.fn(),
  };
});
```

**Mock Ordering**

- Place `vi.mock` calls at the **top** of test files for critical dependencies
- Use `vi.hoisted()` for functions referenced in `vi.mock` factories

**Commonly Mocked Modules**

- Node.js: `fs`, `fs/promises`, `os`, `path`, `child_process`
- External: `@google/genai`, `@modelcontextprotocol/sdk`
- Internal: Other package dependencies

**React Component Testing (Ink)**

```typescript
import { render } from 'ink-testing-library';

const { lastFrame } = render(<MyComponent />);
expect(lastFrame()).toContain('expected text');
```

**Async Testing**

```typescript
// Promises
await expect(promise).rejects.toThrow('error');
await expect(promise).resolves.toBe(value);

// Timers
vi.useFakeTimers();
await vi.advanceTimersByTimeAsync(1000);
await vi.runAllTimersAsync();
```

### Test Coverage

- Examine existing tests before adding new ones
- Follow established patterns in similar test files
- Test public APIs, not private implementation details

## Key Integration Points

### 1. Tool System

**Adding a New Tool** (`packages/core/src/tools/`)

1. Create tool file: `my-tool.ts`
2. Define tool with schema:
   ```typescript
   export const myTool = defineTool({
     name: 'my_tool',
     description: 'What the tool does',
     parameters: z.object({
       param1: z.string().describe('Parameter description'),
     }),
     execute: async ({ param1 }, context) => {
       // Tool implementation
       return { result: 'success' };
     },
   });
   ```
3. Register in `tool-registry.ts`
4. Add test: `my-tool.test.ts`

**Tool Categories**

- **Modifiable Tools**: Require user approval (edit, write, shell)
- **Read-only Tools**: No approval needed (read-file, ls, grep)
- **MCP Tools**: Dynamically loaded from MCP servers

### 2. MCP (Model Context Protocol) Integration

**MCP Servers** (`packages/core/src/mcp/`)

- Configure in `~/.gemini/settings.json`
- Managed by `mcp-client-manager.ts`
- Individual clients in `mcp-client.ts`
- Wrapped as tools in `mcp-tool.ts`

**Usage Pattern**

```typescript
// User can invoke MCP tools with @ syntax
> @github List my open pull requests
> @slack Send message to #dev channel
```

### 3. CLI Commands

**Custom Commands** (`packages/cli/src/commands/`)

- Extensions system for reusable commands
- Slash commands like `/help`, `/chat`, `/checkpoint`
- Command processors in `packages/cli/src/services/prompt-processors/`

### 4. Configuration System

**Settings Schema** (`schemas/settings.schema.json`)

- User settings in `~/.gemini/settings.json`
- Project-specific: `.gemini/settings.json`
- Generate schema: `npm run schema:settings`
- Generate docs: `npm run docs:settings`

**Environment Variables**

- `.env` files supported
- Project-specific: `.gemini/.env`
- Common vars: `GEMINI_API_KEY`, `GOOGLE_CLOUD_PROJECT`, `GEMINI_SANDBOX`

### 5. Sandbox Integration

**Sandbox Types**

- macOS: Seatbelt (`sandbox-exec`)
- Containers: Docker or Podman
- Profiles: `permissive-open`, `restrictive-closed`, custom

**Custom Sandbox**

- Create `.gemini/sandbox.Dockerfile`
- Create `.gemini/sandbox.bashrc`
- Build with `BUILD_SANDBOX=1 npm run build:all`

### 6. API Integration

**Gemini API** (`packages/core/src/`)

- Client in `packages/core/src/core/`
- Authentication: OAuth, API key, Vertex AI
- Model routing in `packages/core/src/routing/`
- Prompt management in `packages/core/src/prompts/`

### 7. Telemetry

**Telemetry System** (`packages/core/src/telemetry/`)

- OpenTelemetry integration
- Google Cloud Monitoring/Trace exporters
- Clearcut logger for Google Analytics
- Privacy-respecting, opt-in/opt-out

## Common Patterns

### 1. Error Handling

```typescript
import { ToolError } from './tool-error';

function myFunction() {
  if (invalidCondition) {
    throw new ToolError('Descriptive error message', { cause: originalError });
  }
}
```

### 2. File System Operations

```typescript
import { readFile } from 'fs/promises';
import { join } from 'path';

const content = await readFile(join(projectRoot, 'file.txt'), 'utf-8');
```

### 3. Context Pattern

```typescript
interface ToolContext {
  projectRoot: string;
  workingDirectory: string;
  confirmationBus: ConfirmationBus;
  // ... other context
}

async function executeTool(params: Params, context: ToolContext) {
  // Use context for operations
}
```

### 4. State Management (React)

```typescript
// Functional state updates
setCount((prevCount) => prevCount + 1);

// Object/array immutability
setState((prev) => ({ ...prev, newField: value }));
setArray((prev) => [...prev, newItem]);
```

### 5. Import Restrictions

- ESLint enforces import rules between packages
- Core cannot import from CLI
- Use relative imports within packages
- Cross-package imports must be explicit

## Quick Reference

### File Locations

| Purpose          | Path                              |
| ---------------- | --------------------------------- |
| CLI Entry        | `packages/cli/src/index.ts`       |
| Core Entry       | `packages/core/src/index.ts`      |
| Tools            | `packages/core/src/tools/`        |
| React Components | `packages/cli/src/ui/components/` |
| Configuration    | `packages/cli/src/config/`        |
| Prompts          | `packages/core/src/prompts/`      |
| Tests (Unit)     | Co-located with source            |
| Tests (E2E)      | `integration-tests/`              |
| Documentation    | `docs/`                           |
| Build Scripts    | `scripts/`                        |
| Schemas          | `schemas/`                        |

### Important Files

- `GEMINI.md` - Developer instructions (this is the primary reference)
- `CONTRIBUTING.md` - Contribution guidelines
- `README.md` - User-facing documentation
- `docs/architecture.md` - Architecture details
- `package.json` - Scripts and dependencies
- `.nvmrc` - Node version specification
- `tsconfig.json` - TypeScript configuration
- `eslint.config.js` - Linting rules
- `.prettierrc.json` - Code formatting

### Key Commands Summary

```bash
npm run preflight    # Full validation (ALWAYS run before PR)
npm run build        # Build all packages
npm run build:all    # Build + sandbox
npm start            # Run CLI from source
npm run debug        # Debug mode
npm test             # Run all unit tests
npm run test:e2e     # Run integration tests
npm run lint         # Lint code
npm run lint:fix     # Auto-fix linting issues
npm run format       # Format code with Prettier
npm run typecheck    # Type check without emitting
```

### Documentation Resources

- **Getting Started**: `docs/get-started/`
- **CLI Features**: `docs/cli/`
- **Tools**: `docs/tools/`
- **Architecture**: `docs/architecture.md`
- **Integration Tests**: `docs/integration-tests.md`
- **FAQ**: `docs/faq.md`
- **Troubleshooting**: `docs/troubleshooting.md`

### CI/CD

- **Main CI**: `.github/workflows/ci.yml`
- **E2E Tests**: `.github/workflows/e2e.yml`
- **Release**: `.github/workflows/release-*.yml`
- **Automation**: Various Gemini-powered automation workflows

### Getting Help

1. Check existing tests for examples
2. Review similar code in the codebase
3. Consult `GEMINI.md` for specific guidance
4. Read relevant documentation in `docs/`
5. Run `/bug` command in CLI to report issues

## Important Reminders

1. **Always run `npm run preflight`** before submitting changes
2. **Follow existing patterns** - examine similar code first
3. **Write tests** - co-located with source files
4. **Type safety** - avoid `any`, use `unknown` + type narrowing
5. **Plain objects** - prefer over classes for data structures
6. **React patterns** - functional components, hooks, immutability
7. **Tool system** - use for extending Gemini capabilities
8. **Documentation** - update when changing user-facing features
9. **Git commits** - follow Conventional Commits format
10. **Security** - be mindful of sandboxing and user approvals

## Additional Context

- This is a monorepo using npm workspaces
- React UI is terminal-based using Ink (not web/DOM)
- Strong emphasis on developer experience and productivity
- Open source under Apache 2.0 license
- Active development with weekly releases (nightly, preview, stable)
- Community-driven with contributions welcome

---

For more detailed information, always refer to:

- `GEMINI.md` - Primary developer guide
- `CONTRIBUTING.md` - Contribution process
- `docs/` - Comprehensive documentation
- Existing code - Best source of truth for patterns
