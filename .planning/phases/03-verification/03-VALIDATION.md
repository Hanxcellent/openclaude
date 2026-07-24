# Validation Contract

## Status

Not executed. This document defines the required evidence for the not-yet-implemented fix.

## Automated Gates

### Type and Build

| ID | Command | Pass condition | Requirements |
|----|---------|----------------|--------------|
| VAL-01 | `bun run typecheck` | Exit 0 | REQ-021, REQ-023 |
| VAL-02 | `bun run build` | Exit 0; CLI and SDK bundle guards pass | REQ-021, REQ-023 |

### Focused Tests

Run:

```bash
bun test \
  src/components/StatusLine.test.ts \
  src/components/PromptInput/KeepMounted.test.tsx \
  src/components/PromptInput/PromptInputFooter.test.tsx \
  src/components/PromptInput/PromptInputFooterLeftSide.test.ts \
  src/hooks/fileSuggestions.test.ts \
  src/hooks/useTypeahead.test.ts \
  src/components/QuickOpenDialog.test.ts
```

Required named coverage:

| Behavior | Required evidence |
|----------|-------------------|
| Mounted-hidden lifecycle | Child mounts once across visible-hidden-visible and unmounts with parent |
| Hidden layout | No blank row; visible wrapped content is not clipped |
| Footer state source | One resolver covers mounted, visible, and custom-active values |
| Hidden statusline | Pending work resolves to cancel and dirty |
| First activation | Pending changes resolve to exactly one immediate run |
| Dirty reactivation | Exactly one immediate run |
| Visible update | Ordinary change resolves to one debounce schedule |
| Prewarm | Production starts refresh and subscribes; test mode skips real Git work |
| Custom provider | Command-backed file suggestions do not start built-in indexing |
| Non-blocking query | Query returns while controlled Git child remains unresolved |
| Refresh reentrancy | Completion subscriber cannot increase tracked spawn count above one |
| `/clear` recovery | Pre-clear subscriber receives post-clear completion |
| Quick Open | Current query reruns on index completion |

Pass condition: all focused tests exit 0.

## Full-Suite Baseline Gate

Run this gate independently for each requirement branch. A passing result on a combined or stacked branch is not evidence for IR-001, IR-002, or IR-003.

| Requirement | Branch | Focused Scope |
|-------------|--------|---------------|
| IR-001 | `fix/input-freeze-slash` | Slash footer mount and layout tests |
| IR-002 | `fix/input-freeze-ctrl-c` | Ctrl+C footer and StatusLine tests |
| IR-003 | `fix/input-freeze-at` | Typeahead, file index, and Quick Open tests |

1. Record the exact `main` commit.
2. Run `bun run check` on `main` and exactly one requirement branch with the same environment.
3. Extract normalized failure names:

```bash
rg '^\(fail\) ' check.log | sed -E 's/ \[[0-9.]+ms\]$//' | sort -u
```

4. Compare sets with `comm`.

Pass conditions:

- `smoke` passes on the implementation.
- `deadcode` does not add an implementation error.
- Implementation failure count is not greater than `main`.
- `comm -23 implementation-fails main-fails` returns no lines.
- Existing `main` failures may be resolved, so `comm -13 implementation-fails main-fails` may contain lines.
- Pass totals may differ because the implementation can add regression tests; pass-count equality is not required.
- Record and decide each requirement independently; never union failure evidence from sibling branches.

Known initialization baseline: local `main` was observed with 92 failures. This number is context, not a permanent exemption; rerun both refs at verification time.

## Runtime Performance Gate

Run the built CLI with debug and frame timing enabled in a real terminal.

Capture these cases:

1. `/ski` to `/skik` or equivalent suggestion disappearance.
2. `@` query to a non-matching value after the five-second freshness floor.
3. Ctrl+C with non-empty input, followed by continuous typing through the feedback timeout.
4. Help menu open/close.

Pass conditions:

- Maximum total frame duration below 16ms on the development machine.
- No approximately 500ms gap in visible input.
- One statusline command execution per legitimate active transition.
- No statusline command starts while inactive.
- One tracked-file scan per refresh cycle.
- Warm query timing is independent of Git child completion.

## Requirement Evidence Matrix

| Requirements | Evidence |
|--------------|----------|
| REQ-001..03 | KeepMounted, footer resolver, focus tests, UAT-01/04/05 |
| REQ-004..06 | Statusline transition tests, debug command count |
| REQ-010..07 | Typeahead/index/Quick Open tests, UAT-02/03/06/07 |
| REQ-017..03 | Frame log, debug log, manual typing observations |
| REQ-020..03 | Focused tests, build/typecheck, main comparison |
| REQ-023 | Dependency diff review and successful build |

## Completion Record

Fill during Phase 3:

| Gate | Result | Evidence location |
|------|--------|-------------------|
| Typecheck | Not run | - |
| Build | Not run | - |
| Focused tests | Not run | - |
| Main comparison | Not run | - |
| Runtime frames | Not run | - |
| UAT | Not run | `03-UAT.md` |
