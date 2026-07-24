# Roadmap:

## Milestone

**CLI Input Responsiveness Independent Fixes v0.1**

Goal: review and decide three prompt responsiveness corrections independently. Acceptance or rejection of one requirement must not determine either sibling requirement.

## Phases

### Review Unit 1: Slash Suggestion Disappearance

**Requirement**: IR-001

**Specification**: `.planning/requirements/01-slash-suggestion-freeze.md`

**Review branch**: `fix/input-freeze-slash`, based directly on `main`

**Requirements**: REQ-001, REQ-002, REQ-017, REQ-020, REQ-021, REQ-022, REQ-023

**Success criteria**:

1. Slash suggestion visibility does not unmount the regular footer subtree.
2. Hidden content collapses without blank rows or clipping.
3. Slash matching and command behavior remain unchanged.
4. Focused checks pass and `bun run check` has no branch-only failure names.

**Depends on**: none

### Review Unit 2: Ctrl+C Delayed Feedback

**Requirement**: IR-002

**Specification**: `.planning/requirements/02-ctrl-c-feedback-freeze.md`

**Review branch**: `fix/input-freeze-ctrl-c`, based directly on `main`

**Requirements**: REQ-003, REQ-004, REQ-005, REQ-006, REQ-007, REQ-008, REQ-009, REQ-017, REQ-020, REQ-021, REQ-022, REQ-023

**Success criteria**:

1. Ctrl+C feedback does not remount long-lived footer or configured statusline state.
2. Hidden statusline work is cancelled and no hidden command starts.
3. Reactivation performs at most one immediate refresh.
4. Ctrl+C timing and keybindings remain unchanged.
5. Focused checks pass and `bun run check` has no branch-only failure names.

**Depends on**: none

### Review Unit 3: At-Sign File Suggestions

**Requirement**: IR-003

**Specification**: `.planning/requirements/03-at-suggestion-freeze.md`

**Review branch**: `fix/input-freeze-at`, based directly on `main`

**Requirements**: REQ-010, REQ-011, REQ-012, REQ-013, REQ-014, REQ-015, REQ-016, REQ-017, REQ-018, REQ-019, REQ-020, REQ-021, REQ-022, REQ-023

**Success criteria**:

1. Cold and refreshing `@` queries do not await repository discovery.
2. Refresh remains single-flight and preserves untracked-file freshness.
3. Only active queries replay; dismissal, acceptance, suppression, and input changes invalidate stale work.
4. Selection remains stable and Quick Open replay does not recursively refresh.
5. `/clear` preserves mounted completion subscribers.
6. Focused checks pass and `bun run check` has no branch-only failure names.

**Depends on**: none

### Independent Verification Gate

Run `.planning/phases/03-verification/03-VALIDATION.md` separately for each review branch. Never validate a combined or stacked branch as evidence for an independent requirement.

For every branch:

1. Record the exact `main` commit.
2. Run focused tests, typecheck, and build.
3. Run `bun run check` on the branch and an isolated worktree at the same `main` commit.
4. Require `fix-only=0` after failure-name normalization and no increased failure count.
5. Run only that requirement's manual UAT.

## Progress

| Review Unit | Status | Dependency |
|-------------|--------|------------|
| IR-001 `/` | Ready for independent review | None |
| IR-002 Ctrl+C | Ready for independent review | None |
| IR-003 `@` | Ready for independent review | None |

Overall acceptance is tracked per requirement; there is no combined completion percentage.
