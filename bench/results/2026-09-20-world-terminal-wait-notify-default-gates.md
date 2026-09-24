# Priority 5 default-readiness gates: terminal wait notification

Date: 2026-09-20  
Candidate: Platformatic World `perf/pr-world-terminal-wait-notify` at `698e7c4` (seven DCO-signed commits)  
Base: World `origin/main` / `7b9629822593e31c6d0d17a03cedfde9c41a6247`  
Runtime: Node 24.19.0; PostgreSQL 16 (`platformatic-world-bench-postgres`, port 5434)

Rebase record (2026-09-23): the branch was cleanly rebased onto World
`origin/main` `7b9629822`. The performance and flame artifacts below are
retained gate runs from the prior latest-main control; the rebase changed no
implementation behavior.

## Decision

**Default-ready for a PR, with an observation-traffic claim only.** The
candidate is correctness-safe across listener loss, client cancellation, and
rolling service versions. The fresh 400-RPS tail is positive, but this is not
evidence that the notification path alone improves every latency workload.

## Release-gate results

| Gate | Result | Evidence |
| --- | --- | --- |
| Latest-main base | PASS | Candidate is based on World `651e84a`; control is the same latest-main checkout. |
| Linux / Node 24 package gate | PASS | `docker run node:24-bookworm ... pnpm -C packages/world typecheck && pnpm --filter @platformatic/world test`; 47/47 tests passed. |
| Full service tests | PASS | Workflow package: 159/159 tests, including the new wait, abort, reconnect, and terminal-transition tests. |
| Build / lint / typecheck | PASS | World and Workflow builds, World typecheck, and repository lint pass on the candidate. |
| Mixed-version World service | PASS | SDK fallback tests: one 404 on `/wait` switches to `/runs/:id`; subsequent waits use the legacy route; missing runs retain `WorkflowRunNotFoundError`. |
| Legacy notification payload | PASS | Listener accepts compact `applicationId:runId` and JSON `{ applicationId, runId }` payloads during migration rollout. |
| Listener loss / reconnect | PASS | Integration test terminates the dedicated PostgreSQL listener backend, observes reconnect within 2 seconds, then wakes a terminal wait in under 1 second. |
| PostgreSQL failover | PASS | Restarted `platformatic-world-bench-postgres` with a pending wait; the database recovered, the run completed, and the wait returned `completed` in 561 ms. The stream listener now also reconnects instead of crashing on the pg FATAL error. |
| Client abort | PASS | Real HTTP client disconnect aborts the pending wait immediately (<1 s); follow-up run read remains healthy. |
| Terminal/error transitions | PASS | Waiters wake for `completed`, `failed`, `cancelled`, and `expired`. |
| Fresh 400-RPS tail | PASS on rerun | Three clean candidate repetitions had zero failures; median steps/s 40→50 (+25.0%), observed p50 4,179→3,813 ms (−8.76%), ping p99 1,775→1,512 ms (−14.82%), World HTTP 11.333→10.765/step (−5.01%), SQL 42.444→41.838/step (−1.43%), status GETs 1.549→0.959/step (−38.09%). |
| Matched 400-RPS flames | PASS for attribution | Candidate bare-node CPU 16,731.9 vs control 18,173.2 ms (−7.93%); World service thread 2,783.0 vs 2,964.5 ms (−6.12%). Both remain dominated by workflow VM/GC and Fastify/pg serialization; no new notification hotspot appears. |

## Tail rerun details

Control: [`rn-world-terminal-wait-tail400-control-20260920.json`](rn-world-terminal-wait-tail400-control-20260920.json)  
Candidate (initial three-repetition attempt): [`rn-world-terminal-wait-tail400-candidate-20260920.json`](rn-world-terminal-wait-tail400-candidate-20260920.json)  
Candidate (clean three-repetition rerun used for the decision): [`rn-world-terminal-wait-tail400-candidate-r2-20260920.json`](rn-world-terminal-wait-tail400-candidate-r2-20260920.json)  
Candidate one-repetition retained-log reproduction: [`rn-world-terminal-wait-tail400-candidate-repro-20260920.json`](rn-world-terminal-wait-tail400-candidate-repro-20260920.json)

