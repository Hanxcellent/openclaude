# Technology Stack

## Runtime and Languages

| Technology | Version/Constraint | Use |
|------------|--------------------|-----|
| TypeScript | 5.9.3 | Application, scripts, tests and SDK types. |
| Node.js | `>=22.0.0` | Runtime for the published CLI. |
| Bun | Repository-managed current version | Dependency management, builds, scripts and tests. |
| ESM | Project-wide | Runtime module format and bundled output. |

## UI and Rendering

| Technology | Version | Use |
|------------|---------|-----|
| React | 19.2.4 | Component and hook model for the terminal UI. |
| React Reconciler | 0.33.0 | Custom Ink-style terminal reconciler. |
| Ink-derived internal renderer | Repository code | Layout, input, terminal patching and rendering. |
| Chalk | 5.6.2 | Terminal color formatting. |

## CLI and Process Integration

- `@commander-js/extra-typings` for CLI arguments.
- `cross-spawn`, `execa`, `tree-kill` and internal process helpers for subprocesses.
- `proper-lockfile` and filesystem abstractions for persistence and coordination.
- `ws`, gRPC/protobuf modules and SSH utilities for remote/headless modes.

## Model and Extension Protocols

- Anthropic SDK and OpenAI-compatible transports.
- Google authentication and Gemini/Vertex integrations.
- MCP SDK for tool/resource/server interoperability.
- Zod schemas for runtime validation and generated public types.
- Tree-sitter/WebAssembly modules for language-aware behavior.

## Data and Content Utilities

- `yaml`, `jsonc-parser`, `marked`, `turndown` for configuration and text formats.
- `js-tiktoken` for token estimation.
- `sharp`, PDF and document utilities for artifact processing.
- No central relational database; persistence is primarily files, JSON/JSONL, settings and provider services.

## Build and Quality Tooling

| Tool | Purpose |
|------|---------|
| `scripts/build.ts` | Bundle CLI and SDK; validate externals and exports. |
| TypeScript strict mode | Static type validation. |
| Bun test | Unit, integration and SDK tests. |
| Knip | Dead-file and dependency analysis. |
| Biome/custom rules | Formatting/lint conventions and source guards. |
| GitHub Actions | PR checks and release automation. |

## Secondary Surfaces

- Documentation website under `web/`.
- VS Code extension under `vscode-extension/openclaude-vscode/`.
- Generated protobuf, integration and SDK type artifacts are checked for synchronization during builds.
