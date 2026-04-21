# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

TrendRadar is a Chinese-language hotspot news aggregator and analyzer. It crawls hot-news boards (Toutiao, Baidu, Weibo, Zhihu, etc.) and RSS feeds on a schedule, filters items by keyword or AI tagging, optionally runs LLM analysis over the results, and pushes a formatted report to chat channels. It ships as two entry points: the main crawler (`trendradar`) and a FastMCP 2.0 server (`trendradar-mcp`) that exposes the same data to MCP clients.

Runtime is Python **3.12+** managed with **uv** (see [pyproject.toml](pyproject.toml) / [uv.lock](uv.lock)). Main package is `trendradar/`, MCP package is `mcp_server/`. Both are installed as wheels by hatchling.

## Commands

```bash
# First-time setup (installs into .venv/)
uv sync

# Run the main crawler (crawl → analyze → push one cycle)
uv run python -m trendradar              # or just: trendradar

# Diagnostic / status subcommands (see trendradar/__main__.py:2141)
uv run python -m trendradar --doctor              # env + config health check, writes output/meta/doctor_report.json
uv run python -m trendradar --show-schedule       # resolve current timeline period + behaviour flags
uv run python -m trendradar --test-notification   # send a test message to every configured channel

# Run the MCP server (stdio by default; HTTP via FastMCP args)
uv run python -m mcp_server.server        # or: trendradar-mcp

# CI-style install (used by .github/workflows/crawler.yml)
uv sync --frozen --no-dev

# Docker container management (inside the container, via docker/manage.py)
python manage.py run | status | config | files | logs | start_webserver | stop_webserver
```

There is **no test suite, linter, or type-checker wired up** — don't claim tests pass; run `--doctor` to validate a configuration change end-to-end instead.

## High-level architecture

### Entry point and orchestration

`trendradar/__main__.py` holds `NewsAnalyzer` (the orchestrator) and `main()`. `NewsAnalyzer.run()` executes the single pipeline: `_crawl_data()` → `_crawl_rss_data()` → `_execute_mode_strategy()`, which branches on `REPORT_MODE` (`daily` | `current` | `incremental`) and internally handles AI analysis and multi-channel push. Each CLI invocation is one cycle — scheduling is **external** (GitHub Actions cron, Docker supercronic).

### AppContext is the dependency container

[trendradar/context.py](trendradar/context.py) defines `AppContext`, which wraps the loaded config dict and exposes every downstream operation (time, storage, frequency matching, report generation, notification rendering, AI filter pipeline, scheduler). **Nothing reads the global config directly** — code takes an `AppContext` and calls its methods. When adding a feature that depends on config, add a property/method on `AppContext` rather than reaching into `config` dicts from leaf modules.

Storage and Scheduler are lazy singletons on the context (`get_storage_manager()`, `create_scheduler()`). `AppContext.cleanup()` flushes retention cleanup and closes DB connections — always call it in a `finally`.

### Timeline scheduler (the "what runs when")

[trendradar/core/scheduler.py](trendradar/core/scheduler.py) resolves `config/timeline.yaml` against the current time into a `ResolvedSchedule` with three independent boolean gates: `collect` / `analyze` / `push`, plus a per-period `report_mode`, `ai_mode`, and `once_analyze` / `once_push` dedupe flags. `config.yaml`'s `schedule.preset` picks one of the named presets (e.g. `morning_evening`, `office_hours`) or `custom` (then reads `timeline.yaml`'s `custom` block). Period boundaries are **half-open intervals** (see commit `ddd3f4d`) — don't re-introduce closed-interval logic.

`notification.enabled` / `ai_analysis.enabled` are hard master switches that the scheduler cannot override. The scheduler only decides *when* within an enabled feature.

### Filter strategies are orthogonal

`filter.method` in config.yaml selects:

