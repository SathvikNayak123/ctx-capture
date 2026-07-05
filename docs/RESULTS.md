# ctx-capture — capture-overhead benchmark

Measured: 2026-07-03T07:34:05.195659+00:00
Python: 3.13.5 (Windows 11)
Iterations: 2000 calls per arm, in-process fake client (isolates SDK overhead from network/provider latency)

| | unwrapped (ms) | wrapped (ms) |
|---|---|---|
| mean | 0.0007 | 0.0181 |
| median | 0.0007 | 0.0175 |
| p95 | 0.0007 | 0.0221 |

**Added overhead per instrumented call: 0.0174 ms mean / 0.0168 ms median.**

Config: default capture path (deepcopy of messages/params on every call, no redaction hook, no
raw-bytes capture). Measures the in-memory capture wrapper only, not the storage write path.

Reproduce: `python scripts/bench_overhead.py`
