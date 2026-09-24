# Platformatic World `events.createBatch` default-readiness report

Date: 2026-09-20  
Decision: **Default-ready for matched World/SDK versions; independent rollout remains gated**

Rebase record (2026-09-23): the route branch was cleanly rebased onto World
`origin/main` `7b9629822` and now ends at `ef76a22e`; all four branch commits
carry DCO sign-off. The performance artifacts below are retained latest-main
gate runs collected before the metadata-only rebase.

## Candidate and controls

- Candidate branch: `perf/pr-world-create-batch`
- Candidate worktree: `/Users/batman/src/platformatic/platformatic-world-create-batch-pr`
- Candidate HEAD: `ef76a22eb8b47bb91e55779edce86472ea4bb19d`
- Base/control: `origin/main` at `651e84a2490f90f9459f12945383ad7c24d66c03`
- Control worktree: `/Users/batman/src/platformatic/platformatic-world-create-batch-control`
- SDK used for both arms: `/Users/batman/src/platformatic/workflow-sdk-latest-control`
- Candidate diff: 5 files, 486 insertions, 11 deletions; `git diff --check` and clean worktree pass.

## What was validated

The candidate adds an optional World batch route for clean fanout only. It accepts `step_created`, `wait_created`, and an adjacent bare `step_started` claim; all terminal, hook, attribute, retry, reconnect, and abort transitions remain on the single-event path.

This is a **new additive API**: `events.createBatch` / `/runs/:runId/events/batch`. It does not replace the existing single-event API. Existing workflows preserve parity with the established behavior because the SDK uses batching only when the optional capability exists and the workflow is eligible; otherwise it continues to issue the same single-event writes.

The current gate is for a matched World adapter/service pair. For independently rolled versions, the remaining hardening is explicit capability negotiation: add `eventsCreateBatch?: boolean` to the typed `WorldCapabilities`, require that declaration in the core batch gate, and surface the service capability through the existing runtime/version probe or a one-time capabilities endpoint. A missing or old response must cache `false` and keep the single-event path. TypeScript can validate the adapter declaration; only a runtime probe can detect a remote service mismatch.

The route validates the whole request before opening a transaction, requires spec version 6+, caps batches at 256 entries and 20 MiB, and commits accepted entries atomically. The quota preflight counts only distinct new event keys, so an exact transport retry does not consume quota twice.

## Contract and compatibility gates

| Gate | Result | Evidence |
| --- | ---: | --- |
| World build and lint | pass | candidate worktree |
| Focused unsupported/malformed/atomicity and quota tests | 29/29 | `packages/workflow/test/events.test.ts`, `packages/workflow/test/quotas.test.ts` |
| SDK suspension handler | 77/77 | latest SDK control |
| SDK World batch wire contract | 14/14 | latest SDK control |
| Workflow v5 e2e (new SDK + new World) | 5/5 | candidate e2e-v5 |
| Workflow v4 e2e (old SDK behavior + new World) | 9/9 | candidate e2e-v4 |
| Linux Node 24 workflow package | 157/157 | `node:24-bookworm` container |
| Candidate diff/worktree | pass | `git diff --check`, clean branch |

The focused matrix rejects every current unsupported event type, empty/orphan/malformed/oversized requests, and pre-v6 runs without leaking rows into the event log. The quota test proves first write, exact retry, and new-batch rejection behavior. Old World + new SDK uses the method-absent single-event fallback. New World + old SDK remains green in v4. This is additive API compatibility, not a promise that a new client can call a missing route on an old server; rollout order is therefore World route first, SDK capability second. Existing workflows remain functional through the unchanged single-event API.

## Benchmark setup

Both arms used bare Node topology `t0`, PostgreSQL at `127.0.0.1:5434`, the same latest SDK, the same workflow workload, three repetitions, pprof, World request probes, and PostgreSQL probes. The low-load arm used concurrency 1, 200 ping RPS, 50-step fanout, and 15 seconds. The tail arm used concurrency 4, 400 ping RPS, 20-step fanout, and 10 seconds.

Raw results:

- [Low-load control](w1-pr4-latest-control-20260920.json)
- [Low-load candidate](w1-pr4-latest-candidate-20260920.json)
- [400-RPS tail control](w1-pr4-tail-control-20260920.json)
- [400-RPS tail candidate](w1-pr4-tail-candidate-20260920.json)

### Low load (median across 3 repetitions)

