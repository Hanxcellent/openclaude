# Independent Requirement: Slash Suggestion Freeze

## Identity

- **Requirement ID**: IR-001
- **Review branch**: `fix/input-freeze-slash`
- **Base branch**: `main`
- **Implementation dependency**: None
- **Review status**: Proposed implementation; not accepted

## Problem

When slash-command suggestions disappear after a command becomes complete, gains an argument space, or stops matching, the prompt footer can remount long-lived children. A custom statusline command or another expensive footer initializer can then block the TUI, causing typed characters to appear in a batch after the stall.

## Required Behavior

1. Showing or hiding inline `/` suggestions shall not unmount the regular prompt footer subtree.
2. Hidden footer content shall collapse to zero rows without clipping visible wrapped content.
3. The change shall not alter slash-command matching, selection, acceptance, argument hints, or command execution.
4. This requirement shall be reviewable and releasable without the Ctrl+C or `@` changes.

## Scope

- `src/components/PromptInput/KeepMounted.tsx`
- `src/components/PromptInput/PromptInputFooter.tsx`
- Direct tests for mount preservation, layout collapse, and footer statusline resolution

## Non-Goals

- Ctrl+C feedback and statusline cancellation
- File indexing, `@` suggestions, Quick Open, or `/clear`
- Changing command suggestion ranking or debounce timing

## Automated Acceptance

```bash
bun test src/components/PromptInput/KeepMounted.test.tsx src/components/PromptInput/PromptInputFooter.test.tsx
bun run typecheck
bun run build
bun run check
```

For `bun run check`, normalize failure names and compare against an isolated run of the exact `main` commit. `fix-only` must be empty and the failure count must not increase.

## Manual UAT

1. Type `/ski` while a matching slash command is visible.
2. Type another character so the match disappears.
3. Repeat by completing a command and typing a trailing space.
4. Continue typing through both transitions.

**Pass**: Suggestions disappear immediately, input remains visible without batching, footer layout does not add a blank row, and a configured custom statusline is not restarted by the transition.

