# cli-bridge: supervisor environment is the effective env SSOT (pitfall + procedure)

A common pitfall when running this bridge under a process supervisor (launchd on macOS, systemd on Linux, etc.).

## Pitfall

A local `.env` file next to the bridge is **NOT authoritative** when the bridge runs under a supervisor.

- `bridge.py`'s `.env` loader is typically **no-override**: `if key not in os.environ: os.environ[key] = value`.
- The supervisor's own environment block (e.g. a launchd plist's `EnvironmentVariables`, or a systemd unit's `Environment=` lines) injects the same variables — these win, so `.env` edits silently do nothing while the service is managed by the supervisor.
- A simple "restart" (e.g. `launchctl kickstart -k`) restarts the process but does **NOT** re-read the supervisor's environment block. A supervisor-level env change requires a full reload of the service definition, not just a process bounce. On launchd, for example, this means unloading and reloading the job (bootout + bootstrap) rather than just kickstarting it. Never bare-kill the PID if the supervisor auto-restarts it (`KeepAlive`-style config) — you'll just get fought.

## Correct procedure for model-list changes

1. Backup: `bridge.py`, `.env`, the supervisor's service definition, and your agent framework's provider config (`cp X X.bak-$(date +%Y%m%d-%H%M%S)`).
2. Edit the supervisor's service definition programmatically (e.g. `plistlib` for launchd — never hand-edit the XML), with an assert-before and a read-back-after check.
3. Mirror the same model list into `.env` (keeps the manual-run path consistent with the supervised path).
4. Mirror into your agent framework's provider/model map for this bridge (parse-and-assert the exact key changed; if any model id contains special characters like `[` or `]`, make sure it's quoted correctly for the config format you're using).
5. Reload the supervisor's service definition fully (unload + reload, not just a bounce), then verify `curl -s http://127.0.0.1:<port>/v1/models`.
6. End-to-end smoke test at least one new model id via `/v1/chat/completions`.

## Synthetic "-max" effort ids (optional convention)

One convenient convention: register synthetic model ids ending in `-max` that map to a base model plus `--effort max`. Example `build_cli_cmd` logic:

```python
if model.endswith("-max"):
    cmd += ["--model", model[:-4], "--effort", "max"]
else:
    cmd += ["--model", model]
```

So `claude-opus-4-8-max` -> `claude --model claude-opus-4-8 --effort max`. This is safe because no real Anthropic slug ends in `-max`, so the suffix is unambiguous. Only Opus 4.6+ accepts `--effort max` — don't register `-max` variants for Sonnet/Haiku tiers, since they'll reject the flag.

## Example registered set

A representative model list for this bridge might look like: `claude-fable-5` (default), `claude-opus-4-8`, `claude-opus-4-8-max`, `claude-opus-4-7`, `claude-opus-4-7-max`, `opus[1m]` (1M context), `claude-sonnet-5`, `sonnet[1m]` (1M context), `opusplan`, `haiku`. Adjust to whichever slugs your Claude Max tier actually has access to — verify each one with the CLI before registering it.
