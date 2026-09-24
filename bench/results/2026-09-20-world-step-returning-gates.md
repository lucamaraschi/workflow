# Priority 3 validation: World SQL `RETURNING` mutation rows

Date: 2026-09-20  
Candidate: `perf/pr-world-step-returning` at `45ee148b01415acee789f3b456e28b3971e02759`  
Base: `platformatic-world` `651e84a2490f90f9459f12945383ad7c24d66c03`

## Decision

The contract and compatibility gates are green, and the candidate consistently reduces World SQL work. It is **not default-ready**: both fresh 400-RPS fanout arms show a non-positive end-to-end tail result. Keep the branch and PR body ready, but hold the PR until the tail regression is explained or removed.

## What changed

`packages/workflow/plugins/events.ts` now returns and formats the mutated step/wait row for the relevant insert/update paths. The redundant post-mutation `SELECT ... WHERE id = $1` calls are gone on normal mutation paths. Terminal/duplicate checks retain narrow lock reads and fetch a full row only for duplicate responses, transaction boundaries are unchanged, and `formatStep`/`formatWait` remain the response boundary.

The branch also adds `packages/workflow/test/returning-contract.test.ts`, which checks that the response contains only the public step/wait fields and preserves values across create, duplicate, start, complete, fail, retry, lazy-create, and duplicate terminal paths.

## Contract and mixed-version gates

| Gate | Result |
| --- | --- |
| Focused RETURNING contract suite | 2/2 suites passed |
| Candidate full suite | World 45/45, Workflow 156/156, Fastify 8/8 |
| Node 24/Linux Docker package + focused contract gate | 45/45, 156/156, 8/8, focused 2/2 |
| Workflow v4.8.9 + World v4.5.0 | e2e-v4 9/9 passed |
| Workflow v5.0.0-beta.53 + World v5.0.0-beta.36 | e2e-v5 5/5 passed |

The Linux Docker e2e-v4 attempt timed out in the harness and is deliberately not counted as a pass. The local v4/v5 gates are the mixed-version compatibility evidence; the Linux unit/integration and focused contract suites pass.

## Fresh benchmark evidence

All arms use the latest Workflow SDK control, latest World control or this candidate, fresh PostgreSQL databases, three repetitions, and the same T0 runner. Raw JSON and rendered pprof flames are linked in the JSON companion report.

### Low load / short sequential workflow

T0, 10 seconds, 20 ping RPS, concurrency 1, five sequential steps, 1 KiB payload.

| Metric (median across repetitions) | Latest-main control | Candidate | Change |
| --- | ---: | ---: | ---: |
| World SQL statements/step | 27.1116 | 25.0514 | **−7.60%** |
| PostgreSQL active ms/step | 1.2365 | 1.2308 | **−0.47%** |
| Steps/s | 64 | 69.5 | **+8.59%** |
| Runtime p50 | 68 ms | 66 ms | **−2.94%** |
| Ping p99 | 8 ms | 6 ms | **−25.00%** (small/noisy) |
| Failures | 0 | 0 | — |

Control: [`w1-pr3-low-control-20260920.json`](w1-pr3-low-control-20260920.json)  
Candidate: [`w1-pr3-low-candidate-r2-20260920.json`](w1-pr3-low-candidate-r2-20260920.json)

### 400-RPS fanout tail

T0, 10 seconds, 400 ping RPS, concurrency 4, twenty fanout steps, 1 KiB payload. Two initial independent arms were run, including a candidate-first order-balanced rerun, followed by a final arm after narrowing terminal lock reads (the final branch refinement).

| Arm | SQL/step control → candidate | Steps/s control → candidate | Runtime p50 control → candidate | Failures |
| --- | ---: | ---: | ---: | ---: |
| First matched arm (pre-refinement) | 38.471 → 37.258 (**−3.15%**) | 106 → 80 (**−24.53%**) | 702 → 938 ms (**+33.62%**) | 0 / 0 |
| Candidate-first rerun (pre-refinement) | 39.042 → 37.399 (**−4.21%**) | 96 → 82 (**−14.58%**) | 834 → 947 ms (**+13.55%**) | 0 / 0 |
| Final terminal-read refinement arm | 39.125 → 37.367 (**−4.49%**) | 92 → 82 (**−10.87%**) | 843 → 955 ms (**+13.29%**) | 0 / 0 |

