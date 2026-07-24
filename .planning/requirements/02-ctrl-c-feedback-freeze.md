# Independent Requirement: Ctrl+C Feedback Freeze

## Identity

- **Requirement ID**: IR-002
- **Review branch**: `fix/input-freeze-ctrl-c`
- **Base branch**: `main`
- **Implementation dependency**: None
- **Review status**: Proposed implementation; not accepted

## Problem

After Ctrl+C clears non-empty prompt input, delayed exit feedback temporarily replaces footer content. Conditional unmounting and statusline side effects can restart or execute external work around that transition, stalling the TUI after the clear operation.

## Required Behavior

1. Ctrl+C shall clear the prompt immediately and preserve existing exit-confirmation timing.
2. Exit and paste feedback shall hide, not unmount, long-lived left-footer content and configured statusline state.
3. Hiding the custom statusline shall cancel pending debounce work and abort in-flight commands.
4. Changes observed while hidden shall mark the statusline dirty without running a hidden command.
5. Reactivation shall run at most one immediate refresh.
6. An aborted first-run or hot-reload command shall not consume its pending result-log marker.
7. This requirement shall be reviewable and releasable without the `/` or `@` changes.

## Scope

- `src/components/PromptInput/KeepMounted.tsx`
- `src/components/PromptInput/PromptInputFooter.tsx`
- `src/components/PromptInput/PromptInputFooterLeftSide.tsx`
- `src/components/StatusLine.tsx`
- Direct layout, footer resolver, and statusline state-machine tests

## Non-Goals

- Slash-command suggestion lifecycle
- File indexing, `@` suggestions, Quick Open, or `/clear`
- Changing the Ctrl+C double-press window or keybindings

## Automated Acceptance

```bash
bun test src/components/PromptInput/KeepMounted.test.tsx src/components/PromptInput/PromptInputFooter.test.tsx src/components/StatusLine.test.ts
bun run typecheck
bun run build
bun run check
```

For `bun run check`, normalize failure names and compare against an isolated run of the exact `main` commit. `fix-only` must be empty and the failure count must not increase.

## Manual UAT

1. Enter non-empty prompt text.
2. Press Ctrl+C once.
3. Immediately type continuously for at least one second through the delayed feedback transition.
4. Repeat with a configured custom statusline command and inspect debug command counts.

**Pass**: Old text clears immediately, new text appears continuously, no delayed batch appears, the footer adds no blank row, and no hidden or duplicate statusline command runs.

