# SDK replay page-limit Priority 6 — default-readiness gates

Date: 2026-09-20  
SDK branch: `perf/pr-sdk-replay-page-limit`  
SDK base: `upstream/main` `69e80c6f7e83731efcd3a8004a96ef7d70aa22d9`  
SDK HEAD: `b39acc7d` (changeset and DCO-signed commits)  
World control: `/Users/batman/src/platformatic/platformatic-world-latest-control` (`a3b9b5b`)

Rebase record (2026-09-23): the branch was cleanly rebased onto SDK
`upstream/main` `69e80c6f7`. The post-rebase targeted smoke is 97/97; the
performance artifacts below are retained pre-rebase gate runs and the rebase
changed no implementation behavior. The benchmark workbench files remain
untracked and are not part of the branch.

## Decision

Priority 6 is **default-ready for review**; nothing has been pushed or opened. The net SDK change requests 500 events per replay page. The earlier 1,000-event experiment is retained in git history but is not the proposed implementation.

The original blocker was real: the 1,000-event candidate produced candidate-only HTTP 500s and duplicate terminal-step attempts under the c4/200-step fanout gate. Retained logs show the failure occurring after the large replay burst, not in the benchmark harness. A 500-event cap removes the failure in the repeated gate while retaining most of the read reduction.

## Acceptance matrix

| Gate | Result | Evidence |
| --- | --- | --- |
| c4 / 200 steps / 20 s / 3 reps | **Pass** — 0 failures in every repetition; cursor reads ~0.75–0.86/step | `rn-sdk-replay-page-limit-p6-candidate500-20260920.json` |
| Short history (20 steps) / 3 reps | **Pass** — 0 failures; observed p50 840 ms | `rn-sdk-replay-page-limit-p6-short500-20260920.json` |
| 400-RPS tail / 50 steps / 3 reps | **Pass** — 0 failures; observed p50 3,755 ms | `rn-sdk-replay-page-limit-p6-tail500-20260920.json` |
| >1,000-event history / 400 steps / controlled open-loop | **Pass** — 1,208 events, 400 steps, 0 failures | `rn-sdk-replay-page-limit-p6-long400-open005-20260920.json` |
| Memory and response bytes for >1,000-event history | **Measured** — normal run RSS peak ~757 MB / heap peak ~408 MB; World event payload 215,685 bytes; optional byte-counted diagnostic saw exact event-list HTTP bodies 159,630,013 bytes across initial/cursor reads | `rn-sdk-replay-page-limit-p6-long400-open005-20260920.json`, `rn-sdk-replay-page-limit-p6-long400-open005-bytes3-20260920.json` |
| Matched flame (fanout 50, c4) | **Pass** — 14.88 vs 16.68 bare-node CPU ms/step (−10.8%); cursor reads 0.79 vs 1.90/step | `rn-sdk-replay-page-limit-p6-flame500-20260920.json`, `rn-sdk-replay-page-limit-p6-flamecontrol-20260920.json`; profiles under `flames/profiles-2026-09-20T21-57-58.130Z-pprof-cpu-world-service-thread-1-0-2026-09-20T21-57-58-132Z.html` and `flames/profiles-2026-09-20T21-58-36.136Z-pprof-cpu-world-service-thread-1-0-2026-09-20T21-58-36-138Z.html` |
| Cancellation/replay | **Pass** — 97 targeted tests | `helpers.test.ts`, `quickjs-partial-preload.test.ts`, `wait-completion-replay.test.ts`, `abort-replay-ordering.test.ts` |
| Full SDK core suite | **Pass** — 2,584 passed, 3 expected failures, 1 skipped | command: `CI=true pnpm --filter @workflow/core test` |
| Older World / mixed version | **Pass** — beta.51 checkout, 4.9 workflows/s, 98 steps/s, 0 failures | `rn-sdk-replay-page-limit-p6-mixed-old-20260920.json` |
| Linux / Node 24 | **Pass** — 97 targeted tests | Docker `node:24-bookworm`, frozen lockfile |
| Linux / Node 22 | **Pass** — 97 targeted tests | Docker `node:22-bookworm`, frozen lockfile, container-local copy |

## Large-history details

The controlled run used `--workflow-rps 0.05 --steps 400 --workflow-mode fanout`, so one workflow completed fully rather than leaving a closed-loop request in teardown. The database contained 1,208 events for the completed run, with 215,685 bytes of serialized `event_data`; the 500-event request therefore represents three logical event-list pages. The normal run recorded SDK RSS/heap peaks of about 757/408 MB. With the optional benchmark-only `BENCH_WORLD_PROBE_BYTES=1` counter enabled, a matched diagnostic recorded exact event-list response bodies of 89,192,992 bytes for initial reads plus 70,437,021 bytes for cursor reads (159,630,013 bytes total across the repeated replay reads). The byte counter adds stream listeners, so its higher 1.02 GB/660 MB process peaks are not used as the memory claim. The World probe recorded 7.938 HTTP operations/step and 36.407 SQL statements/step. Both runs had no failed requests or runtime-read errors.

A separate 45-second c1 stress arm produced one `read ECONNRESET` in both candidate and 100-event control, after the World service reached the local macOS transport/CPU limit. It is retained as an environmental stress result, not counted as a candidate regression or a release-gate failure.

## Flame interpretation

The matched flame runs use the same fanout-50 workload and Node 24. The candidate completed 1,200 steps with 17,856.9 bare-node CPU ms; the control completed 1,000 steps with 16,682.1 ms. Normalizing by completed steps gives 14.88 vs 16.68 ms/step (about 11% lower). The hottest stack remains workflow VM execution; this change primarily removes replay-read amplification rather than changing user workflow execution.

## Review and rollback

- Files changed: `packages/core/src/runtime/helpers.ts` and three replay expectation tests.
- No World, wire-protocol, schema, or public API change.
- The cap is intentionally below the World maximum: histories between 501 and 1,000 events may take one additional page versus a 1,000-event request, in exchange for lower transient replay burst/memory and the lifecycle safety demonstrated above.
- Rollback is a one-line constant change, but restoring 1,000 requires a new lifecycle investigation.

## Publication

This branch is independent of the Priority 4 World capability stack. The source worktree contains benchmark-only untracked fixtures and `.pnpm-store/`; these are not part of the implementation commit. The final review gate is green.
