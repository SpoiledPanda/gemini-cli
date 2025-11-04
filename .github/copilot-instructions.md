## Quick onboarding for automated coding agents (Gemini CLI)

This file gives focused, actionable knowledge for an AI coding agent to be
productive in this repository. It is intentionally concise — prefer concrete
file references and commands over generic advice.

### Big-picture architecture
- Monorepo using NPM workspaces: root `package.json` -> `packages/*`.
- Two primary runtime packages:
  - `packages/cli` — the user-facing terminal UI (Ink/React). The CLI is
    bundled into a single executable (`bundle/` / `dist`) and provides input
    parsing, rendering, and UX.
  - `packages/core` — backend orchestration: prompt construction, Gemini API
    calls, tool registration, and session/state management.
- Tools live under `packages/core/src/tools/` — these are modular capabilities
  (file system, shell, web fetch, MCP integration) invoked by the model.

See `docs/architecture.md` for the flow: CLI → Core → Gemini API → optional
tool execution → CLI display.

### Critical developer workflows (commands & where to run them)
- Install and workspace basics (root):
  - `npm install` (root) — installs all workspaces.
  - Node engine: Node >= 20 (see root `package.json`).
- Build & run:
  - `npm run build` (root) — compile all packages (invokes `scripts/build.js`).
  - `npm run build:all` — build packages + sandbox image.
  - `npm start` — run the local CLI (after build).
  - `npm run build --workspace @google/gemini-cli` or `npm run build --workspace @google/gemini-cli-core` to build single packages.
- Tests:
  - `npm run test` — unit tests across workspaces.
  - `npm run test:integration:sandbox:none` — integration tests without sandbox.
  - `npm run test:e2e` — E2E (sets VERBOSE/KEEP_OUTPUT as needed).
  - `npm run preflight` — full local quality gate (clean, install, format, lint, build, typecheck, tests).
- Debugging:
  - `npm run debug` — starts node with `--inspect-brk` (root script configured to debug the bundled CLI).
  - Use React DevTools to inspect Ink-based UI (see `docs/local-development.md`).
- Telemetry / dev traces:
  - `npm run telemetry -- --target=genkit` and run `GEMINI_DEV_TRACING=true gemini` to collect traces.

### Project-specific patterns & conventions
- Bundling: CLI is bundled into `bundle/` / `dist` and the root `bin` points to that file. See `package.json` and `packages/cli/package.json`.
- Workspaces: use `--workspace` flags from root instead of CDing into packages when possible.
- Sandboxing: the repo supports macOS Seatbelt and container sandboxes. Use `GEMINI_SANDBOX` and `.gemini/` overrides when simulating execution.
- Tool approval flow: tools that modify state typically require explicit confirmation before executing. Concrete examples:
  - `packages/core/src/tools/write-file.ts` (shows acceptance flow & status handling).
  - IDE diff acceptance: `packages/vscode-ide-companion/src/diff-manager.ts` and `packages/core/src/ide/ide-client.ts` (look for `ide/diffAccepted`).
- Custom agent/context files: the repo looks for project-specific context (`GEMINI.md`) and supports custom agent discovery (tests reference `CUSTOM_AGENTS.md`). See `GEMINI.md` and memory discovery tests under `packages/core/src/utils/memoryDiscovery.test.ts`.

### Integration points & external dependencies
- Gemini API / auth:
  - Multiple auth modes: Google OAuth, Gemini API key, Vertex AI. See `README.md` and `docs/get-started/authentication.md`.
- MCP (Model Context Protocol): The CLI can call external MCP servers; configuration lives in `~/.gemini/settings.json` and docs under `docs/tools/mcp-server.md`.
- Telemetry / Genkit / Jaeger: `npm run telemetry` scripts. Dev tracing is opt-in via `GEMINI_DEV_TRACING`.

### Where to look for common editing tasks
- To modify CLI behavior/UI: `packages/cli/src/` (Ink components and command parsing in `packages/cli/src`).
- To add or change tools (file/shell/web): `packages/core/src/tools/`.
- To inspect how prompts are constructed and model calls are made: `packages/core/src` (search for `prompt` and model/GenAI client usage).

### Quick examples an agent can use
- Add a small tool: create `packages/core/src/tools/my-tool.ts`, register it in tool registry, and add tests under `packages/core/test`.
- Run unit tests for only `core` while developing:
  - `npm run test --workspace @google/gemini-cli-core`
- Run integration tests locally (no sandbox):
  - `npm run test:integration:sandbox:none`

### Contract for changes made by an AI agent
- Inputs: target files under `packages/*`, a short PR description, and tests to validate.
- Outputs: small, focused PR that updates code + tests + docs (if user-visible).
- Error modes: failed build, lint, or typecheck — run `npm run preflight` and fix the first failing error.

### Edge cases and gotchas
- Node version: many dev tools depend on Node ~20.19.0; mismatch can cause build/test failures.
- Sandbox tests may require Docker/Podman and longer runtime; run `test:integration:sandbox:none` for fast feedback.
- Large changes across packages require workspace-aware builds (`npm run build` from root).

---
If any of these areas are unclear or you'd like more examples (e.g., prompt construction files, a sample tool implementation, or the register/registry code path), tell me which area to expand and I'll iterate.
