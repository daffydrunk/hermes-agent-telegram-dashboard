# Local operator patch — telegram_dashboard_probe 0.11.0 (2026-11-02)

Why: this installation has one paid provider (`opencode-go`). Claude, Codex, Grok,
Kimi and Gemini have no account here, so their lines were permanent `no data`
noise; opencode-go had no line at all.

## Changes

`telegram_dashboard/limits.py`
- `PROVIDERS` reduced to `(("opencode-go", "opencode-go"),)` — the usage facade is
  asked only for this provider (`agent.account_usage.fetch_account_usage` answers
  with a `go_usage_api` source: rolling window / weekly / monthly).

`telegram_dashboard/collect.py`
- `_PROVIDER_LABELS = {"opencode-go": "OpenCode Go"}` (was claude/codex).
- `_HIDDEN_PROVIDERS`, `_HIDDEN_SOURCES`, `_HIDDEN_INCIDENT_PREFIXES` +
  `only_visible()`: their quota lines, sources and incidents are dropped after
  collection. The collectors still run (they cost nothing without a credential).
- Both `build_snapshot(...)` call sites wrap `capacity=` in `only_visible(...)`;
  `build_snapshot` filters `sources` and `incidents`.

## Verified

`python -m telegram_dashboard --config <cfg> --dry-run` (engine venv
`installs/38f8a3518868e803/environments/f703f1f0.../venv/Scripts/python.exe`,
`PYTHONPATH=<plugin dir>;<hermes-agent>`) prints a limits block of exactly one line:

    OpenCode Go 1% (4h27m) · 0% (6d19h) · 74% (7d9h)

## After a gateway restart

The panel returns within one `period_seconds` (settings live in
`~/.hermes/config.yaml` → `plugins.entries.telegram_dashboard_probe.settings`).

## Revert / update

- Revert: restore `telegram_dashboard/limits.py.bak-20261102` and
  `telegram_dashboard/collect.py.bak-20261102`.
- `hermes plugins update telegram_dashboard_probe` reinstalls the catalog pin and
  WIPES these edits — re-apply this file's diff afterwards.
- Upstream-worthy: a `limits_providers` setting would replace this patch.
  Repo: https://github.com/bablobanov/hermes-agent-telegram-dashboard

## Third patch: restart-free re-wiring (`__init__.py`)

Needed because the live gateway was not restarted (operator had bot access only).
Adapters dedupe plugin factories by `(plugin, factory.__qualname__)`
(`gateway/platforms/base.py::_wire_plugin_handlers`), so re-registering the bound
method `runtime.wire` after a reload is skipped and the tick loop stays dead.

`register()` now registers a local wrapper `wire_after_reload(native, adapter)`
with its own qualname, which is a fresh key. Reload trigger used:

    hermes plugins disable telegram_dashboard_probe
    hermes plugins enable telegram_dashboard_probe --no-allow-tool-override

Gateway log then shows `Wired native handlers from plugin 'telegram_dashboard_probe'`
and the record (`ticks`) climbs again. Equivalent programmatic call from the engine
venv: `gateway.control_socket.query_gateway_control(Path(home), 'reload-plugins')`.
A full `hermes gateway restart` is not required.

## Fourth patch: Windows liveness + quiet unobserved lines

Two things the screenshot showed that are not this installation's state but the
plugin's gaps on Windows:

`telegram_dashboard/collect.py`
- `default_pid_probe()` answered `None` on Windows by design ("cannot be checked on
  this platform"), which forced `Gateway unknown · Telegram unknown` and made the
  whole `gateway_state` source `unavailable`. Added `_windows_pid_alive()`:
  `OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION)` + `GetExitCodeProcess`, with
  `ERROR_INVALID_PARAMETER` (87) -> dead, `ERROR_ACCESS_DENIED` (5) -> alive.
- `_QUIET_SOURCES = {"drift", "backup", "cron", "cron_runs"}`: a source whose state
  is `unsupported` (no drift script, no backup status file, no cron jobs file) is
  dropped from `sources`, and the matching `backup`/`drift`/`cron` summaries are
  set to `None` in `build_snapshot`, so the header no longer carries
  `Backup: not observed` / `Drift: …` / `Cron: …` lines or names them under
  `Not observed`. When such a source IS configured it stays visible as before.

Result of the dry-run after the patch: `Gateway ✓ · Telegram ✓`, one limits line,
no not-observed block.

## The dynamic qualname

`wire_after_reload.__qualname__` is stamped with the load time, so every reload is
a distinct dedupe key — a static wrapper would be deduped on the second reload.

