# ctx-capture — Design

Status: draft v0.1 · scope: this document only (no code yet)

## The pain

Agent failures are usually context failures, not reasoning failures: a tool result got
truncated before it reached the model, the system prompt silently dropped off after a context
compaction, a token budget overflowed and the framework cut messages from the middle, a retry
duplicated a tool call the model never saw resolve. The bug is not "the model reasoned badly" —
it's "the model never saw the thing you assume it saw."

Today, answering "what did the model actually see at step 12" requires archaeology: stitching
together app logs, framework debug output, provider dashboards, and hope. Existing LLM
observability (Langfuse, LangSmith) is built to show *what happened* — spans, durations, a trace
tree you can eyeball. It is weak at the narrower, harder claim this tool exists to make: a
faithful, replayable, byte-accurate reconstruction of the exact model input at any step, captured
in a way that doesn't depend on which agent framework you used.

ctx-capture is an MCP server for that one job: capture exactly what an LLM agent saw at every
step, and let any MCP client query it precisely.

## Decision table

Format: chosen · alternatives · why · what would change my mind.

### 1. Trace/context schema philosophy

- **Chosen**: a versioned JSON schema — trace → steps → model call (full message array as sent,
  model/params, token counts, truncation events, timestamps, cost) + tool calls (raw args,
  result as returned, result as inserted). Designed for byte-exact replay from day one; that
  replay guarantee is the schema's acceptance test, not an aspiration.
- **Alternatives**: (a) log-and-summarize — store hashes/token counts instead of full content,
  cheaper but destroys the ability to answer the product's core question; (b) OTel GenAI
  semantic-conventions shape only — good interop, but the spec doesn't mandate raw message
  capture and collectors commonly cap span attribute size; (c) store only the final flattened
  prompt string — simplest, but loses the tool-call/tool-result structure that anomaly
  detection needs.
- **Why**: the entire value proposition is "show me exactly what the model saw." A schema that
  doesn't guarantee byte-exact reconstruction has quietly become a worse Langfuse.
- **What would change my mind**: if full-fidelity storage cost turns out to be prohibitive for
  real users, the fix is opt-in sampling/redaction on top of the schema — not silently dropping
  fidelity as the default.

### 2. Capture mechanism

- **Chosen**: SDK-first — a small Python decorator/wrapper around the model client call and
  around tool functions, capturing the actual Python objects the agent code passes, before any
  framework serializes or truncates them.
- **Alternatives**: (a) OTel span ingestion — consume existing GenAI OTel spans; (b) proxy the
  model API — a reverse proxy between agent and provider that captures wire bytes.
- **Why**: the wrapper sees the tool result *before* the agent framework decides how much of it
  to keep — the only place "as returned" and "as inserted" can both be captured honestly. A
  proxy captures bytes faithfully too, but only after the client library has already serialized
  (and possibly the framework has already truncated) the payload, and it couples the project to
  each provider's wire format (breaks on provider API changes, needs TLS interception or a
  `base_url` override that fights other tooling). OTel ingestion is valuable for interop but is
  additive, not primary: today's GenAI semantic conventions don't guarantee full-message capture.
- **What would change my mind**: if most fidelity gaps turn out to be provider-side wire
  mangling rather than framework-side truncation, proxy capture would matter more. If the
  target user base is already instrumented with full-fidelity OTel GenAI spans, ingestion could
  become primary instead of an adapter.

### 3. Storage

- **Chosen**: SQLite by default (zero external dependency), Postgres optional via the same
  schema/migrations for shared or remote deployments.
- **Alternatives**: Postgres-only; a columnar/log store (DuckDB, Parquet files); cloud-only SaaS
  backend.
- **Why**: zero-dependency OSS adoption — `pip install ctx-capture` should produce a working MCP
  server with no infrastructure. The write pattern (mostly append, effectively single-writer per
  trace in local dev) and payload shape (large JSON blobs as TEXT/JSON columns) both suit SQLite
  fine. Postgres exists for the case the streamable-HTTP transport implies: multiple
  processes/users writing and querying concurrently.
- **What would change my mind**: if real traces regularly produce multi-GB databases or high
  concurrent writers, move to Postgres-first or split large payloads into blob/object storage
  with SQLite/Postgres holding pointers and metadata only.

### 4. MCP surface — tools vs. resources split

- **Chosen**: 5 tools (`list_traces`, `get_step_context`, `diff_step_contexts`,
  `find_context_anomalies`, `get_token_accounting`), each with `outputSchema` and
  `structuredContent`; traces additionally exposed as MCP resources (`trace://{id}`,
  `trace://{id}/step/{n}`) for clients that browse.
- **Alternatives**: tools-only (no resources); resources-only (no tools, everything is browsed);
  many fine-grained tools (10+) mapping one-to-one to schema fields.
