# ctx-capture — Adversarial Audit Report

**Auditor role:** independent, adversarial. Nothing below is taken from the README/DESIGN on
trust; every PASS/GAP was produced by running, grepping, or byte-comparing on a clean checkout
(Python 3.13.5, Windows 11, fresh venv, `pip install -e ".[dev]"`).

---

## Verdict: **FIX-FIRST**

The engine is real and better-tested than most v0.1 repos: the schema round-trips faithfully, the
5-tool + 2-resource MCP surface works end-to-end over real stdio transport, pagination provably
serves a >1 MB step without silent drop, and anomaly detection fires on genuinely captured
traces. **34/34 tests pass.**

But the two *headline* claims are overstated or broken, and both are load-bearing for this
project's entire pitch:

1. **The flagship "byte-for-byte" fidelity guarantee is under-tested versus its own design spec
   and has demonstrable holes** (positional tool args dropped; tuple/set/bytes coerced; no real
   provider or wire-serialization ever exercised).
2. **The documented 60-second quickstart does not run** — `uvx ctx-capture` 404s because the
   package was never published to PyPI (and the publish step was deliberately removed).

Neither is fatal-in-place: I found **no evidence the capture path corrupts data for the common
dict/kwargs message path** (my independent byte-compare matched, order-sensitive). So this is not
NOT-READY. But shipping a fidelity product whose fidelity test is near-tautological, and a
"60-second quickstart" that errors on line 1, is precisely what an auditor should block on. Fix
those two, and this is a SHIP.

---

## PASS / GAP table

### Universal checks

| # | Check | Result | Evidence / Fix |
|---|---|---|---|
| U1 | Clean-clone: documented run works | **GAP** | README step 1 is `uvx ctx-capture --db my_agent.db`. The package is **not on PyPI** (`GET pypi.org/pypi/ctx-capture/json` → **404**), and `release.yml` explicitly dropped the publish step (commit `9d9b6ea`). `uvx ctx-capture` cannot resolve. Undocumented steps actually needed: `git clone` → `pip install -e .` → `python -m ctx_capture.mcp --db x.db`. **Fix:** either publish to PyPI, or change the quickstart to `uvx --from git+https://github.com/… ctx-capture` / a pip-from-source path, and stop telling readers `uvx ctx-capture` works. |
| U2 | Claim → artifact tracing | **PASS (1 gap)** | Overhead `~0.017ms` → `docs/RESULTS.md` + reproduce script (traced). Fidelity → `tests/test_fidelity.py`. Real-client → `docs/proof/*transcript.txt` (regenerated cleanly by me). **Gap:** the DESIGN + CHANGELOG claim the fidelity harness runs "per supported provider" — there is **no artifact for any real provider**; the only provider is an in-memory fake. |
| U3 | Reproduce one headline number at smoke scale | **PASS (noisy)** | Re-ran `scripts/bench_overhead.py`: **median 0.0168 ms — exact match** to published 0.0168; mean 0.0212 vs published 0.0174 (~20% high). Median (the stable stat) reproduces; mean is noise-dominated at sub-µs scale (no warmup / outlier trimming). `~0.017ms` is defensible. |
| U4 | CI actually gates on regression | **PARTIAL** | `ci.yml` runs `pytest -v`; a failing `test_fidelity.py` fails the job (verified: `pytest` returns non-zero → job red), so the fidelity gate is real. **Gaps:** the overhead number is *not* re-checked in CI (no threshold logic), and there is **no red run in history** to demonstrate the gate firing (repo has only `main`, no PRs). |
| U5 | Design-doc ↔ code integrity | **PASS (3 drifts)** | Decision table mostly matches code. Drifts: (a) "servers refuse (not silently coerce) traces with an unsupported major version" — **not implemented** (`grep` for refusal logic in `src/` → nothing; `sqlite_repo.get_trace` reads any `schema_version` verbatim); (b) fidelity harness "per supported provider" — fake client only; (c) fixture matrix (below) incomplete. |
| U6 | Eval economics | **N/A (honest)** | Project has **no evals / no LLM judge** by design (explicit non-goal, honored). CI runs pure `pytest`, zero API calls → ~$0 per PR cycle and per nightly. No cheap/expensive-judge concern exists. |
| U7 | Secrets / hygiene | **PASS (1 note)** | `git log -p --all` → no keys/tokens committed; no `.env`/credential files tracked; `.gitignore` covers `.streamlit/secrets.toml`. MIT `LICENSE` present; no datasets to license. **Note:** deps are floor-pinned (`pydantic>=2`, `mcp>=1.26`), not locked — fine for a library, but builds aren't byte-reproducible. |
| U8 | README voice | **PASS (1 note)** | Prose is largely evidence-backed and hedged. **Note:** "**byte-for-byte**" (title + §"The fidelity test") is stronger than the test proves — the test canonicalizes with `sort_keys=True` (order-insensitive) and never touches wire bytes. See P1. |

