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

## Rich view: on, and verified on this client

`plugins.entries.telegram_dashboard_probe.settings.rich = true` (2026-11-02). Telegram
accepted it (`screen_format: rich`, `rich_error: null`) and the owner confirmed the tables
and folds render correctly in their client. The **engine-wide** switch
`platforms.telegram.rich_messages` stays **off** — it would change every bot reply, and the
plugin calls `adapter._bot.do_api_request` itself, so only the dashboard message is rich.

What the rich view adds over HTML: a real `Limits used` table, the `🕒 Cron` fold with the
per-job table (name, frequency, next run), sectioned Details. If a rich edit is ever refused
the plugin falls back to HTML in the same tick and retries after `RICH_RETRY_TICKS`; the
record then says `screen_format: html` with the reason in `rich_error`.

Rollback to the classic view:

```bash
hermes config set plugins.entries.telegram_dashboard_probe.settings.rich false
hermes plugins disable telegram_dashboard_probe && hermes plugins enable telegram_dashboard_probe --no-allow-tool-override
```

## Fifth patch: the cron job visible in the HTML view

1.0.0 draws the cron job table only in the rich form, so with `rich: false` (the previous
state) a job was invisible: the ✓ row said `✓ Cron` and Details said only
`Cron 1 active · 0 paused`. `render.py` now carries the count in the compact row
(`✓ Cron 1`, via a `row` label on `_Item`) and lists names with the engine's schedule words
in Details (`Cron jobs: memory-review (every day at 9am)`). `verify.py` asserts the
`Cron jobs:` line whenever `cron/jobs.json` holds a job, so a future upstream refactor of
the row or the details cannot silently drop it again.

## Rich replies: on for all three profiles

`platforms.telegram.rich_messages = true` in `default`, `docs`, `plc` (2026-11-02), verified by
the owner: a pipe table in a reply renders as a native table. Mechanics that matter:

- The flag is read when the adapter is built, so a config change alone does nothing.
- The multiplexer's config watcher rebuilds only NON-launch profiles
  (`changed = [... and n != active ...]`), and `unserve-profile`/`serve-profile` refuse the
  launch profile ("a non-launch profile name is required"). Therefore `default` needs a real
  `hermes gateway restart`, while `docs`/`plc` can be re-served with `hermes -p <n> gateway restart`.
- Ordinary replies stay MarkdownV2; only content with a pipe table, task list, `<details>` or
  block math takes the rich path. Streaming preview stays raw (`rich_drafts` off).
- A profile's agent will otherwise repeat the stale belief "Telegram does not render tables";
  the rule written into `profiles/docs/memories/USER.md` corrects it (memory is injected at
  session start, so it applies after `/new`).

## Sixth patch: never deliver with another profile's bot

`live_adapter()` used to resolve `runner.adapters["telegram"]` — the LAUNCH profile's adapter —
when the plugin was not wired yet. On the first tick after a gateway restart a secondary
profile's dashboard was therefore created in the default chat (plc wrote message 109 there).
Now: no adapter reference → wait; the launch profile's dict is consulted only when the plugin's
own home is the launch home; otherwise only `_profile_adapters` are considered. Verified by
`adapter_generation: 1` in all three records and a recreate in the right chat.

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