| Metric | Latest main | Candidate | Change |
| --- | ---: | ---: | ---: |
| Steps/s | 126.67 | 163.33 | **+28.95%** |
| Workflows/s | 2.53 | 3.27 | **+28.95%** |
| Runtime p50 | 384 ms | 310 ms | **−19.27%** |
| Ping p99 | 50 ms | 39 ms | **−22.00%** |
| World HTTP / step | 8.1503 | 6.5796 | **−19.26%** |
| Service SQL / step | 37.7882 | 32.1524 | **−14.91%** |
| DB active ms / step | 22.3982 | 16.7417 | **−25.25%** |
| DB commits / step | 18.8200 | 16.0712 | **−14.60%** |
| Batch route calls / step | 0 | 0.0600 | exercised |
| Failures / backlog slope | 0 / 0 | 0 / 0 | unchanged |

### 400-RPS tail (median across 3 repetitions)

| Metric | Latest main | Candidate | Change |
| --- | ---: | ---: | ---: |
| Steps/s | 106 | 114 | **+7.55%** |
| Workflows/s | 5.30 | 5.70 | **+7.55%** |
| Runtime p50 | 729 ms | 691 ms | **−5.21%** |
| Ping p99 | 227 ms | 123 ms | **−45.81%** |
| World HTTP / step | 8.8246 | 7.7877 | **−11.75%** |
| Service SQL / step | 38.9750 | 35.0206 | **−10.14%** |
| DB active ms / step | 15.0665 | 10.0298 | **−33.43%** |
| DB commits / step | 19.8886 | 17.4598 | **−12.21%** |
| Batch route calls / step | 0 | 0.1000 | exercised |
| Failures / backlog slope | 0 / 0 | 0 / 0 | unchanged |

## Flame evidence

The flames validate the optimization mechanism. The candidate bare-node process remains dominated by workflow VM trace/context setup and GC; the new client serializer is not a new dominant hotspot. The candidate World service flame shows the batch route's `insertEvent`/pg/Fastify work alongside the existing single-event path without a new dominant service hotspot.

Representative matched artifacts:

- Low-load control: [bare Node](flames/profiles-2026-09-20T14-11-26.693Z__concurrency-1__rep-3__t0__pprof-cpu-bare-node-0-2026-09-20T14-12-05-657Z.html), [World service](flames/profiles-2026-09-20T14-11-26.693Z__concurrency-1__rep-3__t0__pprof-cpu-world-service-thread-1-0-2026-09-20T14-12-04-589Z.html).
- Low-load candidate: [bare Node](flames/profiles-2026-09-20T14-12-35.046Z__concurrency-1__rep-3__t0__pprof-cpu-bare-node-0-2026-09-20T14-13-13-729Z.html), [World service](flames/profiles-2026-09-20T14-12-35.046Z__concurrency-1__rep-3__t0__pprof-cpu-world-service-thread-1-0-2026-09-20T14-13-12-661Z.html).
- Tail control: [bare Node](flames/profiles-2026-09-20T14-13-44.566Z__concurrency-4__rep-3__t0__pprof-cpu-bare-node-0-2026-09-20T14-14-13-657Z.html), [World service](flames/profiles-2026-09-20T14-13-44.566Z__concurrency-4__rep-3__t0__pprof-cpu-world-service-thread-1-0-2026-09-20T14-14-12-592Z.html).
- Tail candidate: [bare Node](flames/profiles-2026-09-20T14-14-37.569Z__concurrency-4__rep-3__t0__pprof-cpu-bare-node-0-2026-09-20T14-15-06-689Z.html), [World service](flames/profiles-2026-09-20T14-14-37.569Z__concurrency-4__rep-3__t0__pprof-cpu-world-service-thread-1-0-2026-09-20T14-15-05-622Z.html).

The first low-load control World-service link above is intentionally a representative path; the exact machine-readable profile paths are also recorded in the four raw JSON files.

## Tradeoffs and remaining rollout constraints

- The initial capability is narrow by design; expanding it to terminal/hook/attribute/retry events needs a separate contract and benchmark.
- The request and transaction caps bound payload size and lock duration. The normal benchmark payload is 1 KiB per step; no schema migration is required.
- Quota preflight adds a distinct-key lookup but still reduces aggregate service SQL and DB active time in both matched arms.
- Reconnect, abort, and terminal transitions do not use the batch route, so their existing behavior is preserved rather than newly optimized.
- Capability negotiation is fail-closed when `createBatch` is absent. Deploy the World route before enabling an SDK that advertises the method; do not claim old-service/new-client support when the route is missing.

## Decision

**Default-ready for matched World/SDK versions.** The evidence supports the optimization and existing-workflow parity. The explicit capability handshake is still required before calling the rollout safe for independently versioned client and service deployments.
