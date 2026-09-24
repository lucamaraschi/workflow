# World candidate-head evidence refresh

Date: 2026-09-24
Scope: Priority 4 capability probe and Priority 5 terminal wait/notify
Status: **Candidate-head refresh complete; deployment checks remain explicit**

This refresh validates the latest World candidate heads after the reviewer-
driven hardening pass. Priority 3 is intentionally excluded.

## Priority 4 capability probe

| Stack layer | Worktree | Branch | Candidate head |
| --- | --- | --- | --- |
| Route/client | `/Users/batman/src/platformatic/platformatic-world-create-batch-pr` | `perf/pr-world-create-batch` | `ef76a22eb8b47bb91e55779edce86472ea4bb19d` |
| SDK contract | `/Users/batman/src/platformatic/workflow-sdk-create-batch-capability` | `perf/world-create-batch-capability` | `4fbb74d47e3f12208e4b709e1cb7f711ed8d2e06` |
| Probe/advertisement | `/Users/batman/src/platformatic/platformatic-world-create-batch-capability` | `perf/world-create-batch-capability` | `727a89fb110c41e6f4f4c4564a32c883ab164f45` |

Command:

```sh
export PATH=/Users/batman/.cache/codex-runtimes/codex-primary-runtime/dependencies/node/bin:$PATH
cd /Users/batman/src/platformatic/platformatic-world-create-batch-capability
pnpm -C packages/world test
```

Result: **52 passed, 0 failed, 0 cancelled** on Node `v24.19.0` in
`254.711625ms`. The suite covers async readiness, capability absence, 404,
old protocol, malformed documents, no per-batch probe, token rotation, and
authorization behavior. The optional integration suite reported
`Workflow service not running at http://localhost:3042 — skipping integration tests`;
therefore a deployed cross-version HTTP probe remains a release/deployment
verification item.

Additional checks:

```sh
git -C /Users/batman/src/platformatic/platformatic-world-create-batch-capability diff --check
git -C /Users/batman/src/platformatic/platformatic-world-create-batch-capability status --short --branch
```

Both checks passed; the worktree is clean.

## Priority 5 terminal wait/notify

| Worktree | Branch | Candidate head |
| --- | --- | --- |
| `/Users/batman/src/platformatic/platformatic-world-terminal-wait-notify-pr` | `perf/pr-world-terminal-wait-notify` | `18b41e5f108f377361fd2b4c851ada9337eb3b79` |

Command:

```sh
export PATH=/Users/batman/.cache/codex-runtimes/codex-primary-runtime/dependencies/node/bin:$PATH
cd /Users/batman/src/platformatic/platformatic-world-terminal-wait-notify-pr
pnpm -C packages/workflow test -- --test-name-pattern='wait|run'
```

Result: **160 passed, 0 failed, 0 cancelled** on Node `v24.19.0` in
`12001.017375ms`. This includes the missing-run `/wait` `404` case and
cross-tenant `/wait` isolation. The worktree is clean and `git diff --check`
passes.

## Evidence boundary and remaining deployment checks

The existing matched 400-RPS JSON measurements and raw profiles remain the
performance evidence; this refresh adds candidate-head correctness evidence
only. Before opening the P4/P5 PRs, the deployment checklist should still
explicitly document:

- P4: the published SDK type range and an authenticated, app-scoped
  cross-version `/capabilities` smoke against a live World service.
- P5: trigger migration/privilege requirements, PostgreSQL connection budget,
  response/error semantics, app/run scoping, and listener diagnostics.
