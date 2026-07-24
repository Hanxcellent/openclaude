# External Integrations

## Model Providers

Provider and gateway support is split between metadata in `src/integrations/` and request execution in `src/services/api/`.

Major categories include:

- Anthropic first-party APIs.
- OpenAI and OpenAI-compatible gateways.
- Gemini and Vertex AI.
- DeepSeek, MiniMax, xAI, Fireworks, Groq, OpenRouter and other gateways.
- Local runtimes such as Ollama and compatible endpoints.
- OAuth-backed routes including Codex, GitHub and xAI flows.

Credentials are sourced from environment variables, secure storage, settings profiles and managed configuration. Route selection must explicitly sanitize incompatible ambient variables.

## MCP

`src/services/mcp/` manages configured servers, transports, registry discovery, authentication and approval. MCP capabilities can contribute tools, resources and prompts to interactive sessions and SDK/headless modes.

## Plugins and Skills

- Plugins can add commands, agents, skills, hooks and MCP servers.
- Skills are discovered from bundled, user, project, plugin and additional-directory sources.
- Precedence and trust boundaries matter because user/project content may override bundled names.
- File watchers and change detectors refresh settings and skill/plugin state during a session.

## Development and IDE Integrations

- VS Code extension provides IDE-facing workflows and at-mention context.
- LSP services support language-aware tools.
- Git and GitHub CLI integrations provide repository, PR and worktree behavior.
- Terminal multiplexers and SSH/remote modes support distributed or background sessions.

## Web and Artifact Services

- Web search supports native provider search and adapter providers.
- Web fetch includes SSRF controls and optional browser/Firecrawl paths.
- File/document/image utilities use external libraries and optional runtime modules.

## Operational Integrations

- Remote managed settings and policy limits can constrain runtime behavior.
- Analytics interfaces exist, although telemetry initialization is currently disabled/no-op in parts of the open distribution.
- Auto-update, package registry and provider discovery paths perform network access and require careful startup deferral.

## Integration Risks

1. Shared environment variables can accidentally route a request through the wrong provider.
2. Optional modules must fail safely when absent from the published bundle.
3. Plugin/skill/MCP content crosses trust boundaries and requires validation and approval.
4. Network and subprocess integrations must not block the TUI event loop.
5. Provider schemas and streaming semantics differ despite compatibility labels.
