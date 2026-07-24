# Requirements

## Independent Requirement Packages

| Package | Detailed Specification | Review Branch | Base | Dependencies |
|---------|------------------------|---------------|------|--------------|
| IR-001 `/` suggestion disappearance | `.planning/requirements/01-slash-suggestion-freeze.md` | `fix/input-freeze-slash` | `main` | None |
| IR-002 Ctrl+C delayed feedback | `.planning/requirements/02-ctrl-c-feedback-freeze.md` | `fix/input-freeze-ctrl-c` | `main` | None |
| IR-003 `@` file suggestions | `.planning/requirements/03-at-suggestion-freeze.md` | `fix/input-freeze-at` | `main` | None |

The packages are independent delivery units, not sequential phases. Shared quality requirements REQ-020 through REQ-023 apply separately to each branch.

## v1 Functional Requirements

### IR-001: Slash Suggestion Lifecycle

- **REQ-001**: Showing or hiding inline `/` suggestions shall not unmount the regular footer subtree. Traces: UC-001.
- **REQ-002**: Slash-suppressed footer content shall collapse without adding blank rows or clipping visible wrapped content. Traces: UC-001.

### IR-002: Ctrl+C Feedback Lifecycle

- **REQ-003**: Ctrl+C exit feedback and paste feedback shall hide, not unmount, long-lived left-footer and configured statusline state. Traces: UC-003, UC-004.

### IR-002: Custom Statusline Activity

- **REQ-004**: A custom statusline shall execute once on initial active mount. Traces: UC-009.
- **REQ-005**: Relevant visible state changes shall schedule at most one 300ms debounced update. Traces: UC-008.
- **REQ-006**: Transitioning inactive shall cancel any pending debounce timer and abort any in-flight command without consuming a pending first-run or hot-reload result log. Traces: UC-008.
- **REQ-007**: Changes observed while inactive shall mark the statusline dirty without executing a command. Traces: UC-008.
- **REQ-008**: Reactivating a dirty statusline shall execute exactly one immediate update. Traces: UC-009.
- **REQ-009**: Statusline scheduling correctness shall not depend on React effect declaration order. Traces: UC-008, UC-009.

### IR-003: File Suggestions

- **REQ-010**: Built-in file suggestions shall start an index prewarm when typeahead mounts outside tests. Traces: UC-005.
- **REQ-011**: Cold and refreshing file queries shall return from the progressive in-memory index without awaiting full discovery. Traces: UC-002, UC-005.
- **REQ-012**: Index refresh shall remain single-flight, including when completion subscribers synchronously request another refresh. Traces: UC-002.
- **REQ-013**: Refresh completion shall re-run only the latest active, unsuppressed `@` query and the current Quick Open query. Accepted, dismissed, or history-suppressed queries shall not replay; the current selection shall remain stable during partial-to-full upgrades; Quick Open replay shall not start another background refresh. Traces: UC-002, UC-005.
- **REQ-014**: `/clear` shall clear cache data without removing lifecycle subscriptions. Traces: UC-007.
- **REQ-015**: The existing five-second untracked-file freshness floor and `.git/index` change detection shall remain enabled. Traces: UC-006.
- **REQ-016**: Command-backed custom file suggestions shall not start the built-in file index. Traces: UC-002.

## v1 Non-Functional Requirements

- **REQ-017**: For each independent requirement, representative transitions should remain below 16ms on the development machine; no observed frame may approach the reported 500ms stall.
- **REQ-018**: Warm `@` searches shall complete in memory without awaiting Git subprocess completion.
- **REQ-019**: Keyboard input entered during background refresh shall render normally rather than appearing in one buffered batch after refresh.
- **REQ-020**: Each requirement branch shall contain direct unit tests for only its behavior and required shared infrastructure.
- **REQ-021**: On each requirement branch independently, `bun run typecheck`, `bun run build`, and that requirement's focused tests shall pass.
- **REQ-022**: For each requirement branch independently, `bun run check` normalized failure names shall be a subset of that branch's exact `main` baseline and its failure count shall not increase. Existing baseline failures may be fixed, but any branch-only failure name is blocking; pass totals may differ when regression tests are added.
- **REQ-023**: No new runtime dependency, Node version change, or Bun workflow change shall be introduced.

## Out of Scope / Deferred

- Filesystem watchers for index invalidation.
- Redesigning statusline command input or output format.
- Changing exit-confirmation timing.
- Resolving unrelated `main` test failures.
- General React/Ink performance refactoring outside the prompt and suggestion paths.

## Traceability

| Requirement | Use Cases | Planned Phase |
|-------------|-----------|---------------|
| REQ-001 through REQ-002 | UC-001 | IR-001 `/` |
| REQ-003 through REQ-009 | UC-003, UC-004, UC-008, UC-009 | IR-002 Ctrl+C |
| REQ-010 through REQ-016, REQ-018, REQ-019 | UC-002, UC-005 through UC-007 | IR-003 `@` |
| REQ-017, REQ-020 through REQ-023 | All applicable use cases | Each independent branch |
