# MCP registry submission — status

`server.json` at the repo root is a **draft**, structured against the schema and PyPI-package
example fetched live from `modelcontextprotocol/registry`'s docs
(`docs/reference/server-json/generic-server-json.md`, schema
`https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json`) during this
session. It has not been submitted — it can't be yet, for two concrete reasons:

1. **`repository.url` is a placeholder** (`github.com/ctx-capture/ctx-capture`). Replace it with
   the real repo once one exists, in both `server.json` and `pyproject.toml`'s `[project.urls]`.
2. **The PyPI package doesn't exist yet.** `identifier: "ctx-capture"` and `version: "0.1.0"`
   assume the release workflow (`.github/workflows/release.yml`) has actually published to PyPI
   first — the registry entry describes a real package, not a plan for one.

## To actually submit, once both of those are true

1. Confirm `server.json` still matches the current schema (it evolves; re-check
   `docs/reference/server-json/` in the registry repo before submitting).
2. Install the registry's `mcp-publisher` CLI and authenticate via GitHub OIDC (the registry
   verifies `name: "io.github.<org>/..."` ownership against the GitHub org/repo in `repository`).
3. Run the publisher CLI against this `server.json` from the repo root.

Nothing here calls the registry or any third-party service — this file and `server.json` are
local, reviewable artifacts pending those two prerequisites.
