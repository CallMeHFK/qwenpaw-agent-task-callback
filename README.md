# Agent Task Callback (QwenPaw plugin)

Opt-in persistent watcher for **inter-agent background tasks**. QwenPaw's
`submit_to_agent` is pull-only: it returns a `task_id` and the result stays in
memory until something calls `check_agent_task`. When a sub-agent finishes,
nothing notifies the caller — so a console session that submitted work and
moved on never learns it completed.

This plugin adds the missing **push** side: a dedicated watcher thread polls the
child task, and on completion delivers the result back into the *registering*
session as a fresh turn.

## Install

QwenPaw keeps **two copies** of a plugin and they are *not* symlinked:

| Path | Role |
| --- | --- |
| `~/.qwenpaw/plugins/<id>/` | **Loaded at runtime** — this is the code that actually runs |
| `~/.qwenpaw/workspaces/<agent>/plugin-dev/<id>/` | Development source tree |

Editing only the dev tree does **nothing** until the plugin is reinstalled.
Keep both in sync, or copy the source over the installed copy and reload:

```bash
ID=agent-task-callback
SRC=~/.qwenpaw/workspaces/default/plugin-dev/$ID
DST=~/.qwenpaw/plugins/$ID

cp "$SRC/plugin.json" "$DST/plugin.json"
mkdir -p "$DST/backend" && cp "$SRC/backend/main.py" "$DST/backend/main.py"
rm -rf "$DST/backend/__pycache__"

# then reload from the console, or:
curl -sS -X POST http://127.0.0.1:19999/api/plugins/install \
  -H 'Content-Type: application/json' \
  -d "{\"source\":\"$SRC\",\"force\":true}"
```

Always verify both copies agree afterwards:

```bash
diff -q "$SRC/backend/main.py" "$DST/backend/main.py" && echo IN SYNC
```

### Which agents see the tools

QwenPaw's plugin router syncs `meta.tools` from `plugin.json` into **every**
agent's `builtin_tools` config on install/reload — that part is automatic, so
the manifest must list the tools:

```json
"meta": { "tools": [ { "name": "watch_agent_task" }, ... ] }
```

The catch: the sync writes entries with **`enabled: false`**. Until an agent's
config flips that to `true`, the agent will not be offered the tool. Enabling
is per-agent and is **not** covered by the plugin:

```python
from qwenpaw.config.config import load_agent_config, save_agent_config

cfg = load_agent_config(agent_id)          # via qwenpaw.config.utils.load_config
for n in ("watch_agent_task", "callback_task_status", "cancel_task_callback"):
    if n in cfg.tools.builtin_tools:
        cfg.tools.builtin_tools[n].enabled = True
save_agent_config(agent_id, cfg)
```

Note this means **newly created agents start disabled** and need the same
one-time flip. `manifest.version` is reported at load time but is *not* used by
the framework for upgrade decisions — bump it anyway so consumers can tell
revisions apart.

## Tools

| Tool | Purpose |
| --- | --- |
| `watch_agent_task` | Watch a submitted task and resume this session on completion |
| `callback_task_status` | Inspect this session's callback jobs |
| `cancel_task_callback` | Cancel a pending callback (not the child task) |

## API base URL resolution

Priority: **framework resolver > environment > default port**.

1. `qwenpaw.agents.tools.agent_management._normalize_api_base_url(None)`
2. Environment — `QWENPAW_RUNTIME_API_URL`, or `QWENPAW_RUNTIME_HOST` +
   `QWENPAW_RUNTIME_PORT`
3. Default — `http://127.0.0.1:19999/api`

```bash
export QWENPAW_RUNTIME_PORT=23456   # only used if the resolver is unavailable
```

Every branch is normalized to keep the required `/api` suffix.

## Delivery semantics

- **409 means "a turn is already running for this chat"** — the plugin retries
  with backoff (30 x 20s) rather than blindly re-delivering, so no duplicate
  turns are produced.
- Jobs are persisted to `~/.qwenpaw/agent-task-callback.json`, so a watcher
  survives a process restart: `on_start` re-arms every `pending`/`unconfirmed`
  job.

## Notes / limitations

- Requires QwenPaw 2.2.0–2.3.0.
- The restart path has not been exercised against a live `systemctl restart`
  in every environment; the recovery logic itself is verified by directly
  invoking `on_start`.
- `target_agent` is not accepted by `watch_agent_task`; identity is carried via
  the request path and headers instead.

## Development

`backend/main.py` is a single self-contained module. It compiles under
Python 3.12 and its URL-resolution helpers are covered by isolated unit tests.

When changing code, do all three or the change will not take effect:

1. edit the dev source tree,
2. copy it over `~/.qwenpaw/plugins/agent-task-callback/`,
3. bump `version` in `plugin.json` and reload.

## Changelog

- **0.1.1** — bump version to match code; document the two-copy install layout
  and the `enabled: false` tool-sync default.
- **0.1.0** — initial release: watcher thread, three tools, persisted jobs,
  409-aware backoff delivery, API base URL resolution
  (framework resolver > environment > port 19999).
