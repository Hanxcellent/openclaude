# OpenClaude CLI Input Responsiveness Fix

## What This Is

This specification defines three independent reliability corrections for OpenClaude's React/Ink CLI prompt: slash-suggestion disappearance, Ctrl+C feedback transitions, and `@` file suggestions. Each requirement is specified, implemented, reviewed, tested, and releasable directly from `main` without depending on either of the other two.

## Core Value

Prompt input must remain immediately responsive while transient UI and background indexing change state.

## Requirements

### Domain / Interaction Gate

- **Interaction gate**: Required, lightweight
- **Reason**: A human operator directly types into a TUI and observes command suggestions, file mentions, statusline output, and exit feedback.
- **Domain model**: `.planning/DOMAIN.md`
- **Use cases**: `.planning/USE_CASES.md`

### Validated

None yet — implementation and UAT are required.

### Active

- [ ] **IR-001 `/`**: Suggestion visibility transitions preserve the regular footer subtree and do not stall input. See `.planning/requirements/01-slash-suggestion-freeze.md`.
- [ ] **IR-002 Ctrl+C**: Delayed feedback preserves footer state while cancelling hidden statusline work. See `.planning/requirements/02-ctrl-c-feedback-freeze.md`.
- [ ] **IR-003 `@`**: Progressive file indexing and replay lifecycle remain non-blocking, fresh, and interaction-safe. See `.planning/requirements/03-at-suggestion-freeze.md`.
- [ ] Each requirement independently passes focused checks and introduces no `bun run check` failure names beyond its own exact `main` baseline.

### Out of Scope

- Redesigning the prompt, suggestion menu, statusline format, or keybindings — this work preserves existing UX.
- Changing the 800ms double-press exit window — timing semantics are not the cause being corrected.
- Removing the five-second untracked-file refresh — stale untracked-file discovery is an unacceptable trade-off.
- Fixing unrelated failures already present on `main` — verification compares against the baseline failure set.
- Adding dependencies or changing Node/Bun runtime requirements — existing stack is sufficient.

## Context

Reported stalls occur after clearing input with Ctrl+C, when `/` or `@` suggestions disappear, and while file discovery refreshes. During a stall, the TUI does not process visible input, then applies buffered characters at once. The implementation area spans `PromptInputFooter`, `PromptInputFooterLeftSide`, `StatusLine`, `useTypeahead`, and `fileSuggestions`.

The target repository uses TypeScript, React, Ink, Bun for builds/tests, and Node.js for the built CLI. Custom statuslines execute user-configured external commands. File suggestions combine tracked files, untracked files, config files, MCP resources, and agents.

## Constraints

- **Compatibility**: Preserve existing `/`, `@`, Ctrl+C, help, paste, statusline, Quick Open, and `/clear` behavior.
- **Performance**: Keyboard-driven searches use in-memory state and must not await full file-index construction.
- **Freshness**: Preserve periodic untracked-file discovery and `.git/index`-driven tracked-file refresh.
- **Lifecycle**: Cache resets must not destroy component-owned subscriptions.
- **Side effects**: Hidden footer services must remain mounted where needed but suspend timers, external processes, focus, polling, and subscriptions that should not run while hidden.
- **Testing**: Behavior changes require direct state-machine and lifecycle regression tests.
- **Baseline**: `bun run check` may retain failures already present on `main`, but may not add failure names or increase the failure count.
- **Independence**: Every review branch is based directly on `main`, contains one requirement commit, and must compile and pass its gates without either sibling requirement.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Keep long-lived footer services mounted and explicitly gate activity | Prevent remount-triggered commands and initialization while retaining state | — Pending |
| Prewarm file indexing and allow progressive queries | Prevent the first `@` query from awaiting repository discovery | — Pending |
| Preserve the five-second refresh | Avoid trading responsiveness for stale untracked-file suggestions | — Pending |
| Treat signal subscriptions as lifecycle state, not cache data | `/clear` should reset data without disconnecting mounted consumers | — Pending |
| Model statusline scheduling as one explicit state machine | Avoid effect-order dependence, duplicate runs, and hidden commands | — Pending |
| Keep `/`, Ctrl+C, and `@` as separate review units | Prevent unrelated behavior and trade-offs from being accepted as one patch | — Pending |

---
*Last updated: 2026-07-11 after specification initialization*
