# Chrome DevToolsActivePort Skill

Codex skill for attaching to an existing local Chrome-family browser session through `DevToolsActivePort`, with a broker-first workflow for environments where Chrome shows a confirmation dialog for each new debugging connection.

## What This Skill Does

- attaches to an already running Chrome instance
  - untested: Chromium, or Edge
- reads `DevToolsActivePort` directly instead of relying on CDP HTTP discovery endpoints
- constructs the browser websocket as `ws://127.0.0.1:PORT/PATH`
- prefers a persistent local broker so repeated Codex runs do not keep triggering Chrome confirmation dialogs
- avoids slow ad hoc profile discovery by searching standard browser profile roots only
- adds Windows console safety guidance for `cp1252` / non-UTF-8 environments

## Why This Exists

Direct CDP connections are workable, but they break down in two common cases:

1. Chrome prompts for approval on every new debugging connection.
2. Agent workflows improvise broad filesystem searches or Python output that is fragile on Windows.

This skill makes those cases explicit and gives Codex deterministic scripts for the critical steps.

## Repository Layout

The repository root is the skill folder.

- [SKILL.md](SKILL.md): the actual skill instructions
- [agents/openai.yaml](agents/openai.yaml): UI metadata
- [scripts/find_devtools_active_port.py](scripts/find_devtools_active_port.py): targeted profile discovery
- [scripts/resolve_devtools_active_port.py](scripts/resolve_devtools_active_port.py): deterministic websocket resolution
- [scripts/cdp_connection_broker.py](scripts/cdp_connection_broker.py): persistent local broker
- [references/connection-notes.md](references/connection-notes.md): supporting notes and failure modes

## Install

### Codex
Install this Skill by beginning the prompt with "/install", select the Skill Installer from the popup, and continue the prompt with:
> Install the skill in ...
(... = path to this skill folder).

Regarding model choice to run the installer, GPT-4.5 Mini is sufficient.

After the installer is done, restart Codex.

## Typical Usage
Enable Remote Debugging in Chrome by entering the following in the address bar:
> chrome://inspect/#remote-debugging

Check the checkbox there.

In Codex, begin the prompt with `$chrome`, select the "Chrome via DevToolActivePort" entry, and continue like so:
> Open ndurner.de and retrieve the three latest blog posts.

Be sure to wait for and confirm the security popup in Chrome.

## Design Notes

- `broker-first` is the default when Chrome approval prompts are present
- direct browser websocket connections are treated as an exception path
- broad recursive searches over `%LOCALAPPDATA%`, `%APPDATA%`, or `%USERPROFILE%` are intentionally discouraged
- UTF-8 console configuration is part of the workflow on Windows
