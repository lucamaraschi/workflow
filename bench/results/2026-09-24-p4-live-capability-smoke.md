# Priority 4 live capability smoke

Date: 2026-09-24
Status: **Passed locally; no branches pushed and no PR opened**

This is a local in-process HTTP smoke using the current Platformatic World
probe branch (`perf/world-create-batch-capability` @ `727a89f`) stacked on the
route branch (`perf/pr-world-create-batch` @ `ef76a22e`). The SDK contract
branch (`perf/world-create-batch-capability` @ `4fbb74d`) was checked separately
by the existing SDK suspension and Vercel wire gates; this smoke exercises the
World client contract over HTTP.

## Command and environment

PostgreSQL was already available on `127.0.0.1:5434`; the test creates and
cleans up two temporary applications and bindings. The Kubernetes TokenReview
API is a local test server, and the World client reads a temporary service
account token. Node is the bundled Node 24 runtime.

```sh
PATH=/Users/batman/.cache/codex-runtimes/codex-primary-runtime/dependencies/node/bin:$PATH \
DATABASE_URL=postgresql://wf:wf@localhost:5434/workflow \
/Users/batman/.cache/codex-runtimes/codex-primary-runtime/dependencies/node/bin/node \
/Users/batman/src/platformatic/workflowsdk-in-watt/bench/.p4-live-capability-smoke.mjs
```

## Result

```json
{
  "auth": {
    "unauthenticated": 401,
    "validApp": 200,
    "crossAppMismatch": 403
  },
  "startup": {
    "beforeProbe": "absent",
    "afterStart": true,
    "requestCount": 1
  },
  "failClosed": {
    "old404": false,
    "oldSpecVersion": false,
    "malformedSpecVersion": false
  },
  "tokenReviewRequests": 1
}
```

The valid client path authenticated with the app binding, resolved the
app-scoped capability endpoint, and advertised `eventsCreateBatch` only after
`start()` awaited the probe. A token bound to app A was rejected for app B.
Before startup the capability remained absent. A 404 old service, an old
capability protocol (`specVersion: 5`), and a malformed protocol value all
left the capability absent after startup. The service account token was sent
as `Authorization: Bearer app-token-a` to the TokenReview API.

## Scope and limitation

This proves the authenticated HTTP handshake, app scoping, startup ordering,
and fail-closed cross-version behavior against the current local branches. It
does not claim a deployed Kubernetes control-plane or production identity
provider test; those remain deployment/CI validation gates.
