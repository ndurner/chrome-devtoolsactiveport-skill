---
name: chrome-devtoolsactiveport
description: Attach to an existing local Chrome or Chromium session over CDP by reading DevToolsActivePort from the selected profile or user-data directory and constructing the browser websocket URL as ws://127.0.0.1:PORT/PATH. Use when Codex needs to inspect or drive websites in a real Chrome window, reuse an already-open browser session, avoid CDP HTTP discovery endpoints such as /json/version, or avoid repeated Chrome confirmation dialogs by using the included local broker instead of direct browser connections.
---

# Chrome DevToolsActivePort

## Overview

Attach to a running Chrome-family browser by reading `DevToolsActivePort` directly and deriving the browser websocket endpoint from its two-line payload. Prefer this skill when the session already exists and the attach flow must not rely on `/json/version` or other discovery endpoints.

## Workflow

1. Identify the browser profile or user-data directory that contains `DevToolsActivePort`. On Windows, do not start with a broad recursive search across `%LOCALAPPDATA%`, `%APPDATA%`, or `%USERPROFILE%`.
2. Prefer `python scripts/find_devtools_active_port.py --json` to search only the standard Chrome, Chromium, and Edge profile roots and sort candidates by recency.
3. Reuse an explicitly provided profile path immediately instead of searching.
4. If Chrome shows a confirmation dialog on new debugging connections, check for an existing broker status file and use its `local_ws_url` first.
5. If no broker is running and repeated prompts are possible, start `python scripts/cdp_connection_broker.py "<profile-dir>" --status-file "$env:TEMP\chrome-cdp-broker.json"` and use the broker's `local_ws_url`.
6. Only skip the broker when a single long-lived direct websocket will definitely be reused for the entire task and no repeated reconnects are expected.
7. Run `python scripts/resolve_devtools_active_port.py "<profile-dir>" --json` only when a direct Chrome websocket is explicitly required.
8. On Windows, set `PYTHONIOENCODING=utf-8` before inline Python or script runs that may print page text, titles, DOM content, or JSON with non-ASCII characters.
9. Enumerate browser targets only after the websocket connection is established.
10. Select or create the page target needed for the task, then continue with normal CDP commands.

## Rules

- Read `DevToolsActivePort` from disk instead of calling `/json/version`, `/json/list`, or other HTTP discovery endpoints to find the browser websocket URL.
- Construct the browser websocket URL exactly as `ws://127.0.0.1:<port><path>`.
- Treat line 1 of `DevToolsActivePort` as the port and line 2 as the websocket path.
- Normalize the path to begin with `/` before concatenating it into the final URL.
- Fail fast when the file is missing, malformed, or stale. Usually that means the wrong profile was chosen or Chrome was not started with remote debugging enabled.
- Keep the browser attachment step separate from target enumeration. Resolve the browser socket first, then inspect tabs or pages through CDP.
- On Windows, avoid broad recursive PowerShell searches across entire home or app-data trees. Search only standard Chrome-family roots or use `scripts/find_devtools_active_port.py`.
- When Chrome presents a confirmation dialog on each new connection, use the broker by default. Do not connect directly to Chrome unless there is a concrete reason not to use the broker.
- Before any browser task in that environment, check whether a broker is already running and reuse it instead of creating a new direct connection.
- If repeated attach/detach cycles are unavoidable, keep the broker in front of Chrome so Codex reconnects to the broker while the broker keeps one approved upstream CDP session open.
- Treat a fresh direct websocket to Chrome as an exception path that requires justification in the response or work log.
- On Windows, assume the console is not UTF-8 unless already configured. Before printing non-ASCII text from Python, set `PYTHONIOENCODING=utf-8` or configure `sys.stdout` and `sys.stderr` to UTF-8 with safe error handling.
- Prefer JSON output with default ASCII escaping when printing structured data from Python in this workflow.

## Quick Start

Find the most likely active profile:

```powershell
python scripts/find_devtools_active_port.py --json
```

Direct endpoint resolution:

```powershell
python scripts/resolve_devtools_active_port.py "C:\path\to\chrome-profile" --json
```

Example output:

