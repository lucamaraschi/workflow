# Priority 7: inline-step limit default-readiness gates

Date: 2026-09-20  
Workflow SDK control: `7840c15617c801e0df8f0a85145f43de25f96cc4`  
Platformatic World control: `a3b9b5b2a4497ec130e3daa542846b3d9751eda1`

PR branch record (2026-09-23): `perf/inline-step-default-16` is now rebased
onto SDK `upstream/main` `69e80c6f7` at `be49d896`. Its changeset and DCO
sign-off are complete; the gate artifacts below were collected against the
retained pre-rebase control and the rebase changed no implementation behavior.

## Decision

`WORKFLOW_MAX_INLINE_STEPS=16` is a high-value setting and is now **ready for a
default-changing PR**. The two release blockers are green: a 60-second,
two-tenant T0 soak under `--max-old-space-size=4096` completed every workflow
with no OOM or failure, and the production-World saturation matrix passed on
Linux Node 22 and Node 24. The intentionally unsustainable 4 workflows/s offer
still grows backlog for both settings, and limit 16 can raise absolute
process heap/RSS as it completes more work; those are documented trade-offs,
not correctness blockers. The PR keeps `WORKFLOW_MAX_INLINE_STEPS` as an
explicit rollback/tenant-tuning override.

The benchmark campaign root only adds controls for injecting step delay,
selecting app ports, and assigning per-service metrics ports. The separate SDK
PR branch `perf/inline-step-default-16` contains the one-line default change,
the five deterministic test pins, and its changeset; Platformatic World is
unchanged.

## Matched workload results

All runs were bare Node/T0 against the latest SDK and World controls, 1 KiB
payloads, fanout-50, 20 ms return-value polling, 200 ping RPS unless noted,
fresh PostgreSQL databases, and two fresh repetitions. `limit 3` is the
latest-main control; `limit 16` is the candidate default.

| Gate | limit 3 | limit 16 | Change / result |
| --- | ---: | ---: | --- |
| Normal fanout runtime p50 | 4,004.5 ms | 2,946 ms | **−26.4%** |
| Normal fanout steps/s | 45 | 65 | **+44.4%** |
| World HTTP / step | 11.314 | 9.183 | **−18.8%** |
| World SQL / step | 42.312 | 35.222 | **−16.8%** |
| Ping p99 | 2,123.5 ms | 1,378 ms | **−35.1%** |
| Failures | 0 | 0 | pass |
| Expensive steps (25 ms every step), runtime p50 | 3,863 ms | 2,884.5 ms | **−25.3%** |
| Expensive steps, steps/s | 40 | 60 | **+50.0%** |
| Mixed steps (100 ms every 10th), runtime p50 | 2,090 ms | 1,093.5 ms | **−47.7%** |
| Mixed steps, steps/s | 88.33 | 166.67 | **+88.7%** |
| Mixed steps, World HTTP / step | 9.797 | 6.998 | **−28.6%** |
| Mixed steps, World SQL / step | 39.288 | 30.805 | **−21.6%** |

Raw artifacts:

- [default 3 normal](rn-inline-limit-p7-default3-long-20260920.json)
- [opt-in 16 normal](rn-inline-limit-p7-inline16-20260920.json)
- [expensive 3](rn-inline-limit-p7-expensive3-20260920.json) and [expensive 16](rn-inline-limit-p7-expensive16-20260920.json)
- [mixed 3](rn-inline-limit-p7-mixed3-20260920.json) and [mixed 16](rn-inline-limit-p7-mixed16-20260920.json)

The default-changing branch was rebuilt and a no-environment-variable smoke
run completed 44/44 workflows with zero failures, confirming the built
artifact selects 16 when `WORKFLOW_MAX_INLINE_STEPS` is unset: [branch smoke](rn-inline-limit-p7-default16-branch-smoke-20260920.json).

## Fairness and saturation

At an intentionally unsustainable open-loop offer of 4 workflows/s (fanout-50,
25 ms every step), both arms grew a queue. Limit 16 still reduced median
runtime from 19,746.5 ms to 15,233.5 ms, reduced p99 launch lag from 2.461 ms
to 1.966 ms, reduced end backlog from 53.5 to 49 workflows, and reduced drain
time from 34.93 s to 26.55 s. The backlog slopes were 2.583 and 2.419
workflows/s respectively: this is an improvement, not proof of sustainability.

For cross-application interference, a light c=1 app ran concurrently with a
heavy c=4 app against the same World database. The light app's p50 was 67 ms in
isolation, 98 ms beside the heavy default-3 app, and 68 ms beside the heavy
limit-16 app. Both concurrent runs had zero failures. This is a positive
fairness signal, but it is one short two-application sample, not a multi-tenant
soak.

Artifacts: [saturation 3](rn-inline-limit-p7-fair3-20260920.json), [saturation 16](rn-inline-limit-p7-fair16-20260920.json), [light baseline](rn-inline-limit-p7-multi-light-baseline-20260920.json), [light beside limit 3](rn-inline-limit-p7-multi-light-under-heavy3-20260920.json), and [light beside limit 16](rn-inline-limit-p7-multi-light-under-heavy16-r3-20260920.json).

### Extended memory-limited multi-tenant soak

