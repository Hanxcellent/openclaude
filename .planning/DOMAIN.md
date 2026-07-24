# Domain Model:

## Interaction Gate

- **Status**: Required, lightweight
- **Reason**: A human operator interacts with a stateful terminal prompt. No business-role or authorization model changes are involved.

## Core Concepts

| Concept | Definition | Lifecycle / Invariant |
|---------|------------|-----------------------|
| CLI Operator | Person typing into the OpenClaude TUI | Must observe continuous input response |
| Prompt Session | Mounted input and footer for the active REPL | Survives transient prompt surfaces |
| Transient Prompt Surface | Suggestions, help, exit, paste, or history UI | Changes visibility, not service ownership |
| Footer Service | Long-lived footer component or hook | Remains mounted; hidden side effects suspend |
| Custom Statusline Update | Debounced external statusline execution | Follows explicit active/dirty state transitions |
| File Suggestion Index | Progressively built in-memory path index | Queries never await full build |
| Index Refresh | Single-flight repository discovery operation | Commits state before notifying listeners |
| Lifecycle Subscription | Component-owned index completion listener | Survives cache reset; ends on owner unmount |

## Top-Level Concepts

### CLI Operator

The person typing into the OpenClaude prompt and expecting keystrokes, suggestions, feedback, and commands to remain responsive.

### Prompt Session

The mounted prompt input and footer for the active REPL session. It owns text input, cursor state, transient feedback, and suggestion state.

### Transient Prompt Surface

Short-lived UI that temporarily replaces or overlays normal footer content:

- Slash-command suggestions
- File and agent `@` suggestions
- Help menu
- Ctrl+C exit confirmation
- Paste feedback
- History search

### Footer Service

A long-lived component or hook whose lifecycle must not be coupled to transient visibility:

- Custom statusline command runner
- Built-in statusline
- Mode indicator and PR status polling
- File-index completion subscriber
- Quick Open completion subscriber

### Custom Statusline Update

A debounced external command execution caused by initial activation, assistant-message changes, permission/model/vim changes, or statusline configuration reload.

Lifecycle states:

1. `inactive` — mounted but hidden; no timer or command may remain active.
2. `active-clean` — visible with current output and no pending refresh.
3. `active-scheduled` — one debounced update is pending.
4. `active-running` — one command is in flight.
5. `inactive-dirty` — hidden after a relevant change; exactly one refresh is required on reactivation.

### File Suggestion Index

An in-memory progressively built index containing project paths. It is populated from tracked files, untracked files, relevant config files, and derived directory names.

### Index Refresh

A single-flight asynchronous repository discovery operation. It may be initiated by startup prewarm, cache reset, `.git/index` change, or the five-second freshness floor.

### Lifecycle Subscription

A component-owned listener that observes index completion and re-runs the latest search. It survives cache-data resets and ends only when its owning component unmounts or disables built-in file suggestions.

## Relationships

- The **CLI Operator** acts through the **Prompt Session**.
- A **Transient Prompt Surface** changes visibility but must not destroy a **Footer Service**.
- The custom statusline is a **Footer Service** that owns **Custom Statusline Updates**.
- `@` suggestions query the **File Suggestion Index**.
- An **Index Refresh** updates the index and notifies **Lifecycle Subscriptions**.
- `/clear` resets index data but does not end lifecycle subscriptions.

## Invariants

1. A visibility change does not imply a lifecycle reset.
2. An inactive custom statusline has no pending timer and no in-flight command.
3. Reactivating an inactive-dirty statusline executes exactly one immediate refresh.
4. A visible ordinary state change produces at most one debounced refresh.
5. An index refresh is single-flight.
6. A file query may return partial or empty results while an index builds, but it does not await the full build.
7. Index completion re-runs the latest relevant query.
8. Clearing caches does not clear lifecycle subscriptions.
9. Periodic untracked-file discovery remains enabled.