### Project checks — Context-Capture MCP Server

| # | Check | Result | Evidence / Fix |
|---|---|---|---|
| P1 | **Fidelity guarantee** (highest priority) | **PASS on behavior · GAP on test rigor** | I wrote my own 4-step agent, instrumented it, and **independently reconstructed step 3's `messages` from storage, byte-comparing order-sensitively (no `sort_keys`)** against an out-of-band observation → **match**. Storage preserves key order; `deepcopy` correctly protects the snapshot against post-call mutation (captured 5 msgs, not the one appended after the call). **But** the repo's own `test_fidelity.py` is near-tautological: its "independent" point (`FakeOpenAIClient.call_log`) is a `deepcopy` of the *same in-memory list* the wrapper `deepcopy`s, and it compares with `sort_keys=True`. It never exercises a real provider SDK's serialization — which the DESIGN spec explicitly requires ("a local recording proxy … the actual outbound request body"). **No mismatch found in the real capture path**, so not NOT-READY; but the guarantee is proven only for an in-memory dict round-trip, not "what the model saw" on the wire. |
| P1b | Fidelity — demonstrable holes | **GAP** | (a) **Positional tool args are silently dropped:** `recorder.py:113` captures `copy.deepcopy(kwargs)` while the wrapper is `def wrapped(*args, **kwargs)`. Demonstrated: `wt('weather', 5)` → `args_raw == {}`. DESIGN promises "exact args object passed to the tool function." (b) **Non-JSON types coerced:** a tool returning `{"pair": (1,2)}` round-trips through storage as `{"pair": [1,2]}` (tuple→list). Both undercut byte-exactness for the *tool-call* half of the schema; both are invisible to tests because the toy agent only ever calls tools with kwargs and JSON-native values. **Fix:** capture `args` positionally (bind against `inspect.signature`) and document that storage is JSON-typed (or store a type-tag). |
| P1c | Fidelity — fixture matrix | **GAP** | DESIGN's acceptance test mandates coverage of *plain text, tool-call, tool-result, multi-block mixed content, **image content blocks**, and oversized truncation.* The actual `toy_agent` covers only text + tool_calls + tool-results + oversized. **No multi-block-mixed and no image-content-block fixture exists.** (Image blocks *do* round-trip in my probe, but the required test isn't there.) |
| P2 | Truncation truth (pre + post + flagged) | **PASS** | `test_fidelity.py::test_truncation_…` + my dogfood run confirm: `result_as_returned` (5000 B) ≠ `result_as_inserted` (truncated), a `truncation_events` entry is recorded, and `find_context_anomalies` flags it (dogfood step 3: `truncation (high)` + `tool_result_mismatch (high)`). **Minor:** one truncation yields **two** anomalies (`truncation` *and* `tool_result_mismatch`) — inflates counts for a single event. |
| P3 | Payload discipline (serve > client window) | **PASS** | `test_large_trace_pagination.py` (ran): a **>1 MB** single step is served across **≥20 capped pages**, each `≤50 000 B`, and the reconstructed message list equals the captured one — no silent drop. Cursor/`resource_uri`/`truncated` flags all set. The irony is avoided. |
| P4 | Contract tests + fuzz | **PASS** | 34/34 pass. I fuzzed all 5 tools with malformed args (wrong types, missing required, bad enum, unknown tool): every case returns protocol-level `isError=True` with a pydantic validation message — **no crashes, no raised exceptions leaking to the client**. `limit:-5` is clamped (returns empty, not error) — acceptable. |
| P5 | Overhead claim reproduces | **PASS (noisy)** | See U3 — median exact, mean noisy-high. |
| P6 | Real-client proof / 60-second | **PASS on capability · GAP on the "uvx 60-second" claim** | Ran `scripts/manual_verify.py` and `scripts/dogfood.py`: both spin up `python -m ctx_capture.mcp` as a **real stdio subprocess**, an actual `mcp.ClientSession` connects, lists 5 tools (all `outputSchema=yes`) + 2 resource templates, browses a `trace://` resource, and pulls a mid-run step's exact context. **Works.** But the *documented* 60-second path (`uvx ctx-capture`) is broken (U1), so the "60-second quickstart" as written is not honestly timeable on a clean machine. |
| P7 | Comparison-table honesty | **PASS (1 slanted cell)** | Spot-checked vs. Langfuse docs: "Evals = core feature" ✓ (Langfuse ships LLM-as-a-Judge), "Multi-agent stitching = Yes" ✓ (nested traces/observations). The **"Traces are typically reshaped for display; framework-level truncation is usually invisible"** cell is the most contestable — Langfuse *does* retain the raw input/output you log — but is hedged ("typically"/"usually") and the pre/post-truncation point is fair. Not a misstatement. |
| P8 | Schema exported / versioned / stamped | **PASS (1 gap)** | `schema/v1.0.json` is present and **current** (re-ran `export_schema.py` → zero diff). `SCHEMA_VERSION="1.0"` defaults onto every `Trace`, is persisted to the `traces` table, and round-trips (verified). **Gap:** the "refuse unsupported major version" promise (U5a) is unimplemented. |

