# Project State

## Project Reference

See: `.planning/PROJECT.md` and `.planning/requirements/`.

**Core value:** Prompt input remains immediately responsive during the affected transition.
**Current focus:** Independent review of IR-001, IR-002, and IR-003.

## Current Position

The work is split into three sibling review units. Each starts from `main`, has no dependency on the others, and receives an independent acceptance decision.

| Requirement | Branch | Status |
|-------------|--------|--------|
| IR-001 `/` | `fix/input-freeze-slash` | Ready for review |
| IR-002 Ctrl+C | `fix/input-freeze-ctrl-c` | Ready for review |
| IR-003 `@` | `fix/input-freeze-at` | Ready for review |

## Accumulated Context

### Decisions

- Do not use stacked branches as acceptance evidence.
- Keep each review branch directly based on the exact `main` commit.
- Apply focused tests and manual UAT only to the requirement under review.
- Preserve five-second untracked-file refresh in IR-003.
- Compare full-suite failures by normalized test-name set against `main`; `fix-only` must be empty and failure count must not increase.
- Existing `main` failures may disappear; added passing tests need not keep pass totals equal.

### Blockers/Concerns

- The local `main` baseline currently contains 92 full-suite failures and the suite is timing-sensitive; rerun exact branch/main pairs when a flaky branch-only failure appears.
- Manual terminal responsiveness remains required because buffered input is not fully represented by unit tests.
- Shared file names between IR-001 and IR-002 do not imply a branch dependency; each branch carries only the infrastructure needed for its own behavior.

### Deferred Items

| Category | Item | Status |
|----------|------|--------|
| Indexing | Filesystem watcher-based invalidation | Out of scope |
| Cleanup | Unrelated `main` test failures | Out of scope |
| Integration | Combining accepted independent requirements | Deferred until individual review decisions |

## Session Continuity

Last activity: 2026-07-11
Stopped at: Three independent requirements specified and ready for separate review
Resume file: None
