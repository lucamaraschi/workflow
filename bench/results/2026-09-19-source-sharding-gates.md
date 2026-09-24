# Priority 1 source-sharding release gates

Date: 2026-09-19  
SDK checkout: `/Users/batman/src/platformatic/workflow-sdk-source-sharding-gates`  
Branch: `perf/sdk-source-sharding-gates`  
Base: `7840c15617c801e0df8f0a85145f43de25f96cc4` (latest upstream main at the start of the gate run)  
Historical gate commits: `4774f6827`, `f9a18972c`, `9030c92e0`

Rebase record (2026-09-23): the branch was cleanly rebased onto SDK
`upstream/main` `69e80c6f7` and now ends at `dd1ad1354`, including the affected-
package changeset and DCO-signed commits. The later hardening commit
`16fca55d7` is the current branch head. The post-rebase targeted smoke passed
8/8; the hardening smoke passed 9/9. The performance artifacts below are the
retained pre-rebase gate runs; the current-head refresh is a shorter unprofiled
check rather than a replacement for those gate runs.

## Decision

The technical reasons for keeping source sharding opt-in have now been exercised: mixed imports, shared modules, serde classes, hooks, manifests, inline sourcemaps, watch-mode behavior, all combined-bundle adapters, Linux Node compatibility, RSS/GC behavior, and a mixed-version Platformatic World run. They passed the gates below.

The implementation remains opt-in (`WORKFLOW_SHARD_VM_BUNDLES=1`) because the deployment artifact is larger. That is a deliberate product/rollout tradeoff, not an unresolved correctness or runtime-compatibility failure.

## Gate matrix

| Gate | Evidence | Result |
| --- | --- | --- |
| Shared local imports in every shard | Builder fixture imports `shared.ts` from two workflow sources; each selected shard contains the shared marker and excludes the other workflow marker. | Pass |
| Serde classes and manifest registration | Fixture includes `GateValue` serializer/deserializer symbols; generated manifest contains `serde.ts`; both selected shards contain the class. | Pass |
| Workflow hooks | Fixture references `WORKFLOW_CREATE_HOOK` and unique hook tokens; both selected shards retain the hook code. | Pass |
| Inline sourcemaps | Fixture build uses `sourcemap: true`; both selected shards contain an inline `sourceMappingURL=data:application/json`. | Pass |
| Deterministic graph handoff | Explicit discovered graph is reused; workflow-to-bundle map contains both workflow IDs. | Pass |
| Production/watch policy | One shared `BaseBuilder` switch; `WORKFLOW_SHARD_VM_BUNDLES=1` enables production, while `config.watch=true` disables it. | Pass (9 current-head targeted tests) |
| Builder integrations | TypeScript/build checks passed for builders, core, SvelteKit, Nitro, Astro, Nest, Next, Vitest, and World simulator. The Nitro production fixture emitted a sharded build successfully. | Pass |
| Runtime selector | Builders/core targeted tests: 9/9 current-head pass; gzip decode, line split, delta patch, cache, deterministic generation invalidation, and legacy forms are covered. | Pass |
| Real World integration | Latest World + Nitro + PostgreSQL, fanout, 3 fresh repetitions per arm, 50 steps/workflow, 1 KiB payload, concurrency 4, 20 ms polling. | Pass, 0 failures |
| Mixed World version | Candidate SDK against the previous World commit `b2598c5d729d820124156c2572a88fbb81c61cdc`, after building its workflow service package. | Pass, 0 failures |
| Linux Node 22 | Clean `node:22-bookworm` container; selector/gzip/delta smoke pass on Linux arm64. Clean dependency install plus `@workflow/utils`, `@workflow/errors`, `@workflow/core`, and `@workflow/builders` TypeScript builds pass. | Pass |
| Linux Node 24 | Clean `node:24-bookworm` container; selector/gzip/delta smoke pass on Linux arm64. | Pass |
| Memory/GC | Matched runtime-probe runs, 3 repetitions per arm. | Positive; details below |

The full recursive Linux build was intentionally not used as the acceptance command because the base container has no Rust toolchain and would try to rebuild the unchanged SWC WASM package. The existing WASM artifact was present in the clean copy; the changed TypeScript builder/core path compiled successfully.

Machine-readable platform evidence: [rn-source-sharding-gates-platform-validation-20260919.json](rn-source-sharding-gates-platform-validation-20260919.json).

## Real World fanout A/B

Artifacts: [sharded](rn-source-sharding-gates-fanout-t0-rerun.json), [control](rn-source-sharding-gates-fanout-t0-control.json).