- **Why**: tools are for an agentic client asking a precise question ("what did the model see at
  step 12" is a function call with args, not a browse). Resources are for a human or client UI
  exploring a trace the way a file tree is explored — debugging a bad run usually starts with
  browsing, not with knowing the exact tool-call args up front. Capping at 5 tools keeps the
  model's own tool-selection burden low; too many near-duplicate tools is a known MCP failure
  mode (the calling model picks the wrong one). Each of the 5 answers one distinct, common
  debugging question and no two overlap.
- **What would change my mind**: if usage shows MCP clients rarely support resource browsing
  well, deprioritize resource polish and go tools-first only.

### 5. Transport

- **Chosen**: stdio for local dev (primary distribution), streamable HTTP for shared/remote use.
  Bearer-token auth for remote mode now; OAuth noted as roadmap, not v1.
- **Alternatives**: HTTP-only (skip stdio); SSE (deprecated MCP transport); a custom protocol or
  gRPC.
- **Why**: stdio is the zero-config path for the primary use case — a developer running an agent
  locally who wants to inspect its own traces in the same session, no server to stand up.
  Streamable HTTP is the current MCP-blessed transport for the shared case (a team's traces
  stored centrally, queried from multiple clients) and is the successor to the deprecated SSE
  transport. Bearer token is the minimum viable gate against shipping a wide-open remote server;
  OAuth is the correct answer for multi-tenant production but is substantial scope that doesn't
  belong in a v1 whose primary distribution target is local.
- **What would change my mind**: if early adopters are predominantly running shared/team
  deployments from day one rather than local dev, pull OAuth forward into v1.

## The schema

Versioned JSON schema. `schema_version` is pinned per trace (a trace written under 1.x is always
read as 1.x). Versioning strategy: **minor versions are additive-only** (new optional fields,
never removing or repurposing a field, never changing a field's meaning) so any 1.x reader can
read any 1.x trace; **major versions may break** and require an explicit migration script.
Servers refuse (not silently coerce) traces whose major version they don't support.

```jsonc
{
  "schema_version": "1.0",
  "trace_id": "uuid",
  "created_at": "2026-07-03T18:04:00Z",
  "agent_name": "string | null",       // free text, whatever the instrumented app calls itself
  "agent_version": "string | null",
  "metadata": { /* free-form user tags, e.g. {"env": "prod", "user_id": "..."} */ },

  "steps": [
    {
      "step_index": 0,               // 0-based, ordering is authoritative
      "step_id": "uuid",
      "started_at": "iso8601",
      "ended_at": "iso8601",

      "model_call": {
        "provider": "anthropic | openai | ...",
        "model": "claude-sonnet-5",
        "params": { /* raw provider params passed through verbatim: temperature, max_tokens, ... */ },

        // The exact message array as sent to the provider API. Messages are captured as
        // opaque, provider-native objects — NOT normalized/reshaped — because normalization
        // is exactly the kind of lossy transform that breaks byte-exact replay.
        "messages": [ /* provider-native message objects, verbatim */ ],

        "request_id": "string | null",   // provider's own request id, if the client exposes one
        "response": { /* raw provider response: content blocks, stop_reason, etc., verbatim */ },

        "token_counts": {
          "prompt_tokens": 0,
          "completion_tokens": 0,
          "total_tokens": 0,
          "cache_read_tokens": 0,
          "cache_write_tokens": 0
        },
        "cost_usd": 0.0,
        "latency_ms": 0,

        // Off by default (size). When enabled, exact wire bytes for true forensic replay.
        "raw_request_bytes_ref": "string | null",
        "raw_response_bytes_ref": "string | null"
      },

      "tool_calls": [
        {
          "tool_call_id": "string",
          "tool_name": "string",
          "args_raw": { /* exact args object passed to the tool function */ },
          "result_as_returned": { /* exact return value from the tool function, pre-truncation */ },
          "result_as_inserted": { /* the version actually placed into the next model call's messages,
                                      post any truncation/summarization the agent framework applied */ },
          "started_at": "iso8601",
          "ended_at": "iso8601",
          "error": "string | null"
        }
      ],

      "truncation_events": [
        {
          "location": "string",        // json-pointer-ish path to what got truncated
          "original_size_bytes": 0,
          "truncated_size_bytes": 0,
          "strategy": "middle-out | tail-cut | summarized | unknown",
          "detected_by": "capture-sdk | inferred"  // "inferred" = detected by diffing
                                                     // result_as_returned vs result_as_inserted,
                                                     // strategy unknown to the capture layer
        }
      ]
    }
  ]
}
```

Design notes:
- `messages` and `response` are intentionally **opaque passthrough**, not normalized into a
  ctx-capture-invented shape. Normalizing is where fidelity tools usually start lying — the
  moment you reshape a provider's content-block structure into your own, you can no longer prove
  what was actually sent. The schema's job is to wrap the provider payload with enough metadata
  to query it, not to reinterpret it.
- `result_as_returned` vs. `result_as_inserted` is the field pair that makes truncation and
  drop detection possible at all — most tools only have one of these, which is why "was this
  truncated" today means diffing logs by hand.
- **Tool `args_raw`/`result_as_*` are captured as their JSON representation, not as live Python
  objects.** They persist through a JSON round-trip, so non-JSON-native types are coerced
  deterministically (tuple/set → array, bytes → string). This is deliberate and faithful to the
  product claim: tool arguments arrive *from* the model as JSON and results are inserted *back
  into* the message array as JSON, so the Python-only tuple/set/bytes distinction never crosses
  the model boundary — capturing it would record something the model never saw. The byte-exact
  opaque-passthrough guarantee applies to the provider-native `messages`/`response` payloads,
  which are already JSON. (True Python-type preservation, e.g. a type-tagged codec, is a possible
  future opt-in if a use case needs the pre-serialization object graph, but it is explicitly not
  what "byte-exact replay of what the model saw" means.)
- `truncation_events` can be populated two ways: directly by the capture SDK when it observes a
  framework's truncation call, or inferred after the fact by diffing `result_as_returned` against
  `result_as_inserted` when the SDK didn't observe the truncation directly (`detected_by:
  "inferred"`). This keeps the schema honest about confidence level instead of overclaiming
  strategy it doesn't actually know.

## MCP surface spec

### Tools

All tools declare `outputSchema` and return `structuredContent`. Every tool response is capped at
a default **50 KB** (configurable) — an MCP server that blows up its client's context window
defeats the point of a context-observability tool. Tools that could exceed the cap return a
bounded summary plus a `resource_uri` for the client to read the full payload via the resource
primitive (see below), rather than ever returning a giant tool result.

```
list_traces(filter?: {
  agent_name?: string, since?: iso8601, until?: iso8601,
  has_anomalies?: bool, tags?: object, limit?: int, cursor?: string
}) -> {
  traces: [{ trace_id, agent_name, created_at, step_count,
             total_tokens, total_cost_usd, has_anomalies }],
  next_cursor?: string
}

get_step_context(trace_id: string, step_index: int, max_bytes?: int) -> {
  trace_id, step_index,
  model_call: { ...full ModelCall as reconstructed for this step... },
  truncated: bool,          // true if this response itself was capped
  resource_uri?: string,    // set when truncated=true; full payload via resource read
  continuation_cursor?: string
}
// This is the "exact reconstructed model input" tool — the schema's acceptance test target.

diff_step_contexts(trace_id: string, step_a: int, step_b: int,
                    diff_type?: "messages" | "tokens" | "tools") -> {
  added_messages: [...], removed_messages: [...], changed_messages: [...],
  token_delta: { prompt: int, completion: int }
}

find_context_anomalies(trace_id: string,
                        types?: ["truncation","budget_overflow",
                                 "dropped_message","tool_result_mismatch"]) -> {
  anomalies: [{ step_index, type, severity, detail, byte_delta }],
  count: int
}

get_token_accounting(trace_id: string, group_by?: "step" | "role" | "tool") -> {
  total_prompt_tokens, total_completion_tokens, total_cost_usd,
  breakdown: [...]
}
```

### Resources

- `trace://{trace_id}` — trace metadata + step index, for clients that browse traces the way a
  file tree is browsed.
- `trace://{trace_id}/step/{n}` — full step detail, including the full byte-exact context. This
  is where the large payload lives when a client wants to read it directly rather than through a
  size-capped tool call.

### Tools vs. resources — why both

A tool call is the right shape when the caller (usually the model itself, debugging its own
prior run, or an agent orchestrating a fix) knows exactly what it wants and needs a small,
structured, reliable answer back in its own context. A resource is the right shape when a human
or a client UI is exploring — browsing recent traces, opening one, paging through steps — where
the read is opt-in and the client controls when the (potentially large) payload enters context.
Collapsing these into "tools only" would force browsing UIs to guess trace/step IDs blind or
enumerate via a tool in a loop; "resources only" would force the model itself to always fetch and
parse a full trace to answer a one-field question like "how many tokens did step 12 cost."

### Pagination and size limits

- `list_traces` uses standard cursor pagination (`limit`/`cursor` in, `next_cursor` out).
- `get_step_context` and `diff_step_contexts` apply a default 50 KB response cap. When the
  reconstructed payload would exceed it, the tool returns a truncated-but-labeled summary
  (`truncated: true`) plus a `resource_uri` for the full content and, where applicable, a
  `continuation_cursor` for chunked re-fetching via the same tool. The cap is configurable per
  deployment; it is never silently raised by returning partial data without the `truncated` flag.

## Capture-fidelity acceptance test

This is the schema's acceptance test, not a nice-to-have:

> Given a fixture agent making real model calls, the capture SDK's recorded `messages` array for
> a step, after canonical JSON serialization, must be **byte-identical** to the actual outbound
> request body's messages field for that same call, as observed by an independent test-only
> interception point (a local recording proxy used only in the test harness, never in
> production). This must hold across a fixture matrix covering: plain text messages, tool-call
> messages, tool-result messages, multi-block content (text + tool_use + tool_result mixed),
> image content blocks, and at least one deliberately-oversized tool result that the framework
> truncates before reinsertion (asserting `result_as_returned` ≠ `result_as_inserted` and a
> `truncation_events` entry is recorded).

This harness runs in CI as a required gate before any release. Today "supported provider" means
one capture path — OpenAI-compatible `chat.completions`-shaped clients — exercised against the
fixture matrix above; an Anthropic Messages API adapter (or others) is roadmap, not implemented.

## Non-goals

- **No dashboards/UI.** MCP clients (Claude Desktop and others) are the UI. We are not rebuilding
  Langfuse's trace viewer. Reasoning: a UI is a large, separate product surface with its own
  maintenance burden; the MCP interface *is* the product here.
- **No eval features** (no scoring, no LLM-as-judge grading). Reasoning: evals are a mature,
  separate category with a different data model and workflow (Langfuse, Braintrust, etc.);
  bolting on a half-featured eval system dilutes the one job this tool does well.
- **No multi-agent trace stitching in v1** (no cross-agent/cross-process causal graphs).
  Reasoning: single-agent, single-process step fidelity is already an under-served, hard problem.
  Stitching requires a correlation-ID / causality design that's premature before the single-agent
  core is proven solid.
- **No prompt management** (no versioning, templating, or a prompt registry). Reasoning: that's a
  build-time authoring concern; ctx-capture is a runtime observability concern. Different
  lifecycle, different tool — conflating them is how tools become unfocused.

## Session map

- **Session 1** — schema + storage layer: finalize the JSON schema above as a checked-in JSON
  Schema file, SQLite persistence layer, capture SDK core (decorator/wrapper around model client
  + tool functions). Unit tests for schema round-tripping. No MCP server yet.
- **Session 2** — MCP surface: implement the 5 tools and 2 resource types over stdio transport,
  with `outputSchema`/`structuredContent`, pagination, and the size-cap/resource-pointer pattern.
  Contract tests per tool (schema validation, pagination, size limits).
- **Session 3** — fidelity + hardening: capture-fidelity acceptance harness (the byte-identical
  test above) as a CI gate, capture-overhead benchmark (published, measured latency added per
  call), streamable HTTP transport + bearer auth, Postgres-optional storage backend, README with
  60-second quickstart and an honest Langfuse/LangSmith comparison table, CI (lint/test/release),
  semver + changelog, license, v0.1 release.

## Risks and mitigations

- **Payload size** — full-fidelity capture means large traces (big tool results, long message
  histories) inflate storage and risk oversized MCP responses. *Mitigation*: the
  pagination/resource-pointer pattern above keeps every tool response bounded regardless of trace
  size; storage offers a configurable capture-depth option (e.g., cap raw-bytes capture, keep
  hashes for anything beyond a size threshold) as an explicit, documented opt-out of full
  fidelity rather than a silent one.
- **PII in captured contexts** — tool results and messages routinely contain PII, credentials, or
  other sensitive data, and this tool's entire purpose is to capture that faithfully.
  *Mitigation*: a pluggable **redaction hook** — `redact(message) -> message` — run at capture
  time, before persistence, with common patterns (emails, API keys, secrets) available as an
  opt-in default ruleset. Off by default (fidelity-first), but the README and quickstart state
  plainly that captured traces should be treated as sensitive data by default, same as any
  request log.
- **Fidelity claim being wrong** — a silent capture gap (a message type the SDK doesn't handle,
  a framework transform it doesn't see) would quietly break the one promise the product makes.
  *Mitigation*: the fidelity acceptance harness above runs as a required CI gate for the
  currently-supported capture path, not an optional test, and grows a gate per provider as each
  new capture path (e.g. an Anthropic adapter) ships.
- **Vendor/provider API changes** altering message or response shapes. *Mitigation*: the schema
  treats `messages`/`response` as opaque passthrough rather than validating provider internals,
  which minimizes what can actually break when a provider changes shape.
- **Framework coupling** — the SDK wrapper touching too much of an agent framework's internals
  makes it brittle across framework versions. *Mitigation*: keep the wrapper surface to the
  model-client call boundary and tool-function boundary only (not framework internals), and
  maintain a compatibility matrix with contract tests pinned to supported framework versions.
