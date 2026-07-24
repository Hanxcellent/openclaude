# Codebase Structure

## Directory Layout

```text
openclaude/
├── bin/                    # Published CLI launchers
├── src/
│   ├── entrypoints/        # CLI, MCP and public SDK entrypoints
│   ├── screens/            # Top-level Ink screens, primarily the REPL
│   ├── components/         # React/Ink TUI components and dialogs
│   ├── ink/                # Custom terminal renderer and reconciler
│   ├── state/              # Global application state and selectors
│   ├── query/              # Query lifecycle, limits and transitions
│   ├── services/api/       # Provider transports and request lifecycle
│   ├── integrations/       # Provider/gateway/model metadata and routing
│   ├── tools/              # Agent-callable tools
│   ├── commands/           # Slash commands and CLI command handlers
│   ├── tasks/              # Local, remote, shell, workflow and agent tasks
│   ├── services/mcp/       # MCP discovery, connection and approval
│   ├── plugins/            # Plugin discovery and bundled plugins
│   ├── skills/             # Bundled skill definitions and loaders
│   ├── utils/              # Shared runtime/config/session utilities
│   └── types/              # Shared protocol and application types
├── tests/                  # Build and SDK integration tests
├── scripts/                # Build, generation, validation and release scripts
├── docs/                   # User and integration documentation
├── web/                    # Documentation website
├── vscode-extension/       # VS Code integration
├── vendor/                 # Vendored compatibility code
└── package.json            # Bun workflows; Node runtime package metadata
```

## Key Entry Points

| Path | Purpose |
|------|---------|
| `bin/openclaude` | Installed executable shim. |
| `src/entrypoints/cli.tsx` | Early CLI argument handling and runtime compatibility setup. |
| `src/main.tsx` | Commander configuration, startup prefetching, mode selection and render orchestration. |
| `src/screens/REPL.tsx` | Main interactive coding-agent screen. |
| `src/entrypoints/mcp.ts` | MCP server entrypoint. |
| `src/entrypoints/sdk/index.ts` | Public programmatic SDK surface. |
| `scripts/build.ts` | CLI/SDK bundling and bundle validation. |

## Module Organization

- UI code is grouped by surface under `src/components/`; top-level interactive flows live under `src/screens/`.
- Provider support is metadata-driven under `src/integrations/`, while transport execution lives under `src/services/api/`.
- Agent capabilities are concrete modules under `src/tools/`; user commands are under `src/commands/`.
- Cross-session state, configuration and filesystem behavior are concentrated in `src/utils/` and `src/services/`.
- The repository also ships SDK, MCP, web documentation and VS Code surfaces from the same source tree; it is a large multi-surface package rather than a workspace monorepo.

## Key Dependency Direction

```text
CLI entry → main/bootstrap → REPL/screens → components/hooks → state/services
QueryEngine → provider resolution → API transport → external model provider
QueryEngine → tool registry → tools → filesystem/process/MCP/web services
Plugins/skills/settings → commands/tools/prompts → REPL and QueryEngine
SDK/MCP entrypoints → shared query, tool and session modules
```

## Scale Snapshot

- Approximately 2,988 TypeScript/TSX files under `src/` and `tests/`.
- Approximately 573 test files.
- About 112 command directories and 56 tool directories.
- Generated and built output is kept in-tree under `dist/` and generated integration/type directories.
