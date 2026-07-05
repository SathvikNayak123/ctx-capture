# ctx-capture

MCP server for context-window observability: captures and lets you query exactly what an LLM
agent saw at every step, byte-accurately and framework-agnostically.

**Read [docs/DESIGN.md](docs/DESIGN.md) first.** It is the source of truth for the schema, the
MCP tool/resource surface, transport, storage, and non-goals. Don't re-derive design decisions
already made there — follow them or propose a change to the doc itself.

## Universal rules

- Byte-exact replay fidelity is the product. Any change that touches the capture path or the
  schema must not silently reduce fidelity — normalizing, reshaping, or dropping provider-native
  message/response content is the one class of change to be most suspicious of.
- The trace/context schema is versioned and additive-only within a major version (see
  docs/DESIGN.md → "The schema"). Never repurpose or remove a field in a minor version; breaking
  changes require a major version bump and a migration script.
- Every MCP tool response is size-capped (default 50 KB) and must use the
  resource-pointer/continuation-cursor pattern from docs/DESIGN.md when a payload would exceed
  it. Never return an unbounded tool result.
- Every MCP tool declares `outputSchema` and returns `structuredContent`.
- Treat captured trace data as sensitive by default (it faithfully contains whatever the agent
  saw, including any PII/secrets in tool results). The redaction hook is opt-in, not a substitute
  for treating storage as sensitive.
- New capture-path or schema changes need a corresponding fidelity/contract test, not just a unit
  test — see "Capture-fidelity acceptance test" in docs/DESIGN.md.

## Schema location

The versioned trace/context JSON schema lives in `docs/DESIGN.md` ("The schema" section) pending
extraction into a checked-in JSON Schema file in Session 1.
