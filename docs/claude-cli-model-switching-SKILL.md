---
name: claude-cli-model-switching
description: Reference for Claude Code CLI model slugs, aliases, and switching mechanisms specific to Claude Max ($100/5x, $200/20x) subscriptions. Load when an agent needs to select, pin, or switch the Anthropic model used by Claude Code CLI, configure effort levels / fallback models, or expose additional models through a local cli-bridge provider.
last_verified: 2026-07-20
---

# Claude Code CLI — Model Reference (Claude Max)

Source of truth: official docs only. This file is a cache — re-verify against the links below before acting if `last_verified` is more than ~30 days old, since Anthropic ships model updates frequently.

## Official documentation (verify here first)

- Model configuration: https://code.claude.com/docs/en/model-config
- Settings reference (settings.json, managed-settings.json, precedence): https://code.claude.com/docs/en/settings
- Models overview (API-side model list, deprecation dates): https://platform.claude.com/docs/en/about-claude/models/overview
- Claude Code model support article: https://support.claude.com/en/articles/11940350-claude-code-model-configuration
- Full docs index: https://code.claude.com/docs/llms.txt

## How to DISCOVER current options on any machine (never guess)

- `claude --help` — grep for `--model`, `--effort`, `--fallback-model`. This is the live, installed-version truth.
- `/model` in-session — interactive picker showing exactly what this account/org can select. Option+P (Alt+P) = quick picker shortcut.
- `/effort` in-session — shows valid effort levels for the current model.
- `/config` in-session — settings including model default.
- `/fast` in-session — toggle faster output mode (Opus 4.6/4.7/4.8 only).
- Cheap slug verification before registering anywhere: `claude --print --model <slug> "reply with exactly: ok"` — exit 0 + "ok" = slug accepted.

## Model aliases and slugs (Claude Max tier)

| Alias | Resolves to | Max availability | Notes |
|---|---|---|---|
| default | account-type or org default | 5x & 20x | clears override |
| best | Fable 5 if org has access, else latest Opus | 5x & 20x | |
| fable | claude-fable-5 | 5x & 20x | hardest/longest tasks |
| opus | claude-opus-4-8 | 5x & 20x | complex reasoning |
| sonnet | claude-sonnet-5 | 5x & 20x | daily coding |
| haiku | claude-haiku-4-5-20251001 | 5x & 20x | fast/simple |
| sonnet[1m] | Sonnet, 1M context | 20x (beta) | long sessions |
| opus[1m] | Opus, 1M context | 20x (beta) | long sessions |
| opusplan | Opus (plan) -> Sonnet (execute) | 5x & 20x | hybrid |

Pinned slugs seen in the wild (verify current list at the model-config link above before hardcoding):
claude-opus-4-8, claude-opus-4-7, claude-opus-4-6, claude-opus-4-5-20251101,
claude-sonnet-4-6, claude-sonnet-4-5-20250929, claude-fable-5, claude-haiku-4-5-20251001

## Switching mechanisms, precedence high -> low

1. `--model <alias|slug>` CLI flag — session only
2. `ANTHROPIC_MODEL` env var — session only (persists if exported in shell rc)
3. `/model` interactive picker — `Enter` = switch + save as default, `s` = session only
4. `model` key in `settings.json` (user `~/.claude/settings.json` or project `.claude/settings.json`) — persistent default
5. Managed/MDM `managed-settings.json` — org-enforced, overrides all of the above

## Effort and fallback controls

| Setting | Values | Scope |
|---|---|---|
| --effort flag | low, medium, high, xhigh, max, auto (xhigh = Opus 4.7/4.8 only; max = Opus 4.6+) | session |
| /effort slash | same | session, low/med/high persist |
| CLAUDE_CODE_EFFORT_LEVEL env var | level name or auto | session |
| effortLevel in settings.json | low, medium, high, xhigh (not max/ultracode) | persistent |
| --fallback-model | alias or comma-separated list, tried in order | print mode (-p) only |
| availableModels (settings.json array) | list of slugs | admin allowlist |
| enforceAvailableModels | bool | locks default option to allowlist |
| /fast on\|off | faster output mode | Opus 4.6/4.7/4.8, session |

## Alias-pinning env vars

- `ANTHROPIC_DEFAULT_OPUS_MODEL`, `ANTHROPIC_DEFAULT_SONNET_MODEL`, `ANTHROPIC_DEFAULT_HAIKU_MODEL`, `ANTHROPIC_DEFAULT_FABLE_MODEL` — pin what an alias resolves to (prevents silent version drift when Anthropic re-points aliases).
- `CLAUDE_CODE_SUBAGENT_MODEL` — model used by Claude Code's own subagents.
- `ANTHROPIC_SMALL_FAST_MODEL` — deprecated; replaced by `ANTHROPIC_DEFAULT_HAIKU_MODEL`.
- Alias resolution differs by provider (Anthropic API vs Bedrock vs Vertex vs Foundry) — verify provider before assuming a mapping.

