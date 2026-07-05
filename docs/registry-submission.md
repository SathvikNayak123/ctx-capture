# MCP registry submission — status

`server.json` at the repo root is a **draft**, structured against the schema and PyPI-package
example fetched live from `modelcontextprotocol/registry`'s docs
(`docs/reference/server-json/generic-server-json.md`, schema
`https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json`) during this
session. `repository.url` and `name` now point at the real repo
(`github.com/SathvikNayak123/ctx-capture`). It still hasn't been submitted — one prerequisite
remains:

- **The PyPI package doesn't exist yet.** `identifier: "ctx-capture"` and `version: "0.1.0"`
  assume the release workflow (`.github/workflows/release.yml`) has actually published to PyPI
  first — the registry entry describes a real package, not a plan for one. Push the `v0.1.0` tag
  to trigger that workflow (it needs a one-time PyPI trusted-publisher configured for this repo
  first — see the comment at the top of `release.yml`).

## To actually submit, once that's true

1. Confirm `server.json` still matches the current schema (it evolves; re-check
   `docs/reference/server-json/` in the registry repo before submitting).
2. Install the registry's `mcp-publisher` CLI and authenticate via GitHub OIDC (the registry
   verifies `name: "io.github.<org>/..."` ownership against the GitHub org/repo in `repository`).
3. Run the publisher CLI against this `server.json` from the repo root.

Nothing here calls the registry or any third-party service — this file and `server.json` are
local, reviewable artifacts pending those two prerequisites.
