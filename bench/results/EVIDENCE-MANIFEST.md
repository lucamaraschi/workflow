# Workflow SDK performance PR evidence

This directory is the reviewer-facing evidence package for the performance PR
queue. It is kept separate from the Workflow SDK and Platformatic World product
branches so benchmark output does not enter their implementation diffs.

The package is published from the public SDK fork `lucamaraschi/workflow` on
branch `evidence/performance-prs-2026-09`. PR descriptions link to this
manifest and the reports below using the immutable evidence commit SHA for
that branch.

## Gate reports

- [Priority 1 — source-sharded VM bundles](2026-09-19-source-sharding-gates.md)
- [Priority 2 — serializer/logger](2026-09-20-sdk-serialization-logger-default-gates.md)
- [Priority 3 — SQL RETURNING](2026-09-20-world-step-returning-gates.md)
- [Priority 4 — World batch route](2026-09-20-world-create-batch-default-gates.md)
- [Priority 4 — capability rollout](2026-09-20-world-create-batch-capability-rollout.md)
- [Priority 5 — terminal wait notification](2026-09-20-world-terminal-wait-notify-default-gates.md)
- [Priority 6 — replay page limit](2026-09-20-sdk-replay-page-limit-default-gates.md)
- [Priority 7 — inline-step default](2026-09-20-sdk-inline-step-limit-default-gates.md)

## Candidate-head refresh notes

- [SDK P1/P2 refresh](2026-09-24-sdk-candidate-head-refresh.md)
- [World P4/P5 refresh](2026-09-24-world-capability-wait-refresh.md)
- [P4 authenticated capability smoke](2026-09-24-p4-live-capability-smoke.md)

Priority 3 remains listed for historical provenance, but it is excluded from
the current hardening and publication refresh.

Each report links to the retained machine-readable benchmark results and the
matched control/candidate flame artifacts used for its decision. The reports
are the authoritative interpretation; raw JSON and rendered flame files are
included for independent inspection.

## Publication rule

PR bodies must use absolute links pinned to the evidence commit, for example:

```text
https://github.com/lucamaraschi/workflow/blob/<evidence-sha>/bench/results/EVIDENCE-MANIFEST.md
```

Do not use workspace-relative `../bench/results/...` links in GitHub PRs.
