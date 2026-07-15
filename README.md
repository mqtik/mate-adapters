# mate-adapters

Community registry of adapter specs for the [Mate](https://github.com/mqtik/mate) app.

An adapter spec is a JSON file that tells Mate how to spawn a CLI agent, encode user messages into stdin bytes, parse the agent's stdout stream into UI events, and expose its tool, model, and slash-command catalogs. The spec is pure data — it is parsed by compiled-in Dart and applied identically on local and remote (iPhone/Android viewer) paths.

The Mate app fetches `registry.json` every 24 hours. When a new spec is available, it is hot-applied without requiring an app update.

---

## Spec format

Each adapter is a JSON file at `specs/<id>.v<schemaVersion>.json`. The full schema is defined in [`AGENT_UPDATE/01-adapter-schema.md`](https://github.com/mqtik/mate/blob/main/AGENT_UPDATE/01-adapter-schema.md) in the main repo.

Summary of top-level sections:

| Section | Required | Description |
|---------|----------|-------------|
| `schemaVersion` | yes | Schema structure version (integer, `[1,10]`). |
| `id` | yes | Stable adapter id, lowercase `[a-z0-9-]`. |
| `name` | yes | Human display name. |
| `version` | yes | Content version; bumped on any data change. |
| `binaryName` | yes | Executable to resolve on PATH. |
| `spawn` | yes | How to invoke the CLI: args, env, working dir. |
| `encoding` | yes | How a user prompt becomes stdin bytes. |
| `parsing` | yes | How stdout lines map to UI events and structured plan updates. |
| `tools` | no | Tool category sets, aliases, display labels. |
| `models` | no | Fallback model catalog. |
| `effortLevels` | no | Effort/reasoning levels for the `$` picker. |
| `slashCommands` | no | Adapter-specific `/` commands. |
| `capabilities` | no | Behavior flags (nativeResume, forkSession, …). |
| `timeouts` | no | Idle/inactivity/spawn timeout knobs. |
| `codePlugin` | no | Pointer to the compiled driver for stateful protocol behavior. |

### Model input capabilities

Models may declare their accepted input modalities. The app evaluates this at the
shared composer boundary after model selection, so local and remote sessions get
the same behavior without adapter-specific UI code.

```json
{
  "id": "example-text-only-model",
  "label": "Example text model",
  "inputModalities": ["text"],
  "unsupportedImageAttachmentMessage": "This model does not support image attachments. Switch models to send images."
}
```

`inputModalities` is optional for backward compatibility. An omitted value is
permissive until the provider publishes capability data. Supported values are
provider-defined modality ids such as `text`, `image`, `audio`, and `file`.

### Plan updates

`parsing.planUpdates` turns a provider's structured plan notification into the
shared task/plan presentation. The adapter supplies paths and status mappings;
the app owns the normalized UI behavior.

```json
"planUpdates": [{
  "match": "turn/plan/updated",
  "itemsPath": "params.plan",
  "idPath": "params.turnId",
  "contentPath": "step",
  "statusPath": "status",
  "statusMap": { "inProgress": "in_progress", "completed": "completed" },
  "toolName": "write_todos",
  "inputKey": "todos"
}]
```

Repeated notifications with the same `idPath` update one normalized plan rather
than creating provider-specific timeline entries.

### Stateful reducers

`parsing.reducers` is the stateful companion to stateless `parsing.rules`. A
reducer is an ordered JSON action interpreted by the generic protocol engine.
It can capture protocol state, emit a normalized stream event, or correlate a
subagent child thread with its parent tool.

```json
"reducers": [
  { "match": "thread/started", "capture": { "threadId": "params.thread.id" } },
  {
    "match": "item/agentMessage/delta",
    "emit": "textDelta",
    "fields": { "text": "params.delta" }
  },
  {
    "match": { "all": [
      { "on": "method", "equals": "item/started" },
      { "on": "params.item.type", "equals": "collabAgentToolCall" }
    ] },
    "parentCapture": {
      "parentIdPath": "params.item.id",
      "childThreadPaths": ["params.item.receiverThreadIds", "params.item.newThreadId"]
    }
  }
]
```

Supported emits are `textDelta`, `thinkingDelta`, `messageStart`,
`messageDone`, `toolStart`, `toolOutput`, `toolDone`, `turnStart`, `turnEnd`,
`statusMessage`, `error`, and `authError`. A reducer with no `emit` performs
state capture only. Reducers fall through to a legacy driver during migration;
fixture parity is required before removing that fallback.

### Declarative autonomy instructions

Bundled drivers may apply a mode-specific instruction declared under their
`codePlugin.config.autonomy` entry. This keeps provider policy text in the
registry while the app only provides the generic transport hook.

```json
"codePlugin": {
  "config": {
    "autonomy": {
      "plan": {
        "approvalPolicy": "untrusted",
        "sandboxMode": "read-only",
        "modeInstruction": "Do not make changes. Publish the plan before responding."
      }
    }
  }
}
```

For Codex, the plan instruction requests `update_plan`; its
`turn/plan/updated` notifications then flow through `parsing.planUpdates` into
the same plan presentation used by every adapter.

### `encoding.mode`

| Value | When to use |
|-------|-------------|
| `template` | Newline-delimited JSON input (Claude Code style). Provide `userMessage.template`. |
| `jsonrpc` | JSON-RPC 2.0 input (Codex/Copilot style). Provide `userMessage.method` and `paramsTemplate`. |
| `plugin` | The compiled driver handles encoding entirely. No `userMessage` needed. |

### `parsing.protocol`

| Value | Description |
|-------|-------------|
| `stream-json` | One JSON object per line, keyed by `parsing.discriminator` (e.g. `type`). |
| `jsonrpc-2.0` | JSON-RPC 2.0 notifications, keyed by `method`. |

### The `codePlugin` field

Most adapters need a `codePlugin` pointer because real CLI protocols involve stateful handshakes, cross-event flags, and JSON-RPC id correlation that cannot be expressed as stateless data rules.

`kind: bundled` — references a compiled-in Dart driver already shipped in the Mate binary. This is how Claude Code, Codex, and Copilot work. Community adapters cannot use `bundled` unless the driver is merged into the app.

`kind: wasm` — references a sandboxed WebAssembly module that Mate downloads and runs in an isolated sandbox on the desktop host (macOS/Windows/Linux only; mobile never executes adapter code). This is the path for community adapters that need custom stream parsing logic.

A simple CLI that emits line-delimited JSON and needs no cross-event state can omit `codePlugin` entirely and rely purely on the declarative `parsing.rules`.

---

## Contributing a new adapter

1. Fork this repository.
2. Create `specs/<your-id>.v1.json`. The `id` must match `^[a-z0-9-]+$`. Start from the minimal example below.
3. Run the validator and fix all errors:
   ```
   python3 validate.py specs/<your-id>.v1.json
   ```
4. Add an entry to `registry.json`:
   ```json
   {
     "id": "<your-id>",
     "name": "Your CLI Name",
     "version": 1,
     "specUrl": "specs/<your-id>.v1.json",
     "contentHash": "<sha256 of the spec file>",
     "schemaVersion": 1,
     "minAppVersion": "0.0.0",
     "minDriverVersion": 0
   }
   ```
   Compute the hash with:
   ```
   shasum -a 256 specs/<your-id>.v1.json
   ```
5. Open a pull request. CI runs `python3 validate.py --all`; the PR cannot merge until it passes.

### Updating an existing adapter

Published spec files are immutable. To update:

1. Create `specs/<id>.v<N+1>.json` with your changes.
2. Bump `"version"` inside the spec.
3. Update `registry.json` to point at the new file and hash.
4. Do not modify or delete the old versioned file.

---

## Minimal example

A one-shot CLI that reads a prompt from stdin and writes plain-text output. No `codePlugin` required.

```json
{
  "schemaVersion": 1,
  "id": "my-cli",
  "name": "My CLI",
  "version": 1,
  "binaryName": "mycli",
  "spawn": {
    "oneShot": true,
    "args": ["--prompt", "{prompt}"]
  },
  "encoding": {
    "mode": "template",
    "userMessage": {
      "template": {
        "type": "user",
        "text": "{prompt}"
      }
    }
  },
  "parsing": {
    "protocol": "stream-json",
    "discriminator": "type",
    "rules": [
      { "match": { "equals": "message" }, "emit": "text", "fields": { "text": "content" } }
    ]
  }
}
```

---

## Security

Community adapter specs are subject to the following restrictions enforced by the Mate runtime:

**A spec MAY:**
- Declare tool category sets, aliases, and display labels.
- Add entries to the model, effort, and slash-command catalogs.
- Specify spawn arguments, env var passthrough, and working directory using the documented variable set (`{model}`, `{effort}`, `{cwd}`, `{prompt}`, `{conversationId}`, `{systemPrompt}`, `{maxTurns}`, `{mcpConfigPath}`, `{bundledBinDir}`).
- Reference a WASM module via `codePlugin.kind: wasm` for custom stream parsing.

**A spec MUST NOT:**
- Use `codePlugin.kind: bundled` (reserved for first-party drivers compiled into the app binary).
- Relax the auto-approval decision for tools — the compiled driver always fails closed on unknown or ambiguous tools; a spec cannot override this.
- Expand the env `passThrough` list beyond the defaults — only `PATH`, `HOME`, `LANG`, `TERM`, `USER`, `SHELL` are forwarded.
- Use `spawn.command` to invoke arbitrary executables. Community specs are restricted to: `{binaryName}` resolved on PATH, `npx`, `node`, `uvx`, `python`, `deno`.
- Inject arbitrary environment variables into the spawned process beyond what the schema's `spawn.env` field allows.

WASM modules declared via `codePlugin.kind: wasm` run in a deny-by-default sandbox with no filesystem, network, subprocess, or environment access. The only grantable capabilities are `log`, `clock`, and `random`, and they require explicit user consent at install time.

---

## Validation

```
python3 validate.py specs/my-adapter.v1.json
python3 validate.py --all
```

Run tests:
```
python3 -m unittest discover tests/
```

The validator requires Python 3.8+ and no third-party packages.

### Stateful protocol reducers

`parsing.reducers` is the provider-neutral stream interpreter. Reducers run in
order and may combine a JSON-path match with `whenState`, `capture`,
`setState`, `parentCapture`, and a normalized `emit`. Field sources may be a
JSON path or `{ "state": "key" }`. `$set` and `$unset` are supported state
predicates. This covers paired tool start/stop frames and Codex collaboration
child-thread routing without adding adapter-specific state flags to the app.

Plan notifications belong in `parsing.planUpdates`. Add each current or legacy
provider notification shape as another entry; the app projects every entry to
the same stable plan card. Codex v6 demonstrates the live
`turn/plan/updated` shape.

### OpenAI-compatible HTTP/SSE adapters

Use a top-level `transport` block with `kind: "http-responses"` or
`"http-chat-completions"`. It posts the JSON emitted by
`encoding.userMessage.template`, resolves endpoint/API-key values from
`endpointEnv` and `apiKeyEnv`, and writes each SSE JSON data frame into the
normal `parsing` reducer pipeline. Header values can reference an environment
variable with `{env:NAME}`.

For raw models that support OpenAI-style function calling, add a registry-owned
`toolRuntime` block:

```json
"toolRuntime": {
  "enabled": true,
  "tools": ["read_file", "write_file", "list_files", "search_files", "run_command", "ask_user", "update_plan", "request_plan_approval"]
}
```

The app has one local executor and one permission/answer continuation path. The
spec controls the wire dialect and exposes only these logical tools; no provider
specific Dart is required. `ask_user` maps to the shared question card and
`request_plan_approval` maps to the shared **Review Plan** card. A provider
that does not return function calls remains text-only; the adapter must not
advertise a capability it cannot exercise. Version a spec and update the
registry to change tool support without rebuilding the app.