| Metric | Monolithic control | Sharded candidate | Change |
| --- | ---: | ---: | ---: |
| Workflow runtime p50 (ms) | 3,708 | 1,475 | **−60.2%** |
| Steps/s | 50 | 130 | **+160.0%** |
| Ping p99 (ms) | 1,664 | 287 | **−82.8%** |
| Failed requests | 0 | 0 | No regression |
| Repetitions | 3 | 3 | Matched |

All runs used the same latest World build and fresh PostgreSQL databases. The earlier invalid run that lacked benchmark routes is not used as evidence; the fixture was staged, rebuilt, and the benchmark rerun successfully.

## RSS and GC

Artifacts: [sharded runtime probe](rn-source-sharding-gates-runtime-candidate-r3.json), [control runtime probe](rn-source-sharding-gates-runtime-control-r3.json).

The T0 application process across three repetitions reported:

| Runtime probe | Control | Candidate |
| --- | ---: | ---: |
| RSS peak range | 2.24–2.60 GB | 0.93–1.13 GB |
| RSS steady range | 1.34–1.37 GB | 0.86–1.01 GB |
| GC ms/completed run | 45.1–55.1 ms | 12.4–14.7 ms |
| Event-loop p99 range | 363–477 ms | 123–137 ms |

This is the expected mechanism-level result: unrelated workflow source is no longer parsed and retained in every cold replay, while the per-run VM realm remains fresh.

## Artifact tradeoff

The current 166-workflow Nitro artifact remains larger with sharding: **1,588,684 → 2,759,228 raw bytes** and **259,150 → 422,963 gzip bytes** in the original matched artifact comparison. The current fixture build shows the same direction (about **7.96 → 9.54 MB** total and **1.95 → 2.12 MB gzip**). Sharding is therefore a runtime-memory/CPU optimization, not a deployment-size optimization.

## Flame evidence

The matched source-sharding CPU capture remains the diagnostic attribution for this priority: [bare-node sharded flame](flames/profiles-2026-09-19T03-32-55.025Z__concurrency-4__rep-1__t0__pprof-cpu-bare-node-0-2026-09-19T03-32-56-109Z.html) and [raw profiled result](w1-source-shard-final-fanout-t0-flame.json). It showed the repeated VM-evaluation stacks collapsing to one selected workflow graph; the remaining leading costs were queue consumption, GC, session setup/context creation, and one VM evaluation path. The new gate runs intentionally remain unprofiled so their latency/RSS numbers are not sampling-distorted.

## Test commands

- `pnpm --filter @workflow/builders test` — **22 files / 254 tests passed** (expected duplicate-ID diagnostics are part of the fixtures).
- `pnpm exec vitest run packages/builders/src/workflow-bundle-sharding.test.ts packages/core/src/runtime/workflow-code.test.ts` — **9 tests passed** on the hardened head; the failed-rebuild publication and deterministic-cache-generation regressions are included.
- Adapter/core builds passed on the host; clean Linux Node 22 builds passed for the changed TypeScript dependency chain.

## Candidate-head refresh (2026-09-24)

The current branch head is `16fca55d710afc4f41befb16b436fa231372cc8f`, and the
focused current-head smoke passes 9/9. A fresh unprofiled two-repetition World
A/B against the same control shows:

| Metric | Control | Candidate | Change |
| --- | ---: | ---: | ---: |
| Workflow runtime p50 | 2,808.5 ms | 1,581.5 ms | **−43.7%** |
| Steps/s | 62.5 | 120 | **+92.0%** |
| Ping p99 | 1,267.5 ms | 326 ms | **−74.3%** |
| Failed requests | 0 | 0 | No regression |

The refresh used a 10-second window, fanout 50, 1 KiB payload, concurrency 4,
20 ms polling, and 200 ping/s. It is shorter and unprofiled, so the retained
three-repetition 20-second A/B and flame remain the primary performance proof.
See [`2026-09-24-sdk-candidate-head-refresh.md`](2026-09-24-sdk-candidate-head-refresh.md),
[`rn-source-sharding-candidate-head-control-20260924.json`](rn-source-sharding-candidate-head-control-20260924.json),
and [`rn-source-sharding-candidate-head-20260924.json`](rn-source-sharding-candidate-head-20260924.json).

## Recommendation

Priority 1 is ready for a gated PR: the compatibility and platform blockers are covered, and the performance/RSS benefit is reproducible. Keep the environment switch opt-in until deployment-size impact is accepted by the SDK maintainers; then enable it by default only for production builds, retaining the explicit watch-mode off switch.