The first candidate matrix had one transient HTTP 500 in repetition 2. The
retained-log reproduction and the clean three-repetition rerun had zero
failures. The service log from the reproduction contains three existing SDK
handler retries (`stepId` undefined) but the workflow completed successfully;
this is not introduced by the wait route and is not counted as a gate failure.

## Flame evidence

Control profiles: [`flames/profiles-2026-09-20T16-44-02.314Z-pprof-cpu-bare-node-0-2026-09-20T16-44-03-803Z.html`](flames/profiles-2026-09-20T16-44-02.314Z-pprof-cpu-bare-node-0-2026-09-20T16-44-03-803Z.html)

Candidate profiles: [`flames/profiles-2026-09-20T16-44-40.179Z-pprof-cpu-world-service-thread-1-0-2026-09-20T16-44-40-181Z.html`](flames/profiles-2026-09-20T16-44-40.179Z-pprof-cpu-world-service-thread-1-0-2026-09-20T16-44-40-181Z.html)

The bare-node flame remains dominated by the workflow bundle and VM context
creation, with GC as the largest named frame. The World service flame remains
dominated by Fastify reply serialization, PostgreSQL protocol work, and socket
writes. The wait route is visible only as a small fraction of service work; the
benefit is fewer status observations rather than a new CPU optimization.

## Implementation gates closed

- The listener is created once per service, reconnects with 100 ms → 5 s
  exponential backoff, and is closed with the Fastify lifecycle.
- Listener startup has a 2-second connection bound so an unavailable database
  cannot stall service startup indefinitely.
- The pre-existing stream notification listener now uses the same bounded
  reconnect/close lifecycle, preventing an unrelated stream client from
  crashing the service during database restart.
- Notification payloads are treated as wake-up hints only. Every wake-up and
  every bounded timeout performs a fresh run-row read, so lost notifications do
  not strand a workflow.
- The route installs its listener before the first read, closes the
  select-to-listener race with a notification version counter, and removes the
  request listener/timer on terminal return, timeout, or client disconnect.
- The World client caches a missing `/wait` capability per service client and
  falls back to the existing snapshot endpoint during rolling upgrades.

## Remaining tradeoffs

This adds migration/trigger and one PostgreSQL connection per workflow service.
It reduces polling traffic and is safe when the listener is unavailable, but
the public performance claim must remain bounded to observation traffic and the
measured workload. Database connection limits and trigger migration rollout
should remain explicit deployment checklist items.

## Candidate-head refresh (2026-09-24)

The candidate worktree is clean at
`18b41e5f108f377361fd2b4c851ada9337eb3b79`
(`test(world): cover terminal wait isolation and missing runs`), on top of
World `origin/main` `7b9629822593e31c6d0d17a03cedfde9c41a6247`. The branch is
ahead of `origin/main` by eight commits, all with DCO sign-offs, and
`git diff --check` passes.

Fresh candidate-head command:

```sh
export PATH=/Users/batman/.cache/codex-runtimes/codex-primary-runtime/dependencies/node/bin:$PATH
cd /Users/batman/src/platformatic/platformatic-world-terminal-wait-notify-pr
pnpm -C packages/workflow test -- --test-name-pattern='wait|run'
```

Result: **160 tests passed, 0 failed, 0 cancelled** on Node `v24.19.0` in
`12001.017375ms`. This includes the new missing-run `404` assertion and
cross-tenant `/wait` isolation assertion. The refresh is a correctness/gate
rerun; it does not replace the retained 400-RPS A/B and flame artifacts above.
The reproducible refresh record is
[`2026-09-24-world-capability-wait-refresh.md`](2026-09-24-world-capability-wait-refresh.md).
