# SDK candidate-head evidence refresh

Date: 2026-09-24

This refresh validates the current rebased SDK heads before the evidence is
published. It does not change or include P3.

## P1 source sharding

- Candidate: `perf/sdk-source-sharding-gates` at
  `16fca55d710afc4f41befb16b436fa231372cc8f`
- Control: `perf/inline-step-default-16` at
  `be49d8962c1b61a62bd6837bb2d4204bb328629a`
- World: `platformatic-world-audit-20260918` at
  `273d8ba43073bfb4c860a33c2447c9e1d1a0d63e`

Focused current-head smoke:

```text
pnpm exec vitest run packages/core/src/runtime/workflow-code.test.ts packages/builders/src/workflow-bundle-sharding.test.ts
2 files, 9 tests passed
```

Fresh unprofiled World A/B (same World build, PostgreSQL, T0, fanout, 50
steps/workflow, 1 KiB payload, concurrency 4, 20 ms polling, 200 ping/s,
10-second window, two repetitions per arm):

| Metric | Control | Candidate | Change |
| --- | ---: | ---: | ---: |
| Workflow runtime p50 | 2,808.5 ms | 1,581.5 ms | −43.7% |
| Steps/s | 62.5 | 120 | +92.0% |
| Ping p99 | 1,267.5 ms | 326 ms | −74.3% |
| Failed requests | 0 | 0 | no regression |

Artifacts:

- [P1 control run](rn-source-sharding-candidate-head-control-20260924.json)
- [P1 candidate run](rn-source-sharding-candidate-head-20260924.json)

The benchmark fixture routes/workflow were staged only for the build, then
removed; the P1 worktree is clean. This run is unprofiled and shorter than the
retained three-repetition 20-second gate, so it refreshes current-head A/B
evidence but does not replace the original pprof/flame attribution.

## P2 serialization/logger

- Candidate: `perf/pr-sdk-serialization-logger` at
  `cdb289393caeb44104f4b649319b81cb5f2a44a6`
- Control: `perf/inline-step-default-16` at
  `be49d8962c1b61a62bd6837bb2d4204bb328629a`
- Runtime: Node `v24.19.0`

Focused current-head tests:

```text
pnpm exec vitest run packages/core/src/logger.test.ts packages/core/src/writable-stream.test.ts packages/core/src/serialization.test.ts packages/core/src/serialization/serialization.test.ts packages/core/src/serialization/compat.test.ts
5 files, 444 tests passed
```

The refreshed value-shape matrix passed all round-trip, byte-size, and
malformed-payload assertions. Candidate/control bytes remained equivalent.
The refreshed logger matrix emitted zero lines for disabled/unrelated patterns
and one line for the enabled pattern in both arms:

| DEBUG pattern | Candidate median | Control median | Candidate speedup |
| --- | ---: | ---: | ---: |
| empty | 0.0901 µs | 0.1229 µs | 1.36× |
| unrelated:* | 0.1095 µs | 0.3865 µs | 3.53× |
| workflow:runtime:debug | 1.3210 µs | 1.4941 µs | 1.13× |

Artifacts:

- [refreshed value matrix](rn-serialization-logging-value-matrix-20260919.json)
- [refreshed logger matrix](rn-serialization-logging-logger-matrix-20260919.json)

The integrated P2 World A/B and matched pprof artifacts remain the retained
pre-rebase evidence; this refresh confirms the candidate-head behavior and
contract without claiming a new end-to-end profile.

## Publication status

The refreshed artifacts are included in this evidence bundle. PR bodies should
pin the immutable evidence commit containing this report and its raw artifacts.
