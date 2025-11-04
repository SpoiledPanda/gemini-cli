## Quick onboarding for automated coding agents (Gemini CLI)

This file gives focused, actionable knowledge for an AI coding agent to be
productive in this repository. It is intentionally concise — prefer concrete
file references and commands over generic advice.

### Big-picture architecture

- Monorepo using NPM workspaces: root `package.json` -> `packages/*`.
- Five main packages:
  - `packages/cli` — the user-facing terminal UI (Ink/React). The CLI is bundled
    into a single executable (`bundle/` / `dist`) and provides input parsing,
    rendering, and UX.
  - `packages/core` — backend orchestration: prompt construction, Gemini API
    calls, tool registration, and session/state management.
  - `packages/a2a-server` — agent-to-agent server for programmatic interactions.
  - `packages/vscode-ide-companion` — VS Code extension for IDE integration.
  - `packages/test-utils` — shared testing utilities across packages.
- Tools live under `packages/core/src/tools/` — these are modular capabilities
  (file system, shell, web fetch, MCP integration) invoked by the model.
  Available tools include: edit, glob, grep, ls, read-file, read-many-files,
  ripGrep, shell, smart-edit, web-fetch, web-search, write-file, write-todos,
  and MCP client tools.

See `docs/architecture.md` for the flow: CLI → Core → Gemini API → optional tool
execution → CLI display.

### Critical developer workflows (commands & where to run them)

- Install and workspace basics (root):
  - `npm install` (root) — installs all workspaces.
  - Node engine: Node >= 20 (see root `package.json`).
- Build & run:
  - `npm run build` (root) — compile all packages (invokes `scripts/build.js`).
  - `npm run build:all` — build packages + sandbox image.
  - `npm start` — run the local CLI (after build).
  - `npm run build --workspace @google/gemini-cli` or
    `npm run build --workspace @google/gemini-cli-core` to build single
    packages.
- Tests:
  - `npm run test` — unit tests across workspaces.
  - `npm run test:integration:sandbox:none` — integration tests without sandbox.
  - `npm run test:integration:sandbox:docker` — integration tests with Docker
    sandbox.
  - `npm run test:integration:sandbox:podman` — integration tests with Podman
    sandbox.
  - `npm run test:integration:all` — runs all integration test variants.
  - `npm run test:e2e` — E2E (sets VERBOSE/KEEP_OUTPUT as needed).
  - `npm run test:ci` — CI test suite (includes all workspace tests + script
    tests).
  - `npm run test:scripts` — tests for scripts in the scripts/ directory.
  - `npm run preflight` — full local quality gate (clean, install, format, lint,
    build, typecheck, tests).
  - Test framework: Vitest with test files co-located (_.test.ts, _.test.tsx).
- Debugging:
  - `npm run debug` — starts node with `--inspect-brk` (root script configured
    to debug the bundled CLI).
  - Use React DevTools to inspect Ink-based UI (see
    `docs/local-development.md`).
  - `DEBUG=1` environment variable enables additional debug output.
  - `VERBOSE=true` for verbose test output.
  - `KEEP_OUTPUT=true` to preserve test artifacts.
- Telemetry / dev traces:
  - `npm run telemetry -- --target=genkit` and run
    `GEMINI_DEV_TRACING=true gemini` to collect traces.
  - `npm run telemetry -- --target=gcp` for GCP telemetry collection.

### Project-specific patterns & conventions

- Bundling: CLI is bundled into `bundle/` / `dist` and the root `bin` points to
  that file. See `package.json` and `packages/cli/package.json`. Use
  `npm run bundle` to create the bundle.
- Workspaces: use `--workspace` flags from root instead of CDing into packages
  when possible.
- Sandboxing: the repo supports macOS Seatbelt and container sandboxes
  (Docker/Podman). Use `GEMINI_SANDBOX` environment variable and `.gemini/`
  overrides when simulating execution. See `docs/cli/sandbox.md` and
  `scripts/build_sandbox.js`.
- Tool approval flow: tools that modify state typically require explicit
  confirmation before executing. Concrete examples:
  - `packages/core/src/tools/write-file.ts` (shows acceptance flow & status
    handling).
  - IDE diff acceptance: `packages/vscode-ide-companion/src/diff-manager.ts` and
    `packages/core/src/ide/ide-client.ts` (look for `ide/diffAccepted`).
- Custom agent/context files: the repo looks for project-specific context
  (`GEMINI.md`) and supports custom agent discovery (tests reference
  `CUSTOM_AGENTS.md`). See `GEMINI.md` and memory discovery tests under
  `packages/core/src/utils/memoryDiscovery.test.ts`.
- Code style: prefer plain objects + TypeScript interfaces over classes. Avoid
  `any` types; prefer `unknown` with type narrowing. Use ES module import/export
  for encapsulation. See `GEMINI.md` for detailed JavaScript/TypeScript
  conventions.
