# Testing Approach

## Test Organization

- Unit and component tests are colocated under `src/**.test.ts(x)`.
- SDK lifecycle and public API tests live under `tests/sdk/`.
- Build and bundle guards live under `tests/build/`, `scripts/*.test.ts` and entrypoint tests.
- The repository currently contains roughly 573 test files.

## Main Commands

| Command | Purpose |
|---------|---------|
| `bun test <file>` | Focused test execution. |
| `bun run typecheck` | TypeScript application typecheck. |
| `bun run typecheck:type-tests` | Public type contract checks. |
| `bun run build` | Build CLI/SDK and validate bundles. |
| `bun run smoke` | Build and run the produced CLI version command. |
| `bun run deadcode` | Knip dead-code/dependency analysis. |
| `bun run check` | Smoke + deadcode + full serial test suite. |
| `bun run security:pr-scan` | PR security diagnostics. |

## Testing Patterns

- Pure functions are extracted for deterministic state-machine and routing tests.
- React/Ink tests use the repository's custom root and stream-backed stdout.
- Process behavior is tested with controlled fake child processes.
- Dynamic module imports with nonces isolate module-level cache state.
- Tests explicitly account for persistent Bun module mocks.
- Provider tests assert both route choice and removal of stale incompatible configuration.

## Current Baseline

At the time of this map, the full `bun run check` completes smoke and dead-code checks but the full suite has an existing baseline of 92 failures in the local environment. The same failure-name set occurs on `main` and the active fix branch, indicating those failures are not caused by the recent input-stall work. This baseline should be documented or resolved before treating a non-zero full-suite result as a new regression signal.

## Coverage Strengths

- Provider routing and compatibility behavior.
- SDK public lifecycle and generated types.
- Tool schemas, permissions and security guards.
- CLI parsing, session persistence and configuration migration.
- Bundle contents and optional-runtime behavior.

## Coverage Risks

- Long-lived TUI effect interactions can require explicit lifecycle/state-machine tests.
- Full-suite global environment and module mocks can cause unrelated cascading failures.
- Performance regressions require dynamic timing/log evidence in addition to unit tests.
- Some tests depend on local tools, credentials, platform behavior or network isolation.
