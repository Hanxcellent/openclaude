# Codebase Concerns

## High-Centrality Modules

- `src/main.tsx` and `src/screens/REPL.tsx` coordinate many responsibilities and are expensive to reason about or modify safely.
- Provider configuration and API transport modules carry broad cross-provider compatibility risk.
- Global state and module-level caches create implicit coupling between UI lifecycles and services.

## Performance and Responsiveness

The interactive CLI is sensitive to synchronous work on the JavaScript thread. Known risk categories include:

- subprocess spawning from input-driven paths,
- filesystem scans and Git discovery,
- component remounts that restart effects,
- terminal layout/render work on large trees,
- timers and subscriptions that survive hidden UI states,
- startup imports and secure-storage/config reads.

The codebase already uses prefetching, lazy imports, progressive indexes, abort controllers and frame diagnostics, showing that responsiveness is an explicit architectural concern.

## Lifecycle Complexity

Long-running components and services share refs, timers, signals and caches. Correctness requires clear ownership:

- cache resets must not remove component lifecycle subscriptions,
- hidden components must suspend expensive side effects without losing state,
- completion signals must publish committed state before notifying listeners,
- cleanup must cancel pending and in-flight work.

## Extension and Trust Boundaries

Plugins, skills, hooks, MCP servers, repository files and managed settings can influence runtime behavior. Risks include command execution, path traversal, credential exposure, prompt injection and configuration precedence errors. Existing approval, validation and security scanning patterns should remain mandatory.

## Provider Compatibility

The breadth of supported providers creates a large regression matrix. Compatibility shims can silently inherit stale environment variables, headers, effort settings or model aliases. Descriptor-driven metadata reduces duplication but does not eliminate transport-specific exceptions.

## Test Baseline

The local full-suite baseline currently contains 92 failures shared with `main`. Until resolved, change validation must compare failure sets, not only exit codes or failure counts.

## Repository Scale

Nearly 3,000 TypeScript/TSX files and multiple shipped surfaces increase navigation and build complexity. Generated files, feature gates and optional modules make broad refactors especially risky. Changes should remain surgical and accompanied by focused tests.

## Planning Implications

Future roadmap phases should prioritize:

1. stable TUI responsiveness and effect lifecycle ownership,
2. provider routing correctness and conformance tests,
3. extension trust/security boundaries,
4. reduction of high-centrality module complexity,
5. reliable full-suite baseline and deterministic test isolation,
6. clear public SDK/CLI compatibility policy.