### Additional finding (not in the checklist)

| # | Finding | Result | Evidence |
|---|---|---|---|
| A1 | Provider coverage overstated | **GAP** | CHANGELOG/DESIGN say "framework-agnostic" and the schema enumerates `provider: "anthropic \| openai"`, but **only the OpenAI-shaped `client.chat.completions.create` is wrapped**. Anthropic's Messages API (`client.messages.create`, different shape) has no capture path yet. README line 33 ("any OpenAI-compatible client") is honest; the broader "framework/provider-agnostic" framing is not. **Fix:** scope the wording to "OpenAI-compatible chat-completions," or add an Anthropic adapter. |

---

## The three weakest points an interviewer would attack

1. **"Your flagship fidelity test proves almost nothing."** It compares two `deepcopy`s of the
   *same in-memory list* and then canonicalizes with `sort_keys=True`, so it can't even catch a
   key-reorder — yet the product is sold as "byte-for-byte." The DESIGN doc itself specifies a
   *recording proxy observing the actual outbound request body, per supported provider*; the code
   ships a fake client and zero real providers. The test verifies storage round-trips, not
   fidelity to what a real SDK serializes onto the wire.

2. **"Your capture path already loses fidelity, and your tests are shaped to miss it."**
   Positional tool args → `args_raw == {}` (`recorder.py:113`, `kwargs`-only), and tuples/sets/
   bytes silently coerce through the JSON storage round-trip. The toy agent only ever calls tools
   with kwargs and JSON-native values, so the very holes in the fidelity claim are the ones the
   fixtures avoid.

3. **"I followed your README and it errored on line 1."** `uvx ctx-capture` 404s (never
   published; publish step removed in `9d9b6ea`), and `server.json` / the registry submission
   point at the same non-existent PyPI package. For a tool whose pitch is rigor, a headline
   quickstart that can't run is the easiest possible hit.

## The single strongest evidence-backed interview claim

**"The MCP query surface is real, disciplined, and provably safe against its own core failure
mode."** Five tools + two resources, each with `outputSchema`/`structuredContent`; a byte-size
pagination + resource-pointer pattern that I watched serve a **>1 MB, context-window-sized step
across 20+ pages each ≤50 KB with zero silent drop** (`test_large_trace_pagination.py`); clean
protocol-level error shapes under adversarial fuzzing (no crashes); and it all works
**end-to-end from a real MCP client over real stdio transport** (`dogfood.py` /
`manual_verify.py`, run live). A context-observability server that refuses to blow up its own
client's context window — and demonstrates it under test — is a genuinely shippable, defensible
piece of engineering.

---

### Top fixes, in priority order

1. Publish to PyPI (or fix the quickstart command + `server.json` to a runnable path). *(U1, P6)*
2. Make `test_fidelity.py` bite: serialize through a real (or realistically-serializing) client,
   compare **order-sensitively**, and add the missing image/multi-block fixtures. *(P1, P1c)*
3. Capture positional tool args and document/tag non-JSON types. *(P1b)*
4. Implement the promised unsupported-major-version refusal, or soften the promise. *(U5a, P8)*
5. Scope "framework/provider-agnostic" to what's implemented, or add an Anthropic adapter. *(A1)*

*Method note: all "ran" claims were executed in a throwaway venv; every file I touched
(`docs/`, `schema/`) was restored via `git checkout`, leaving the working tree clean.*
