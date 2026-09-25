# Claude Code — Field Notes

Independent, observational documentation of how **Claude Code** (the CLI)
*behaves* at runtime: how it resolves models, decides its context window, fires
auto-compaction, assembles requests, runs tools/hooks/MCP, gates permissions,
counts tokens, and what every environment variable does.

Written for **operators** — people running Claude Code through a gateway or
router, tuning context and cost, writing hooks, or locking down egress — who need
to know *how it behaves* and *which knobs control it*.

> ### Scope, method & caveats
>
> - **This documents observable behavior**, for the purpose of operating and
>   integrating with the tool correctly. It is **not** a source release and ships
>   **no** binary, decompiled module, proprietary asset, or credential.
> - **Minified identifiers** (`jf`, `dO7`, `w37`, `xM`, …) and the short code
>   fragments shown under each doc's *Evidence* section are **version-specific
>   citations** — the smallest excerpt needed to substantiate a behavioral claim,
>   so a reader can verify it. They change every release and are meaningless
>   outside the named version.
> - **Not affiliated with, endorsed by, or representing Anthropic.** Claude Code
>   is Anthropic's product; this is independent field documentation. For official
>   guidance, always defer to Anthropic's own docs.
> - **Behavior drifts between releases.** Every claim is tagged with the version
>   it was checked against. Treat anything unverified for your version as a
>   hypothesis, not fact.

## How to read these notes

Each document is **behavior-first**: plain-language description of what Claude
Code does and which setting/env var controls it, written so you can act on it
without reading any code. Where a claim is load-bearing, a compact **Evidence**
block cites the version-specific fragment it was verified against. If you only
want to operate the tool, read the prose and skip the Evidence blocks.

## Map

### Context, budgets & cost
- [Context window & auto-compaction](docs/context-window-and-auto-compact.md) — how the 1M window is gated (the `[1m]` model-name suffix; `ANTHROPIC_1M_CONTEXT` is a no-op), when auto-compaction fires, and the per-session 1M rate-limit latch.
- [Thinking & effort budget](docs/thinking-and-effort.md) — effort levels → thinking-token budget → request params.
- [Usage, cost & token counting](docs/usage-cost-tokens.md) — how context is counted locally vs from API usage; cost.

### Models & routing
- [Model resolution & aliases](docs/model-resolution.md) — how `--model`/aliases/`[1m]` become a concrete wire id; provider modes.
- [Background & subagent models](docs/bg-and-subagent-models.md) — which traffic uses the small/background model vs the main model; how subagents pick a model.

### Requests
- [Transport & headers](docs/transport-and-headers.md) — the `anthropic-beta` registry, per-request beta selection, retry/backoff, SSE; what a gateway actually receives.

### Tools & extensions
- [Tools & deferred tool-search](docs/tools-and-deferral.md) — tool lifecycle and the deferred tool-search mechanism.
- [MCP](docs/mcp.md) — how MCP servers/tools are loaded, prefixed, and surfaced.
- [Hooks](docs/hooks.md) — every hook event, when it fires, and the stdin/stdout JSON contract.

### Sessions & configuration
- [Session, transcript & restore](docs/session-transcript-restore.md) — the JSONL transcript format, how `message.model` persists, what resume reads.
- [Settings precedence](docs/settings-precedence.md) — the managed/user/project/local merge order and how the `env` block applies.
- [Env var registry](docs/env-var-registry.md) — every notable `CLAUDE_CODE_*` / `ANTHROPIC_*` env var and its effect.

### Control surfaces
- [Permissions, auto-mode & sandbox](docs/permissions-automode-sandbox.md) — permission modes, the auto-mode classifier, allow/deny matching, the Bash sandbox.
- [Statusline](docs/statusline.md) — the exact JSON payload a custom statusline command receives.
- [Terminal color depth](docs/terminal-colors.md) — how the 24-bit vs 256-color decision is made, why WSL under Windows Terminal looks washed out, and why `FORCE_COLOR=3` doesn't fix it.
- [Telemetry & egress](docs/telemetry-and-egress.md) — every network egress and its kill switch (telemetry, OTEL, statsig, autoupdate).

## Versions

Notes are tagged inline with the Claude Code version each claim was checked
against (currently **2.1.178** and **2.1.179**). Identifiers and exact numbers
may differ on your version; the *behavior* is usually stable, the *symbols* are
not.
