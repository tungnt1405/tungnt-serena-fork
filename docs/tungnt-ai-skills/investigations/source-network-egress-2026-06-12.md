# Investigation: Source Network Egress

## Hand-off Brief

1. **What happened.** Static review found startup and dashboard code that calls external URLs, plus conditional download/API paths.
2. **Where the case stands.** Complete; no runtime commands were executed.
3. **What's needed next.** Disable or gate the confirmed startup/dashboard egress if the fork should run offline by default.

## Case Info

| Field | Value |
| --- | --- |
| Ticket | N/A |
| Date opened | 2026-06-12 |
| Status | Complete |
| Evidence sources | Static source search and targeted reads in `src/`, `.serena/project.yml`, `pyproject.toml` |

## Problem Statement

User asked whether this source contains code that runs in the background and calls external APIs.

## Confirmed Findings

### Finding 1: Agent startup sends usage info externally

**Evidence:** `src/serena/agent.py:715`, `src/serena/agent.py:717`, `src/serena/agent.py:728`

**Detail:** `SerenaAgent.__init__` calls `_send_usage_info()` at the end of initialization. Unless `CI=true`, `GITHUB_ACTIONS=true`, or `SERENA_USAGE_REPORTING=false`, it sends a GET request to `https://oraios-software.de/serena_usage.php` with OS, dashboard flag, version, backend, and context.

### Finding 2: Dashboard startup launches a background thread that fetches remote news

**Evidence:** `src/serena/agent.py:680`, `src/serena/agent.py:701`, `src/serena/dashboard.py:220`, `src/serena/dashboard.py:695`, `src/serena/dashboard.py:714`

**Detail:** When `web_dashboard` is enabled, agent startup creates `SerenaDashboardAPI`. Its constructor starts a daemon thread for `_fetch_news`, which GETs `https://oraios-software.de/serena_news.json`.

### Finding 3: Dashboard frontend fetches banner manifest from an external host

**Evidence:** `src/serena/resources/dashboard/dashboard.js:94`, `src/serena/resources/dashboard/dashboard.js:96`

**Detail:** When the dashboard page loads in a browser/webview, JavaScript calls `https://oraios-software.de/serena-banners/manifest.php`.

### Finding 4: Dashboard frontend loads fonts and chart scripts from external CDNs

**Evidence:** `src/serena/resources/dashboard/index.html:11`, `src/serena/resources/dashboard/index.html:13`, `src/serena/resources/dashboard/index.html:16`, `src/serena/resources/dashboard/index.html:17`

**Detail:** Opening the dashboard page preconnects to Google Fonts, loads a Google Fonts stylesheet, and loads Chart.js plus chartjs-plugin-datalabels from jsDelivr.

### Finding 5: Default config enables the dashboard

**Evidence:** `src/serena/resources/serena_config.template.yml:38`, `src/serena/resources/serena_config.template.yml:49`, `src/serena/config/serena_config.py:719`, `src/serena/config/serena_config.py:720`

**Detail:** The template and dataclass defaults set `web_dashboard: True` and `web_dashboard_open_on_launch: True`, so the dashboard path is active unless overridden.

## Conditional Findings

### Finding 6: Language server support may download external binaries/resources

**Evidence:** `src/solidlsp/ls_utils.py:232`, `src/solidlsp/ls_utils.py:250`, `src/solidlsp/ls_utils.py:282`, `src/solidlsp/ls_config.py:75`, `src/solidlsp/ls_config.py:103`, `src/solidlsp/ls_config.py:119`, `src/solidlsp/ls_config.py:124`, `src/solidlsp/ls_config.py:176`

**Detail:** Shared download helpers use `requests.get(..., stream=True)`. Several language server configurations document automatic downloads when their language is selected and resources are missing.

### Finding 7: Current project languages can trigger package-manager egress on first use

**Evidence:** `.serena/project.yml:31`, `.serena/project.yml:32`, `src/solidlsp/language_servers/pyright_server.py:47`, `src/solidlsp/language_servers/pyright_server.py:48`, `src/solidlsp/ls.py:337`, `src/solidlsp/ls.py:376`, `src/solidlsp/language_servers/typescript_language_server.py:203`, `src/solidlsp/language_servers/typescript_language_server.py:206`

**Detail:** This repo's Serena project selects `python` and `typescript`. Python uses a Pyright dependency provider that builds an `uvx` / `uv x` launch command for `pyright`; uv may fetch the package/interpreter if missing. TypeScript runs managed `npm install` for `typescript` and `typescript-language-server` when the managed executable is missing.

### Finding 8: Token usage estimator can call Anthropic if configured

**Evidence:** `src/serena/analytics.py:46`, `src/serena/analytics.py:62`, `src/serena/resources/serena_config.template.yml:160`, `src/serena/resources/serena_config.template.yml:164`

**Detail:** Default `token_count_estimator` is `CHAR_COUNT`, which is local. If set to `ANTHROPIC_CLAUDE_SONNET_4`, token counting sends text to Anthropic's API.

### Finding 9: `scripts/agno_agent.py` uses external LLM providers only when explicitly run

**Evidence:** `scripts/agno_agent.py:1`, `scripts/agno_agent.py:17`

**Detail:** The script imports Gemini and Claude model integrations and selects Gemini. This is not a packaged default entrypoint.

## Local-only Background Activity

| Activity | Evidence | Scope |
| --- | --- | --- |
| Dashboard Flask server | `src/serena/dashboard.py:796`, `src/serena/dashboard.py:799` | Local listener, default `127.0.0.1` |
| Dashboard browser/viewer process | `src/serena/agent.py:481`, `src/serena/dashboard.py:816` | Opens local dashboard URL |
| Project ignore-spec gathering | `src/serena/project.py:70`, `src/serena/project.py:74` | Local filesystem |
| Language server subprocesses | `src/solidlsp/ls_process.py:461`, `src/solidlsp/ls_process.py:465`, `src/solidlsp/ls_process.py:485` | Local child processes; their own language servers may have separate behavior |
| JetBrains plugin client | `src/serena/jetbrains/jetbrains_plugin_client.py:193`, `src/serena/jetbrains/jetbrains_plugin_client.py:217` | Local `127.0.0.1` HTTP service |

## Conclusion

**Confidence:** High

Confirmed: yes, the source contains code that can run in the background and call external endpoints. The two most important default-path findings are startup usage reporting and dashboard remote-news fetching. Dashboard banner, font, and CDN fetching happens when the dashboard frontend is opened. Language-server downloads/package installs and Anthropic token counting are conditional on language/config/cache state.

## Recommended Next Steps

1. Set `SERENA_USAGE_REPORTING=false` for runtime mitigation.
2. Disable `web_dashboard` or add config gates for remote news/banner fetching if offline-by-default behavior is required.
3. Review selected project languages before first run because some language servers auto-download resources when missing.
