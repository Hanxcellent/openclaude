# Independent Requirement: At-Sign Suggestion Freeze

## Identity

- **Requirement ID**: IR-003
- **Review branch**: `fix/input-freeze-at`
- **Base branch**: `main`
- **Implementation dependency**: None
- **Review status**: Proposed implementation; not accepted

## Problem

The first or refreshing `@` query can couple keyboard input to repository discovery. Index-completion replay can also reopen dismissed, accepted, or history-suppressed suggestions, disturb the current selection, or cause Quick Open to start redundant refreshes.

## Required Behavior

1. Built-in file indexing shall prewarm after typeahead mounts outside tests.
2. Cold and refreshing queries shall return progressive in-memory results without awaiting full Git discovery.
3. Refresh shall remain single-flight and retain the existing five-second untracked-file freshness floor and `.git/index` invalidation.
4. Completion shall replay only the latest active, unsuppressed query.
5. Escape dismissal, completed Enter/Tab acceptance, input changes, and Ctrl+R history suppression shall invalidate pending replay and in-flight results.
6. A selected partial result shall remain selected while complete-index results are merged.
7. `/clear` shall reset cache data without removing mounted completion subscribers.
8. Quick Open shall query the committed index after completion without starting another refresh or replacing equal result arrays.
9. Command-backed custom file suggestions shall not start the built-in index.
10. This requirement shall be reviewable and releasable without the `/` or Ctrl+C changes.

## Scope

- `src/hooks/fileSuggestions.ts`
- `src/hooks/useTypeahead.tsx`
- `src/components/QuickOpenDialog.tsx`
- Direct tests for prewarm, non-blocking queries, single-flight refresh, replay lifecycle, selection preservation, `/clear`, and Quick Open

## Non-Goals

- Slash-command suggestions or footer remount behavior
- Ctrl+C feedback or custom statusline scheduling
- Removing periodic untracked-file discovery or adding filesystem watchers

## Automated Acceptance

```bash
bun test src/hooks/useTypeahead.test.ts src/hooks/fileSuggestions.test.ts src/components/QuickOpenDialog.test.ts
bun run typecheck
bun run build
bun run check
```

For `bun run check`, normalize failure names and compare against an isolated run of the exact `main` commit. `fix-only` must be empty and the failure count must not increase.

## Manual UAT

1. Immediately after startup, type an `@` query while indexing is incomplete and continue typing.
2. Move selection away from the first partial result and wait for completion.
3. Repeat and separately dismiss with Escape, accept with Enter/Tab, and enter Ctrl+R before completion.
4. Run `/clear`, issue an `@` query, and do not type an extra character after completion.
5. Create an untracked file, wait through the freshness interval, and search for it.

**Pass**: Input never batches, active results upgrade automatically, selection stays stable, inactive queries never reopen, `/clear` recovers without another keystroke, and untracked-file freshness is preserved.

