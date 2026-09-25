# Priority 2 — SDK serializer fast path and quiet logger

Date: 2026-09-20  
Candidate: `perf/pr-sdk-serialization-logger` (`cdb289393`, cleanly rebased)  
Control: latest SDK `upstream/main` (`7840c15617c801e0df8f0a85145f43de25f96cc4`)  
World: latest upstream checkout (`651e84a2490f90f9459f12945383ad7c24d66c03`)

Rebase record (2026-09-23): the branch was cleanly rebased onto SDK
`upstream/main` `69e80c6f7`. The affected-package changeset is present, all
branch commits carry DCO sign-off, and the post-rebase targeted smoke passed
86/86. A later current-head focused run passed 444/444 tests, and the value/
logger matrices were refreshed on Node 24; see [the 2026-09-24 refresh](2026-09-24-sdk-candidate-head-refresh.md).
The end-to-end performance and matched flame artifacts below remain the
retained pre-rebase gate runs; the rebase changed no implementation behavior.

## Default-readiness gates

| Gate | Evidence | Result |
| --- | --- | --- |
| Serializer value-shape compatibility | 7-case matrix: ASCII, HTML/control, Unicode fallback, Unicode separators, incompressible, typed/map/set/date, sparse array; plain/compressed/encrypted byte sizes; round-trip assertions | **PASS** — candidate/control bytes are identical; malformed payloads fail with the same `SyntaxError`; matrix `pass: true` |
| Logger semantics | Empty, unrelated, and enabled DEBUG patterns; console emission count; 7×200,000-call timing matrix | **PASS** — disabled patterns emit 0, enabled emits 1; candidate is 1.36× faster for empty DEBUG, 3.54× for unrelated DEBUG, 1.27× with DEBUG enabled |
| Core test suite | Host candidate: 122 files (121 pass, 1 skip), 2,586 passed, 3 expected failures, 1 skip | **PASS** |
| Linux/Node 22 | Clean `node:22-bookworm` install, build, and full core suite | **PASS** — same 122-file / 2,586-pass result |
| Linux/Node 24 | Clean `node:24-bookworm` install, build, and normal `pnpm --filter @workflow/core test` | **PASS** — 122 files passed, 2,586 passed, 3 expected failures, 1 skipped |
| Short workflow A/B | 15 s, 3 reps, T0 c=4, 20 steps, 256 KiB payload, latest World | **PASS** — p50 388→333 ms (−14.2%); 197.3→234.7 steps/s (+18.9%); 0 failures |
| Medium workflow A/B | 20 s, 3 reps, T0 c=4, 20 steps, 256 KiB payload, latest World | **PASS** — p50 382→321 ms (−16.0%); 204→242 steps/s (+18.6%); 0 failures |
| Logger-enabled workflow | 10 s, 1 rep per arm, `DEBUG=workflow:runtime:debug`, 1 KiB payload | **PASS** — both complete with 0 failures; 275 ms control vs 278 ms candidate (noise-level difference; this gate is semantic/safety, not a throughput claim) |
| Matched flame | 10 s, T0 c=4, 20 steps, 256 KiB payload, pprof on both arms | **PASS** — control bare-node CPU 7.24 s and `_libs/devalue.mjs` hottest; candidate 6.28 s and native code hottest. Control’s two leading `stringify_string` frames are 14.5% and 9.1% self; candidate’s leading frame is 6.1% self |

## Interpretation

The implementation is safe across the tested serializer shapes and preserves compressed/encrypted wire sizes. The fast path is intentionally selective: strings containing separators or UTF-16 surrogate pairs stay on the legacy loop. The integrated gain is payload-dependent; small payloads and incompressible data are approximately neutral, while ordinary large strings show the largest benefit.

The Node 24 Linux failure was a test-observation bug: the test concatenated `write()` and `writeMulti()` mock buckets instead of recording their chronological delivery order. The test now records delivery at the mock boundary and keeps the ordering assertion; no runtime code changed for this fix.

## Reproducible artifacts

- [value-shape matrix](rn-serialization-logging-value-matrix-20260919.json)
- [logger matrix](rn-serialization-logging-logger-matrix-20260919.json)
- [fresh microbenchmark](rn-serialization-logging-micro-20260920.json) — 75,536 equivalence checks; ASCII 256 KiB 5.11× faster; HTML/control 1.52×; Unicode fallback 1.12×
- [short candidate](rn2-sdk-short-candidate-20260919.json) / [short control](rn2-sdk-short-control-20260919.json)
- [medium candidate](rn2-sdk-medium-candidate-20260919.json) / [medium control](rn2-sdk-medium-control-20260919.json)
- [logger-enabled candidate](rn2-sdk-debug-candidate-20260920.json) / [logger-enabled control](rn2-sdk-debug-control-20260920.json)
- [Node 24/Linux full-suite result](rn-node24-sdk-core-test-20260920.json)
- Matched flame aggregates are preserved in the linked benchmark run data above; the flattened visual flame reports are intentionally omitted from this compact evidence branch because no active PR body links them directly.

## Recommendation

Priority 2 has completed its missing evidence gates and is ready for PR review. Keep the serializer patch tied to devalue `5.9.2`, retain the fallback path, and describe the gain as workload-dependent. The Node 24 test correction is test-only and should remain with this branch so the supported CI command observes the real delivery order.
