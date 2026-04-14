# Chrome DevToolsActivePort Notes

## File Format

`DevToolsActivePort` is a small text file written by Chrome-family browsers when remote debugging is enabled. Treat the first two lines as:

1. TCP port
2. Browser websocket path

Build the browser websocket URL as:

```text
ws://127.0.0.1:<port><path>
```

Do not depend on `/json/version` or similar HTTP discovery endpoints for the initial browser websocket URL when this skill is in use.

## Typical Flow

1. Locate the active browser data directory or direct path to `DevToolsActivePort`.
2. On Windows, prefer the skill's targeted finder script over broad recursive PowerShell searches.
3. If the environment is known to show Chrome approval prompts on new connections, look for an existing broker and reuse it first.
4. If no broker is running, start the broker and use its local websocket.
5. Read `DevToolsActivePort` and connect directly only when the direct path is explicitly justified.
6. On Windows, set UTF-8 output before running Python that may print non-ASCII browser data.
7. Enumerate or attach to page targets through the chosen websocket session.

## Finder Script

This skill includes `scripts/find_devtools_active_port.py` to avoid broad recursive searches.

Example:

```powershell
python scripts/find_devtools_active_port.py --json
```

Behavior:

- searches only standard Chrome, Chromium, and Edge user-data roots
- returns `DevToolsActivePort` candidates sorted by newest first
- avoids recursive scans of all of `%USERPROFILE%`, `%APPDATA%`, or `%LOCALAPPDATA%`

## Reuse Strategy

Chrome may show a confirmation prompt for each new debugging connection. The prompt is tied to opening a new websocket client, not to each CDP command.

Prefer this order:

1. Reuse an existing local broker if one is already running.
2. If no broker exists, introduce a persistent local broker that:
   - opens one upstream websocket to Chrome,
   - keeps that approved connection alive,
   - multiplexes commands or exposes a local control surface to downstream clients.
3. Only use a direct Chrome websocket when one connection can stay open for the whole task and that exception is intentional.

## Broker Script

This skill includes `scripts/cdp_connection_broker.py` for the reuse case.

Example:

```powershell
python scripts/cdp_connection_broker.py "C:\path\to\chrome-profile" --status-file "$env:TEMP\chrome-cdp-broker.json"
```

The broker prints a JSON status object on startup and optionally writes the same information to `--status-file`, including:

- `upstream_ws_url`: the real Chrome browser websocket
- `local_ws_url`: the local websocket for downstream clients
- `active_downstream`: whether a downstream client is currently attached

Current design limits:

- one active downstream websocket client at a time
- browser-level proxying only; it forwards raw CDP frames
- no event replay for clients that disconnect and reconnect later

That design is intentional for now. It is enough to avoid repeated Chrome approval prompts without adding session-rewriting logic.

## Common Failure Modes

- Missing file: wrong profile path, wrong user-data directory, or remote debugging not enabled.
- Slow or timed-out profile discovery: the caller used a broad recursive filesystem scan instead of the targeted finder script or standard browser roots.
- Refused websocket connection: stale file after a browser restart, or the browser exited.
- Repeated permission prompts: the client is creating fresh websocket connections instead of reusing one existing approved session.
- Broker was available but not used: workflow ambiguity or the caller skipped the broker-first step. Fix the workflow; do not normalize this behavior.
- Unexpected path shape: normalize a missing leading slash before constructing `ws://127.0.0.1:<port><path>`.
- `UnicodeEncodeError` from `cp1252`: Python printed non-ASCII data to a non-UTF-8 Windows console. Set `PYTHONIOENCODING=utf-8` or reconfigure `sys.stdout` and `sys.stderr` before printing.

## Platform Notes

- On Windows, user-data directories are commonly rooted under `%LOCALAPPDATA%` for Chrome-family browsers.
- On macOS, user-data directories are commonly rooted under `~/Library/Application Support/`.
- On Linux, user-data directories are commonly rooted under `~/.config/`.

Prefer the exact browser-specific path already present in the task context over generic defaults.
