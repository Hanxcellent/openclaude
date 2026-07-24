# Codebase Conventions

## TypeScript

- Strict TypeScript and ESM imports are standard.
- Runtime imports generally use explicit `.js` suffixes even from TypeScript sources.
- Types are imported with `import type` where practical.
- Zod schemas are used where values cross configuration, API or SDK boundaries.
- Existing local patterns take precedence over introducing new abstractions.

## React and Ink

- Hooks and focused components are preferred over class components.
- Expensive work must not occur synchronously during render.
- Components frequently use refs to keep async callbacks stable while reading current state.
- Long-lived effects must clean up timers, subscriptions, abort controllers and subprocess work.
- React Compiler output/style appears in some source files; avoid unrelated reformatting.

## Services and State

- App-wide state is selected through `useAppState`; imperative reads use a stable store reference.
- Module-level caches are common for startup and query performance. Cache state and subscriber lifecycle must be separated.
- Cancellation is represented with `AbortController`/`AbortSignal` and explicit cleanup.
- Debug logging uses repository utilities rather than direct console output in runtime paths.

## Providers and Integrations

- Provider metadata belongs in integration descriptors; transport exceptions belong in transport/service layers.
- Route resolution must avoid leaking ambient credentials or model settings between providers.
- Provider changes require exact-path tests because many providers share compatibility shims.
- Generated integration lists and public types must remain synchronized.

## Tests

- Tests are colocated as `*.test.ts` or `*.test.tsx`; broader SDK/build tests live under `tests/`.
- Bun's test runner and `mock.module` are used extensively.
- Module mocks can persist process-wide; tests explicitly restore or gate them to avoid cross-file contamination.
- For async behavior, tests prefer controlled promises, fake subprocesses and observable lifecycle assertions.

## Change Discipline

- Keep patches focused and avoid unrelated renames or formatting.
- Add tests for behavior changes.
- Do not add dependencies without strong justification.
- Do not change the Node/Bun runtime contract without maintainer agreement.
- Run the narrowest useful validation first, then build/typecheck/full checks as appropriate.
