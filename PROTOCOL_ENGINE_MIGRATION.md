# Protocol Engine Migration

## Goal

After the generic protocol engine ships, an update to Claude Code, Codex, or an
OpenAI-compatible endpoint must be published as an immutable adapter spec and
registry update. It must not require an app build unless the provider introduces
a genuinely new transport or data primitive not represented by the versioned DSL.

## Runtime contract

The app owns a provider-neutral interpreter. `mate-adapters` owns all provider
values, including method names, JSON paths, request templates, event rules,
model capabilities, autonomy policy text, plan projection, and subagent
correlation paths.

```json
{
  "transport": { "kind": "stdio-jsonrpc", "framing": "ndjson" },
  "state": {
    "threadId": { "capture": "thread/started", "path": "params.thread.id" },
    "turnId": { "capture": "turn/started", "path": "params.turn.id" }
  },
  "requests": {
    "initialize": { "method": "initialize", "params": {} },
    "startTurn": {
      "method": "turn/start",
      "requires": ["threadId"],
      "params": { "threadId": "{state.threadId}", "input": "{input}" }
    }
  }
}
```

## DSL primitives

- `transport`: `stdio-jsonrpc`, `stdio-ndjson`, `http-responses`, and `sse`.
- `state`: JSON-path captures, scoped maps keyed by item/thread/turn IDs,
  bounded delta buffers, and terminal cleanup.
- `requests`: templates, conditions, response correlation, and control-response
  templates.
- `events`: exact/regex/discriminator matches, field extraction, item-type
  dispatch, normalized event emission, and ordered reducer actions.
- `plans`: list projection, status maps, stable IDs, and plan-review policy.
- `subagents`: parent tool id, child-thread id paths, lifecycle terminals, and
  transcript fallbacks.
- `models`: runtime-discovered metadata merged with manifest fallback entries;
  per-model input modalities and constraints are data.
- `autonomy`: approval/sandbox values, developer instructions, and plan policy
  are specs, never provider conditionals in the UI.

## Standard transport profiles

`stdio-jsonrpc` covers Codex-style app servers. `stdio-ndjson` covers Claude
Code-style streams. `http-responses` provides an OpenAI-compatible Responses
profile with configurable base URL, authentication source, request template,
SSE event map, and model discovery endpoint. Vendors only need a new spec when
they use one of these profiles.

## Migration sequence

1. Build the stateful reducer engine and run built-in adapters through it in
   shadow mode, comparing normalized events against existing drivers.
2. Publish complete Claude and Codex protocol specs, including handshake,
   item lifecycle, permissions, plans, and subagents.
3. Switch built-ins to the engine. Keep legacy drivers only as version-pinned
   compatibility fallbacks while remote specs roll out.
4. Add the generic OpenAI-compatible HTTP/SSE profile and conformance fixtures.
5. Remove provider-specific parser branches once shadow comparisons remain clean.

## Release gate

Every published adapter revision must include recorded JSON fixtures for
handshake, text/reasoning, tool lifecycle, permission, plan, subagent, and
terminal paths. CI replays those fixtures through the generic engine and
rejects an unclassified method or projection drift.

## Current shipped slice

- Codex v5 declares thread/turn identity, text/thinking deltas, plan deltas, and collaboration child-thread correlation.
- Claude v3 declares message, text, thinking, tool-input, tool lifecycle, and turn reducers.
- Reducers support ordered matching, state captures, conditional state predicates, state writes, field-path and state-backed fields.
- Permission responses and transcript-backed background agents are still explicit plugin boundaries; their generic transport lifecycle is the next migration gate.

## Codex plan update compatibility

Codex v6 adds the current app-server `turn/plan/updated` notification. Its
`params.plan[]` schema is projected into the same stable plan card as legacy
`item/plan/delta`, keyed by `params.turnId`. Protocol revisions can add a new
`planUpdates` entry and registry version without changing the app.

## Native Codex modes and delegation

Codex v7 maps each Mate autonomy mode to the app-server's native
`collaborationMode` and `multiAgentMode` fields. `plan` uses the server's Plan
preset; the other modes use Default. `explicitRequestOnly` enables requested
Agent delegation without surprising proactive subagents. The driver is only a
generic serializer; the mode and delegation values are immutable adapter-spec
data.

## OpenAI-compatible Responses transport

`openai-compatible.v3.json` is a manifest-only Responses API adapter. The app
has one reusable HTTP tool runtime; endpoint/auth, request body, model
capabilities, event names, reducer paths, and the enabled logical tools are
registry data. The runtime handles standard Responses `function_call_output`
and Chat Completions `tool_calls` continuation envelopes, while the shared app
permission, question, and plan-review UI owns user decisions.

Codex v8 uses `multiAgentMode: "proactive"`. Live app-server validation showed
`explicitRequestOnly` declined an explicit delegation request, whereas proactive
emitted `collabAgentToolCall` items with receiver thread IDs. This is therefore
the required compatibility policy for the current Codex release.
