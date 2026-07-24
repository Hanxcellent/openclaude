# Use Cases:

## Actors and Roles

| Actor / Role | Description | Allowed Operations |
|--------------|-------------|--------------------|
| CLI Operator | Person operating the OpenClaude terminal prompt | All operations listed in UC-001 through UC-009 |

## Use Case Matrix

| ID | Actor | Operation | Domain Concepts | Expected Outcome |
|----|-------|-----------|-----------------|------------------|
| UC-001 | CLI Operator | Narrow or dismiss `/` suggestions | Prompt Session, Transient Prompt Surface, Footer Service | Input remains responsive and custom statusline is not remounted or redundantly executed |
| UC-002 | CLI Operator | Narrow, navigate, accept, suppress, or dismiss `@` suggestions | File Suggestion Index, Index Refresh | Search responds from memory; completion upgrades only an active query without reopening accepted, suppressed, or dismissed suggestions |
| UC-003 | CLI Operator | Press Ctrl+C once with text present | Prompt Session, Custom Statusline Update | Text clears immediately; the delayed exit hint transition does not stall input |
| UC-004 | CLI Operator | Open and close help or paste text | Transient Prompt Surface, Footer Service | Hidden services retain state but suspend hidden-only side effects |
| UC-005 | CLI Operator | Use `@` shortly after startup | File Suggestion Index | Prewarm provides full or progressive results without awaiting discovery |
| UC-006 | CLI Operator | Create an untracked file and later use `@` | Index Refresh | File becomes discoverable through the existing five-second freshness behavior |
| UC-007 | CLI Operator | Run `/clear`, then use `@` | Lifecycle Subscription, File Suggestion Index | Index rebuild completion automatically refreshes the active query |
| UC-008 | CLI Operator | Hide footer during a pending statusline debounce | Custom Statusline Update | Timer and command are cancelled; no hidden command starts |
| UC-009 | CLI Operator | Reactivate footer after hidden changes | Custom Statusline Update | Exactly one immediate refresh executes |

## Independent Requirement Mapping

| Requirement | Primary Use Cases | Review Branch | Dependency |
|-------------|-------------------|---------------|------------|
| IR-001 `/` | UC-001 | `fix/input-freeze-slash` | None; base is `main` |
| IR-002 Ctrl+C | UC-003, UC-004, UC-008, UC-009 | `fix/input-freeze-ctrl-c` | None; base is `main` |
| IR-003 `@` | UC-002, UC-005, UC-006, UC-007 | `fix/input-freeze-at` | None; base is `main` |

## Derived Access Rules

No role-based access changes exist. The CLI Operator is allowed to perform all listed operations. Missing operations are outside this specification rather than denied.

## UAT Trace

- UC-001 → UAT-01
- UC-002 → UAT-02, UAT-03, UAT-08
- UC-003 → UAT-04
- UC-004 → UAT-05
- UC-005 → UAT-02
- UC-006 → UAT-06
- UC-007 → UAT-07
- UC-008 → automated statusline state-machine tests
- UC-009 → automated statusline state-machine tests and UAT-01/UAT-04
