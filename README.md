# mate-adapters

Public registry of agent CLI adapters for the **Mate** app. Mate fetches
`registry.json` from this repo (on launch and when you tap **Refresh models**).

## What the app does with it

- **Built-in adapters (`claude`, `codex`, `copilot`)** — the app applies **only the
  `models` array** from here. Everything else (how the CLI is spawned, the driver,
  output parsing) stays bundled in the app and **cannot** be changed from this repo.
  This means you can ship new model names/updates without an app release, while a
  compromised registry can never change how a built-in is launched.
- **Third-party adapters** (any other `id`) — the full manifest is used.

## Updating models

Edit the adapter's `models` array in `registry.json`:

```json
{ "id": "claude-x", "label": "Claude X", "contextWindow": 200000, "maxOutput": 64000 }
```

`id` and `label` are required; `contextWindow`/`maxOutput` are optional. Commit to
`main`. The app picks it up on its next refresh (cached up to 24h; **Refresh models**
forces it). Note: **Codex** discovers its models live from the CLI, so its list here
is only a fallback.