- Testing: use Vitest framework. Mock with `vi.mock()`, place mocks at top of
  file for module-level constants. Use `vi.hoisted()` for mocks needed in
  factory functions. See `GEMINI.md` for full testing guidelines.
- React: functional components with hooks only (no classes). Keep components
  pure, avoid mutations, follow Rules of Hooks. Rely on React Compiler for
  optimization (no manual useMemo/useCallback). See `GEMINI.md` for
  comprehensive React guidelines.

### Integration points & external dependencies

- Gemini API / auth:
  - Multiple auth modes: Google OAuth, Gemini API key, Vertex AI. See
    `README.md` and `docs/get-started/authentication.md`.
  - Environment variables: `GEMINI_API_KEY`, `GOOGLE_API_KEY`,
    `GOOGLE_GENAI_USE_VERTEXAI`, `GOOGLE_CLOUD_PROJECT`.
- MCP (Model Context Protocol): The CLI can call external MCP servers;
  configuration lives in `~/.gemini/settings.json` and docs under
  `docs/tools/mcp-server.md`. Implementation:
  `packages/core/src/tools/mcp-client.ts`, `mcp-client-manager.ts`,
  `mcp-tool.ts`.
- IDE Integration: VS Code companion extension (`packages/vscode-ide-companion`)
  provides diff viewing and acceptance flows.
- Telemetry / Genkit / Jaeger: `npm run telemetry` scripts. Dev tracing is
  opt-in via `GEMINI_DEV_TRACING`.
- Dependencies: Uses `@google/genai` SDK, `@modelcontextprotocol/sdk`, Ink for
  terminal UI, Vitest for testing.

### Where to look for common editing tasks

- To modify CLI behavior/UI: `packages/cli/src/` (Ink components and command
  parsing).
- To add or change tools (file/shell/web): `packages/core/src/tools/`. Register
  new tools in `tool-registry.ts`.
- To inspect how prompts are constructed and model calls are made:
  `packages/core/src` (search for `prompt` and model/GenAI client usage).
- To modify authentication flows: `packages/core/src/auth/` and related config
  handling.
- To change build/bundle process: `esbuild.config.js`, `scripts/build.js`,
  `scripts/copy_bundle_assets.js`.
- To update settings schema: `scripts/generate-settings-schema.ts` and
  `scripts/generate-settings-doc.ts`.
- For CI/CD changes: `.github/workflows/` (ci.yml, e2e.yml, release-\*.yml,
  etc.).
- For custom commands: `.gemini/commands/` and see
  `docs/cli/custom-commands.md`.

### Quick examples an agent can use

- Add a small tool: create `packages/core/src/tools/my-tool.ts`, register it in
  `packages/core/src/tools/tool-registry.ts`, and add tests as
  `packages/core/src/tools/my-tool.test.ts`.
- Run unit tests for only `core` while developing:
  - `npm run test --workspace @google/gemini-cli-core`
- Run unit tests for only `cli` while developing:
  - `npm run test --workspace @google/gemini-cli`
- Run integration tests locally (no sandbox):
  - `npm run test:integration:sandbox:none`
- Build only the core package:
  - `npm run build --workspace @google/gemini-cli-core`
- Debug a specific test file:
  - `cd packages/core && npx vitest src/tools/my-tool.test.ts`
- Generate settings documentation:
  - `npm run docs:settings` (after building core)

### Contract for changes made by an AI agent

- Inputs: target files under `packages/*`, a short PR description, and tests to
  validate.
- Outputs: small, focused PR that updates code + tests + docs (if user-visible).
- Error modes: failed build, lint, or typecheck — run `npm run preflight` and
  fix the first failing error.

### Edge cases and gotchas

- Node version: many dev tools depend on Node >= 20.0.0 (preferably ~20.19.0);
  mismatch can cause build/test failures.
- Sandbox tests may require Docker/Podman and longer runtime; run
  `test:integration:sandbox:none` for fast feedback.
- Large changes across packages require workspace-aware builds (`npm run build`
  from root).
- Bundle creation: `npm run bundle` is called during `prepare` hook and creates
  the bundled CLI. The root `bin` field points to `bundle/gemini.js`.
- Ink version override: the project uses a forked version `@jrichman/ink@6.4.0`
  (see overrides in package.json).
- Windows path handling: see `scripts/test-windows-paths.js` for path
  normalization tests.
- Deflaking tests: use `npm run deflake` scripts to retry flaky tests (see
  `scripts/deflake.js`).
- Lock file validation: `npm run check:lockfile` ensures package-lock.json
  integrity.
- Pre-commit hooks: Husky runs `scripts/pre-commit.js` which includes
  lint-staged for formatting and linting.

---

If any of these areas are unclear or you'd like more examples (e.g., prompt
construction files, a sample tool implementation, or the register/registry code
path), tell me which area to expand and I'll iterate.