Two fresh tenants shared one PostgreSQL database for 60 seconds while the
heavy tenant ran fanout-50/25-ms steps at closed-loop c=2 and the light tenant
ran fanout-10 at c=1. Both processes inherited
`NODE_OPTIONS=--max-old-space-size=4096`; external RSS sampling covered the
application and World processes. Every scheduled workflow completed and both
arms exited cleanly:

| Arm | Heavy workflows / failures | Light workflows / failures | Heavy p50 / steps/s | Light p50 / steps/s | Combined RSS peak |
| --- | ---: | ---: | ---: | ---: | ---: |
| limit 3 | 52 / 0 | 573 / 0 | 2,000 ms / 41.67 | 105 ms / 95.33 | 5.67 GiB |
| limit 16 | 80 / 0 | 829 / 0 | 1,331 ms / 65.00 | 68 ms / 138.00 | 7.05 GiB |

The higher absolute RSS is expected from the higher completed-work rate; no
process hit the 4 GiB V8 heap cap or restarted. Artifacts: [limit-3 heavy](rn-inline-limit-p7-soak-heavy3b-20260920.json), [limit-3 light](rn-inline-limit-p7-soak-light3b-20260920.json), [limit-16 heavy](rn-inline-limit-p7-soak-heavy16c-20260920.json), and [limit-16 light](rn-inline-limit-p7-soak-light16c-20260920.json).

## Memory and runtime probes

The limit-16 arm was not a memory-free win. In the normal matched run, process
RSS peak was 3.60 GB vs 3.55 GB (+1.5%) and heap peak was 3.02 GB vs 2.63 GB
(+14.8%). In the all-expensive run, RSS was 4.02 GB vs 3.84 GB (+4.8%) and
heap was 3.42 GB vs 3.13 GB (+9.2%). The saturated 4/s run moved RSS down
(4.13 GB vs 4.68 GB) because the limit-3 arm accumulated more in-flight work;
that does not remove the need for memory-limit testing.

Every measured benchmark arm completed with zero workflow failures. Runtime
probe data is embedded in the raw JSON under `runtimeProbe`.

## Flame evidence

Matched synchronized T0 profiles captured the same fanout-50 workload:

- [limit 3 flame JSON](rn-inline-limit-p7-flame3-20260920.json)
- [limit 16 flame JSON](rn-inline-limit-p7-flame16-20260920.json)

The flattened visual flame files are intentionally omitted from this compact
evidence branch because no active PR body links them directly.

The application CPU normalized from 16.245 to 11.663 ms/step (−28.2%). The
dominant `runInContext` stack remained visible (16.2% → 17.4% of sampled app
CPU) and GC stayed at 13.4% in both profiles; the setting reduces replay/World
boundaries, not the VM's fundamental cost. World traffic in the matched flame
fell from 11.315 to 9.190 HTTP calls/step and SQL from 42.308 to 35.235.

## Cancellation, shutdown, and platform gates

- Host Node 24: **112/112** targeted abort, reconnect-cancel, replay-ordering,
  and warm-deployment tests passed.
- Linux Node 24: **112/112** of the same targeted tests passed in
  `node:24-bookworm`.
- Linux Node 22: **112/112** passed in `node:22-bookworm`.
- Latest-main core suite on Node 24: **2,584 passed, 3 expected failures, 1
  skipped** (122 files passed, 1 skipped). The expected failures are existing
  fixture/diagnostic cases and were not introduced by the default change.

### Linux production-World saturation repeat

The open-loop 4 workflows/s fanout-50 workload was repeated in clean
`node:22-bookworm` and `node:24-bookworm` containers against the production
World service, with two repetitions per limit, runtime/World/database probes,
and the same 4 GiB heap cap. All four arms completed 160/160 workflows with
zero failures:

| Runtime | Limit | Runtime p50 | Steps/s | Ping p99 | App RSS peak | GC ms/run |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Linux Node 22 | 3 | 44,140 ms | 37.50 | 2,050 ms | 3.84 GiB | 259.5 |
| Linux Node 22 | 16 | 28,477 ms | 55.00 | 1,428 ms | 4.13 GiB | 179.3 |
| Linux Node 24 | 3 | 31,855 ms | 45.00 | 1,406 ms | 3.89 GiB | 92.5 |
| Linux Node 24 | 16 | 23,386 ms | 62.50 | 1,395 ms | 3.82 GiB | 60.7 |

Artifacts: [Linux Node 22 limit 3](rn-inline-limit-p7-linux22-3-20260920.json), [Linux Node 22 limit 16](rn-inline-limit-p7-linux22-16-20260920.json), [Linux Node 24 limit 3](rn-inline-limit-p7-linux24-3-20260920.json), and [Linux Node 24 limit 16](rn-inline-limit-p7-linux24-16-20260920.json).

## Post-merge follow-up

1. Keep the 4 GiB heap cap and per-tenant tail/RSS dashboards in the rollout
   checklist; the soak shows higher absolute memory in exchange for more
   completed work.
2. Run a multi-hour soak in the target deployment environment after merge to
   establish production-specific capacity limits. This is operational follow-up,
   not a blocker for the small, reversible default change.
3. Roll back per process or tenant with `WORKFLOW_MAX_INLINE_STEPS=3` (or a
   lower value) without reverting code.