## Generic pattern: a local cli-bridge provider

This pattern applies to any setup that fronts the Claude Code CLI with a small local HTTP proxy (e.g. this repo's `bridge.py`, launchd/systemd-managed), registered in an agent framework as a custom OpenAI-compatible provider pointing at `http://127.0.0.1:<port>/v1`.

Architecture facts, generalized from a working deployment:

- **The bridge's model list is advertisement only, zero enforcement.** `/v1/models` returns whatever the model-list env var (e.g. `CLI_BRIDGE_MODELS`) contains. Any model id in a request is passed straight through as `claude --print --model <slug>`. So the CLI's full alias/slug set is reachable even if not advertised.
- Config is env-driven — typical keys: bind host (keep it `127.0.0.1`, never `0.0.0.0`), port, path to the `claude` binary, model list, default model, request timeout. **SSOT caveat:** if the process runs under a supervisor (launchd/systemd), that supervisor's environment block is often the *effective* source, not a `.env` file — many bridges load `.env` no-override (`if key not in os.environ: ...`), so supervisor-injected values win and `.env` edits silently do nothing. Edit the supervisor's environment definition (e.g. via `plistlib` for launchd — never hand-edit the XML) and mirror the same values into `.env` so the manual-run path stays consistent.
- Bridge is typically stateless: one CLI subprocess per request, no credential storage; auth comes from the Claude Code CLI's own logged-in session.

### To expose more models through a bridge like this (3 steps)

1. **Verify each candidate slug first**: `claude --print --model <slug> "reply with exactly: ok"`. Register only slugs that pass.
2. **Edit the supervisor's environment config** (the effective SSOT when running under a process supervisor) — extend the model-list variable with the new slug(s), assert-before + read-back-after. **Then mirror the same list into `.env`** so the manual-run path stays consistent. A supervisor-level env change usually requires a full process reload, not just a restart-in-place, because a simple restart may not re-read the supervisor's injected environment. Verify with `curl -s http://127.0.0.1:<port>/v1/models`.
3. **Mirror into your agent framework's provider config**: most frameworks intersect the provider's advertised model list with their own local `models:` map, so new slugs must be added there too (with context length, e.g. 200000). Follow a standard safe-write protocol: backup the config file → edit programmatically (never with `sed`) → parse and read back to assert the exact key changed → diff key-by-key.

### Gotchas / lessons learned

- Aliases (`opus`, `sonnet`) auto-track latest — convenient interactively, but **silent model drift** for an orchestrator provider. For provider registration in an agent framework, prefer pinned full slugs.
- If only one model shows up for the bridge provider, the bottleneck is almost always the model-list env var and/or the framework's own model map — not the CLI, not the subscription.
- `--fallback-model` only works in `-p` (print) mode — which is exactly the mode a bridge typically uses, so it's usable per-request if wired into the bridge command.
- A well-built bridge serves both `/v1/models` and `/models` (base-URL-with-or-without-`/v1` footgun handled server-side).
- `effortLevel` in settings.json does NOT accept `max` — only low/medium/high/xhigh.
- `xhigh` effort = Opus 4.7/4.8 only; `max` = Opus 4.6/4.7/4.8 only. Other models reject them.
- Cheap health check pattern: a `/health` endpoint returning `{ok, version, host, port, cli_bin, models}`.

## Usage instructions for an agent

1. When a task requires selecting a model for a Claude Code CLI invocation, prefer the alias (opus/sonnet/haiku) over a pinned slug unless version stability is explicitly required. For provider registration in an agent framework, invert this: prefer pinned slugs.
2. For long-context tasks (>200k tokens), use `sonnet[1m]` or `opus[1m]` — confirm current beta status via the model-config link first.
3. Never assume alias->slug mapping without checking the `last_verified` date; if stale, re-fetch https://code.claude.com/docs/en/model-config before executing.
4. Persist a project-wide default via `.claude/settings.json` `model` key rather than per-session flags when consistency across a repo/team matters.
5. If overloaded/rate-limited in `-p` (print) mode, set `--fallback-model sonnet` to avoid hard failure.
6. Do not fabricate slugs. If a requested model/alias is not in the tables above, verify with `claude --print --model <slug> "reply with exactly: ok"` or official docs before using it in a command.