PostgreSQL active time falls in all arms (for the final arm, 13.99 → 10.66 ms/step), so the SQL mechanism is real; it does not translate into a stable application tail win. The terminal-read refinement was measured independently and did not remove the regression. Ping p99 is lower or tied, but workflow runtime and throughput are the decision metrics here.

## Row-width and memory tradeoff

Composite PostgreSQL row measurements show that `RETURNING *` adds only row metadata over the narrow formatter projection: +16 B for an empty step, +16 B for a 1 KiB step, +24 B for a 64 KiB step, +24 B for a 1 MiB step, and +20 B for an empty wait. In a 100-query × 5-repetition Node `pg` check on a representative 64 KiB row, JSON serialization differed by 52 B and V8 serialization by 48 B. Post-GC heap deltas were noisy and showed no material retention increase.

This clears the row-width/memory concern for the tested payloads, but it does not explain the fanout tail regression.

## Flame evidence

Rendered HTML/Markdown flames are available in:

- Low-load control: `flames/flames/profiles-2026-09-20T05-44-03.038Z-pprof-cpu-bare-node-0-2026-09-20T05-44-31-308Z.html-pprof-cpu-bare-node-0-2026-09-20T05-44-31-308Z.html`
- Low-load candidate: `flames/flames/profiles-2026-09-20T06-04-20.956Z-pprof-cpu-world-service-thread-1-0-2026-09-20T06-04-48-052Z.html-pprof-cpu-world-service-thread-1-0-2026-09-20T06-04-48-052Z.html`
- Tail control: `flames/flames/profiles-2026-09-20T05-45-54.554Z-pprof-cpu-world-service-thread-1-0-2026-09-20T05-46-22-763Z.html-pprof-cpu-world-service-thread-1-0-2026-09-20T05-46-22-763Z.html`
- Tail candidate: `flames/flames/profiles-2026-09-20T05-46-50.470Z-pprof-cpu-bare-node-0-2026-09-20T05-47-19-390Z.html-pprof-cpu-bare-node-0-2026-09-20T05-47-19-390Z.html`
- Final-arm tail control: `flames/flames/profiles-2026-09-20T06-05-31.227Z-pprof-cpu-bare-node-0-2026-09-20T06-06-00-284Z.html-pprof-cpu-bare-node-0-2026-09-20T06-06-00-284Z.html`
- Final-arm tail candidate: `flames/flames/profiles-2026-09-20T06-02-13.383Z-pprof-cpu-world-service-0-2026-09-20T06-02-42-188Z.html-pprof-cpu-world-service-0-2026-09-20T06-02-42-188Z.html`

The low-load profiles retain the same workflow-VM and Node/native/pg parsing shape. The tail profiles show the candidate still dominated by the workflow VM and GC in bare Node, with PostgreSQL/pg parsing on the World thread; there is no clean candidate-only flame signature proving a latency win. The first and final arms have rendered flames. The middle order-balanced rerun produced `.pb` profiles but not rendered flames because the renderer process lacked `node` on PATH; those are not used as the rendered-flame claim.

## Recommendation and next actions

- Keep the implementation and contract test on `perf/pr-world-step-returning`; do not push or open the PR yet.
- Investigate a narrow projection that returns exactly the formatter fields, or another way to retain the round-trip reduction without increasing the fanout response/decoding path. The terminal-read refinement is insufficient by itself.
- Re-run the low-load and two-arm 400-RPS matrix after the next variant with `node` available to the flame renderer, plus the existing contract, mixed-version, and Linux gates.
- Open only if fanout tail reaches control parity (or the regression is explained and explicitly accepted); the PR must claim reduced SQL work rather than universal latency improvement.
