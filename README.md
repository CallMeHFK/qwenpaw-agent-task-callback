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

```bash
cp -r . ~/.qwenpaw/workspaces/default/plugin-dev/agent-task-callback
# then reload the plugin from the QwenPaw console (or restart the service)
```

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