- **`keyword`** — classic regex/literal matching over `config/frequency_words.txt` (`[GLOBAL_FILTER]` + `[WORD_GROUPS]` sections, see [trendradar/core/frequency.py](trendradar/core/frequency.py)).
- **`ai`** — LLM-based tagging. `AppContext.run_ai_filter()` ([context.py:519](trendradar/context.py#L519)) reads `config/ai_interests.txt` (or a custom file in `config/custom/ai/`), hashes it, and extracts tags via LiteLLM. When the hash changes it asks the model for a `keep`/`add`/`remove` diff; if `change_ratio >= reclassify_threshold` it does a full re-classify, otherwise an incremental update. Results and per-news "already analyzed" markers live in the AI-filter tables defined by [trendradar/storage/ai_filter_schema.sql](trendradar/storage/ai_filter_schema.sql). `convert_ai_filter_to_report_data()` then reshapes AI results into the same `(hotlist_stats, rss_stats)` tuple the keyword pipeline produces, so downstream report/notification code is mode-agnostic.

### Storage backends are pluggable and auto-selected

[trendradar/storage/manager.py](trendradar/storage/manager.py) picks `local` (SQLite + optional TXT/HTML snapshots under `output/`) or `remote` (S3-compatible: R2/OSS/COS/S3/MinIO). `backend: "auto"` in config means: if running in GitHub Actions **and** `S3_*` env vars are set, use remote; otherwise local. Remote mode exposes `begin_batch()` / `end_batch()` — the AI filter wraps its multi-write pipeline in those to avoid N separate DB uploads. Data layout is flat: `output/news/{date}.db`, `output/rss/{date}.db`, `output/html/{date}/*.html`, `output/txt/{date}/*.txt`. `storage.pull` lets the MCP container sync recent days from remote on startup so it can answer queries locally.

### Notification fan-out

[trendradar/notification/](trendradar/notification/) — `renderer.py` produces per-platform markdown (feishu/dingtalk have their own flavours; `formatters.py` handles Slack's mrkdwn quirks). `splitter.py` chunks long content to per-channel byte limits (see `advanced.batch_size` in config). `dispatcher.NotificationDispatcher` handles multi-account fan-out (semicolon-separated webhook URLs, capped by `max_accounts_per_channel`) and AI translation. Supported channels: feishu, dingtalk, wework, telegram, email, ntfy, bark, slack, generic webhook.

### Config loading and overrides

[trendradar/core/loader.py](trendradar/core/loader.py) reads `config/config.yaml` (path overridable via `CONFIG_PATH`) and flattens it into the UPPER_SNAKE_CASE dict the rest of the code expects (`PLATFORMS`, `REPORT_MODE`, `AI`, `STORAGE`, …). Many env vars override specific fields (`TIMEZONE`, `DEBUG`, `SCHEDULE_ENABLED`, `SCHEDULE_PRESET`, `SORT_BY_POSITION_FIRST`, `MAX_NEWS_PER_KEYWORD`, `STORAGE_RETENTION_DAYS`, `AI_API_KEY`, `AI_MODEL`, `AI_API_BASE`, `S3_*`, all channel webhook/token vars — see `docker/docker-compose.yml` and `.github/workflows/crawler.yml` for the complete list). When adding a config field, decide up front whether it needs an env override (Docker and GHA users cannot edit the YAML at runtime in GHA mode).

### Versioning and update checks

Three independently-versioned artifacts, each tracked in its own file:

- [version](version) (main program, currently 6.6.1) — Docker tag is `v{version}`.
- [version_mcp](version_mcp) (MCP server, currently 4.0.3) — Docker tag is `mcp-v{version_mcp}`; [.github/workflows/docker.yml](.github/workflows/docker.yml) picks the image to build based on tag prefix.
- [version_configs](version_configs) — `filename=version` map; individual config files carry their own `Version:` header on line 1-20 and `check_all_versions()` in [__main__.py:95](trendradar/__main__.py#L95) compares them against this remote manifest on each run.

Bump the relevant file(s) when shipping a user-facing change — the config version check surfaces as a warning in the console and is how users know to re-pull the config.

### MCP server

`mcp_server/` is a FastMCP 2.0 app. Tools are grouped by concern under `mcp_server/tools/` (data queries, analytics, search, config management, storage sync, article reader, notification test). They read the same `output/` storage the crawler writes, so in Docker the two containers share `/app/output` via a volume. Resources (`@mcp.resource`) surface config metadata; tools (`@mcp.tool`) surface queries and mutations.

### Docker runtime specifics

[docker/Dockerfile](docker/Dockerfile) installs **supercronic** and runs it as **PID 1**. `entrypoint.sh` validates the `CRON_SCHEDULE` env var (regex-checked for legal cron characters only — don't bypass this), writes `/tmp/crontab`, optionally runs once (`IMMEDIATE_RUN=true`), starts a sandboxed `python -m http.server` over `output/`, and execs supercronic. `docker/manage.py` is the in-container admin tool (status, logs, webserver lifecycle). Because supercronic is PID 1, restarting a cron job means restarting the container.

### GitHub Actions crawler — 7-day trial gate

[.github/workflows/crawler.yml](.github/workflows/crawler.yml) self-disables after 7 days of continuous runs unless the user manually triggers a "Check In" workflow (which resets the 7-day timer). Don't remove this gate without coordinating with the upstream project — it's a fair-use mechanism for shared GHA resources and the newsnow upstream API. Long-term deployments are expected to run the Docker image.

### Issue Guard

[.github/workflows/issue-guard.yml](.github/workflows/issue-guard.yml) is a two-layer spam filter on issues / comments / PRs: accounts < 7 days old are auto-closed; older accounts get AI-moderated. Changes here affect contributor-facing moderation.

## Conventions worth knowing

- **Everything uses the configured timezone.** Never call `datetime.now()` directly in code that touches display/scheduling — use `AppContext.get_time()` / `ctx.format_date()` / `ctx.format_time()`. See [trendradar/utils/time.py](trendradar/utils/time.py).
- **Chinese comments + log messages are the norm** throughout `trendradar/` and `docker/manage.py`. New code in those packages should match; leave existing bilingual console output alone.
- **Config files begin with a `Version: X.Y.Z` header comment.** When editing a shipped config, bump the header **and** the matching line in [version_configs](version_configs) — the update-check UX depends on both.
- **The visual config editor lives at https://sansan0.github.io/TrendRadar/** (source: [docs/index.html](docs/index.html) and [index.html](index.html)). YAML keys it emits may be quoted; keep the loader tolerant of that (see commit `dd27ed8`).
- **Do not introduce backward-compat shims for removed config fields** — users regenerate config.yaml from the editor, and the update check nudges them to re-pull.
