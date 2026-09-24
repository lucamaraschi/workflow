# Priority 4 capability-rollout validation

Date: 2026-09-20  
Status: **Prepared locally; no branch pushed and no PR opened**

Rebase record (2026-09-23): the route branch is now `ef76a22e` and the probe
branch is now `9b95ad3`; both are rebased on World `origin/main`
`7b9629822` and their introduced commits carry DCO sign-off. The SDK contract
is `4fbb74d` on SDK `upstream/main` `69e80c6f7`, with its changeset and DCO
sign-offs complete. The validation artifacts below are retained from the
pre-rebase gate runs.

## Branch stack

| Layer | Repository | Branch / commit | Role |
| --- | --- | --- | --- |
| Route/client | Platformatic World | `perf/pr-world-create-batch` @ `ef76a22e` | Adds the additive batch endpoint and client serializer. |
| SDK contract | Workflow SDK | `perf/world-create-batch-capability` @ `4fbb74d` | Adds typed `WorldCapabilities.eventsCreateBatch` and the fail-closed core gate. |
| Probe/advertisement | Platformatic World | `perf/world-create-batch-capability` @ `9b95ad3` | Stacked on the route; probes the remote service once and advertises only after success. |
| Audit | Campaign worktree | `perf/pr4-capability-rollout-audit` @ `f7c945b` | Compatibility matrix and merge/deploy order. |

The identical branch spelling is in two different repositories; always qualify
it with the repository in review links and push commands.

## Contract and lifecycle evidence

- SDK suspension handler: **78/78**.
- SDK World Vercel batch wire tests: **14/14**.
- Platformatic World package: **50/50**.
- Platformatic Workflow app capability tests: **10/10**.
- Platformatic Workflow full suite: **158/158**.
- Platformatic Workflow Fastify suite: **8/8**.
- Build/typecheck/lint and `git diff --check`: pass on each prepared branch.
- Probe test: a World instance makes exactly one `/capabilities` request,
  remains false before `refreshCapabilities()`/`start()`, and never probes per
  event batch. Absent, malformed, transport-error, and 404 responses remain
  false.
- Lifecycle check: Platformatic `workflow-fastify` awaits `world.start()` in
  its `onReady` hook (`packages/workflow-fastify/src/index.ts`), so the normal
  integration completes the probe before serving workflow traffic. Integrations
  that do not call `start()` retain safe single-event behavior until they call
  `refreshCapabilities()` explicitly.

## Performance and flame context

The handshake adds no request to the event/batch hot path. Its only network
operation is the cached startup probe. The performance proof remains the
matched latest-main route A/B:

- Low load: **+28.95% steps/s**, runtime p50 **−19.27%**, World HTTP/step
  **−19.26%**, SQL/step **−14.91%**, DB active **−25.25%**.
- 400-RPS tail: **+7.55% steps/s**, runtime p50 **−5.21%**, ping p99
  **−45.81%**, World HTTP/step **−11.75%**, SQL/step **−10.14%**.
- All repetitions completed with zero failures and zero backlog slope.

Raw measurements:

- [`w1-pr4-latest-control-20260920.json`](w1-pr4-latest-control-20260920.json)
- [`w1-pr4-latest-candidate-20260920.json`](w1-pr4-latest-candidate-20260920.json)
- [`w1-pr4-tail-control-20260920.json`](w1-pr4-tail-control-20260920.json)
- [`w1-pr4-tail-candidate-20260920.json`](w1-pr4-tail-candidate-20260920.json)
- [`world-create-batch-default-gates`](2026-09-20-world-create-batch-default-gates.md)

Representative matched flames remain linked from the route PR body. They show
the candidate reduction in World HTTP/transaction work without a new dominant
serializer or service hotspot; bare-node CPU remains dominated by workflow VM
trace/context setup and GC. The SDK capability PR itself is a control-plane
gate, so it should not be credited with an additional steady-state speedup.

## Compatibility decision

The route and SDK contract are additive and safe to land independently. The
World probe is safe against old services because 404/invalid responses fail
closed. Full independent rollability still requires publishing the SDK field,
updating the Platformatic World type range in the release wiring, and verifying
that every production integration awaits `start()` or explicitly refreshes
capabilities. Until then, describe Priority 4 as matched-version ready, not
universally default-ready.
