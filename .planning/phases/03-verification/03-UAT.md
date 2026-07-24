# User Acceptance Testing

## Status

Not executed. Run only the cases mapped to the independent requirement branch under review.

## Setup

1. Use a real interactive terminal.
2. Build and launch the development CLI:

```bash
bun run build
node bin/openclaude --debug
```

3. Enable a custom statusline command representative of normal user configuration.
4. Use a Git repository containing tracked and untracked files.

## Cases

### UAT-01 — Slash suggestion disappears

**Trace**: UC-001, REQ-001, REQ-006

1. Type `/ski` when `/skills` or another matching command exists.
2. Type the next character so the command no longer matches, or complete a command and add a space.
3. Continue typing during the transition.

**Pass**: Suggestions disappear immediately; every character appears without pause or batching; no redundant statusline command runs.

### UAT-02 — First `@` query after startup

**Trace**: UC-005, REQ-010, REQ-011

1. Start the CLI and immediately type `@` plus a partial filename.
2. Continue typing while the index may still be building.

**Pass**: Input remains responsive. Partial or initially empty results are acceptable, but results upgrade automatically when indexing completes.

### UAT-03 — `@` suggestion disappears during refresh

**Trace**: UC-002, REQ-011, REQ-012

1. Wait more than five seconds after the previous index refresh.
2. Type an `@` query that initially matches files.
3. Continue to a non-matching query while typing additional characters.

**Pass**: Suggestion disappearance and background Git activity do not pause input; only one refresh is observed.

### UAT-04 — Ctrl+C clear and delayed feedback transition

**Trace**: UC-003, REQ-001, REQ-006, REQ-008

1. Enter non-empty text.
2. Press Ctrl+C once.
3. Immediately type continuously for at least one second, spanning the exit-confirmation timeout.

**Pass**: Original text clears immediately; new text appears continuously; no delayed batch appears when feedback disappears.

### UAT-05 — Help and paste transient surfaces

**Trace**: UC-004, REQ-002, REQ-003

1. Open and close prompt help.
2. Paste multiline text.
3. Observe footer layout and continue typing.

**Pass**: No blank row, clipped visible content, focus theft, polling restart, or input pause occurs.

### UAT-06 — Untracked-file freshness retained

**Trace**: UC-006, REQ-015

1. Create a new untracked file without running `git add`.
2. Wait through the existing freshness interval.
3. Search for it with `@`.

**Pass**: The new file appears without restarting the CLI or modifying `.git/index`.

### UAT-07 — `/clear` index recovery

**Trace**: UC-007, REQ-014

1. Run `/clear` while the prompt remains mounted.
2. Enter an `@` query immediately after clear.
3. Do not type an extra character after index completion.

**Pass**: Results automatically refresh when indexing completes; no remount or extra keystroke is required.

### UAT-08 — `@` replay lifecycle

**Trace**: UC-002, REQ-013

1. While an `@` query is visible and indexing is incomplete, move the selection away from the first item and wait for completion.
2. Repeat the query and dismiss it with Escape before completion.
3. Repeat and accept a file with Enter or Tab before completion.
4. Repeat and enter Ctrl+R history search before completion.

**Pass**: The selected item remains selected during an active upgrade. Dismissed, accepted, and history-suppressed suggestions do not reappear when indexing completes.

## Result Table

| Case | Status | Observed delay | Notes |
|------|--------|----------------|-------|
| UAT-01 | Not run | - | |
| UAT-02 | Not run | - | |
| UAT-03 | Not run | - | |
| UAT-04 | Not run | - | |
| UAT-05 | Not run | - | |
| UAT-06 | Not run | - | |
| UAT-07 | Not run | - | |
| UAT-08 | Not run | - | |

## Acceptance Rule

Acceptance is independent:

- **IR-001 `/`** requires UAT-01.
- **IR-002 Ctrl+C** requires UAT-04 and the paste transition in UAT-05.
- **IR-003 `@`** requires UAT-02, UAT-03, UAT-06, UAT-07, and UAT-08.

A failure blocks only its mapped requirement unless the same behavior is independently reproduced on another branch. Any visible pause, buffered-input batch, stale untracked file after the refresh interval, duplicate statusline command, duplicate index scan, replayed inactive suggestion, selection jump, or missing automatic post-`/clear` refresh is blocking for the applicable requirement.
