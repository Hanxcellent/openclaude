# Architecture Analysis

## Architectural Style

OpenClaude is a modular TypeScript application centered on one long-running Node.js CLI process. It combines:

- a React/Ink terminal UI,
- a query/agent orchestration engine,
- metadata-driven provider routing,
- a tool execution layer,
- plugin, skill and MCP extension systems,
- SDK, MCP and headless execution entrypoints.

The design is a layered modular monolith with event-driven and registry-based extension points.

## Runtime Flow

```text
Terminal input
  → CLI parsing and startup initialization
  → AppStateProvider + REPL render
  → PromptInput / commands / dialogs
  → QueryEngine turn lifecycle
  → provider routing and API transport
  → assistant stream events
  → tool dispatch and task orchestration
  → state updates
  → custom Ink renderer writes terminal frames
```

## Major Subsystems

| Subsystem | Primary Locations | Responsibility |
|-----------|-------------------|----------------|
| Bootstrap and CLI | `src/entrypoints/`, `src/main.tsx`, `src/bootstrap/` | Environment setup, arguments, startup prefetch, mode selection. |
| TUI | `src/screens/`, `src/components/`, `src/hooks/`, `src/ink/` | Interactive terminal rendering and input handling. |
| State | `src/state/AppState.tsx`, contexts | Global mutable session state exposed through selectors. |
| Query orchestration | `src/QueryEngine.ts`, `src/query/` | Turn lifecycle, cancellation, compaction, limits and tool loops. |
| Provider layer | `src/integrations/`, `src/services/api/` | Provider metadata, route resolution, authentication and transport adaptation. |
| Tools and tasks | `src/tools/`, `src/tasks/` | Model-invoked capabilities and long-running work. |
| Extensibility | `src/services/mcp/`, `src/plugins/`, `src/skills/` | External servers, plugins, commands, agents and reusable instructions. |
| Persistence | `src/utils/sessionStorage*`, settings/config utilities | Sessions, transcripts, project/global configuration and caches. |

## Patterns

| Pattern | Usage |
|---------|-------|
| Registry/descriptor | Providers, gateways, models, tools and commands are resolved from metadata or registries. |
| Adapter | OpenAI-compatible, Anthropic-compatible, Gemini/Vertex and local transports normalize into shared query behavior. |
| Observer/signal | Settings changes, index completion, notifications and application state updates. |
| React context/store | App state and UI-specific contexts feed the TUI. |
| Progressive/lazy initialization | Expensive integrations, file indexes, keychain and managed settings are prefetched or dynamically imported. |
| Feature gates | `bun:bundle` compile-time features remove unavailable product surfaces. |
| Task state machine | Background agents, shells and workflows have explicit task types and lifecycle states. |

## State Management

`AppStateProvider` owns the primary global store. Components subscribe through selector hooks; services may read the stable store imperatively. Additional focused contexts manage overlays, notifications, permissions, selection and terminal behavior. Module-level caches are also used for performance-sensitive discovery and configuration paths.

## Architectural Constraints

1. The built CLI must run on Node.js 22+, while Bun is the development/build/test runtime.
2. React rendering must not perform blocking filesystem or process work; expensive work is expected to use effects, caches, prefetching or event-loop yielding.
3. Provider behavior must preserve compatibility across first-party and third-party transports.
4. Optional modules and feature-gated source must remain absent from bundles when disabled.
5. The SDK bundle must not leak React/Ink implementation dependencies.

## Observations

- `src/main.tsx`, `src/screens/REPL.tsx`, provider configuration and API transport modules are high-centrality files.
- Performance depends heavily on lifecycle correctness because components remain mounted for long sessions and many services use timers, subprocesses and subscriptions.
- Extension systems are powerful but create multiple sources of commands, agents, tools, prompts and configuration that must be merged deterministically.