```json
{
  "input": "C:\\path\\to\\chrome-profile",
  "devtools_active_port": "C:\\path\\to\\chrome-profile\\DevToolsActivePort",
  "port": 53124,
  "path": "/devtools/browser/01234567-89ab-cdef-0123-456789abcdef",
  "ws_url": "ws://127.0.0.1:53124/devtools/browser/01234567-89ab-cdef-0123-456789abcdef"
}
```

Connect the client to `ws_url`, then query `Target.getTargets`, `Target.attachToTarget`, or equivalent page-selection APIs through the websocket session.

## Connection Reuse

Chrome commonly treats each fresh websocket connection as a new debugging client. If the browser shows a confirmation dialog per connection, the practical fix is to avoid reconnecting.

- Best default when prompts exist: run a persistent local broker process that holds the approved upstream CDP connection to Chrome and exposes a stable local websocket for Codex.
- Acceptable direct path: keep one approved websocket open and reuse it for the whole Codex session.
- Unacceptable default: reconnect directly each time and accept repeated confirmations.

If Codex runs in short-lived processes that cannot hold a persistent socket, direct reuse is not sufficient by itself. In that case, a broker is the correct architecture.

## Broker Usage

Start the broker once:

```powershell
$env:PYTHONIOENCODING = "utf-8"
python scripts/cdp_connection_broker.py "C:\path\to\chrome-profile" --status-file "$env:TEMP\chrome-cdp-broker.json"
```

The broker will:

- read `DevToolsActivePort`,
- open one upstream websocket to Chrome,
- keep that approved connection alive,
- expose a local websocket such as `ws://127.0.0.1:51327`,
- update the JSON status file with both upstream and local connection details.

Point Codex or any downstream CDP client at the broker's `local_ws_url` from the status file instead of connecting to Chrome directly.

Broker-first decision rule:

- If the environment is known to show Chrome approval prompts on new CDP connections, assume broker-first even for apparently small tasks.
- If a broker status file already exists, reuse it before doing anything else.
- If no broker exists, start one before the first browser command.
- Do not fall back to direct Chrome connections merely because the task appears one-off.

## Windows Console Safety

When the task may print titles, page text, DOM content, JSON values, or any other non-ASCII data from Python on Windows, set UTF-8 first:

```powershell
$env:PYTHONIOENCODING = "utf-8"
```

If the script is inline Python or otherwise under active editing, also prefer:

```python
import sys
sys.stdout.reconfigure(encoding="utf-8", errors="backslashreplace")
sys.stderr.reconfigure(encoding="utf-8", errors="backslashreplace")
```

Use `errors="backslashreplace"` when robustness matters more than perfect display fidelity. That avoids `UnicodeEncodeError` crashes in `cp1252` terminals.

## Troubleshooting

- If `DevToolsActivePort` is absent, verify that Chrome was launched with remote debugging enabled and that the selected directory is the active browser data directory.
- If the file exists but the websocket refuses the connection, re-read the file. The browser may have restarted and the port may have changed.
- If the browser data directory is unknown, search the expected Chrome user-data tree for `DevToolsActivePort` and prefer the most recently updated file.
- If profile discovery is slow or times out, switch to `scripts/find_devtools_active_port.py` instead of widening the recursive search.
- If the browser keeps prompting for approval, verify that downstream clients are using the broker's `local_ws_url` rather than opening fresh direct connections to Chrome.
- If Python crashes with `UnicodeEncodeError` from `cp1252`, rerun with `PYTHONIOENCODING=utf-8` or reconfigure `sys.stdout` and `sys.stderr` before printing.
- If a task needs platform-specific launch or path details, read [references/connection-notes.md](references/connection-notes.md).

## Resources

- Use `scripts/resolve_devtools_active_port.py` for deterministic parsing and URL construction.
- Use `scripts/find_devtools_active_port.py` to search only standard Chrome-family roots and avoid slow ad hoc recursive filesystem scans.
- Use `scripts/cdp_connection_broker.py` when repeated Chrome confirmation prompts make direct CDP connections impractical.
- Read [references/connection-notes.md](references/connection-notes.md) only when launch flags, file format details, or troubleshooting details are needed.
