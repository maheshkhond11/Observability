# Team 3 Brief — "Act & Show" (Action, Verification, Dashboard/API)

**Reference:** Full architecture doc `Observability-POC-SPARK-info.md`, sections 15 (Action Engine), 16 (Verification Engine), 22 (REST API), 23 (Dashboard).
**Team size:** ~2-3 engineers.
**Language:** Go (Gin/Echo) for API/controller, any lightweight frontend for the dashboard.
**You own:** actually fixing the problem, checking it worked, and showing the whole loop live — including the cluster's general observability view, not just the incident timeline. This is the team the demo hinges on visually — budget real time for the dashboard, not just the backend.
**You produce:** executed actions, verified outcomes, and the live UI.
**You depend on:** `RemediationPlan` CRs from Team 2 (build against hand-written sample CRs until their controller is ready), and Prometheus from Team 1 (for the Cluster Health view, §7).

---

## 1. Mission

Watch approved `RemediationPlan` CRs, execute exactly one of three safe actions against the cluster, verify the workload actually recovered, and surface the entire Observe→Decide→Act→Verify timeline on a dashboard in real time.

## 2. In scope

- A controller watching `RemediationPlan` CRs where `status.phase` is `Pending` or `Remediating`.
- Three action types only: `RestartPod` (delete the pod, let the controller recreate it), `RollbackDeployment` (revert to the previous ReplicaSet revision), `ScaleDeployment` (change replica count).
- Basic guardrails: a dry-run flag, a max-blast-radius check (e.g. never act on more than N pods at once), a structured audit log line per action.
- A verification loop: after acting, poll workload health (pod Ready condition, restart count stabilized) for up to M minutes, then set `status.phase` to `Resolved`, `Rollback`, or `Escalated`.
- A REST API exposing `GET /incidents`, `GET /plans`, `GET /topology` — read directly from the Kubernetes API via client-go. No Postgres/ClickHouse for this POC.
- A minimal dashboard: incident list, incident detail with the full lifecycle timeline, a **Cluster Health view** sourced from Team 1's Prometheus, and a live-updating view during the demo.
- The failure-injection demo script (`kubectl apply` broken workloads) and a rehearsed demo flow.

## 3. Out of scope

- Any action beyond the 3 listed — no node/network/storage healing.
- Real authentication (OIDC) — an unauthenticated local API is fine for a POC demo, note it as a known gap.
- Persisting audit logs to a database — structured stdout logs plus the CR's own status/conditions are enough.
- A production-grade frontend framework debate — a single-page React app (or even server-rendered HTML with a bit of JS for polling) is enough. Optimize for "looks good live," not architecture purity.

## 4. Action execution contract

You read `RemediationPlan.spec.action`:

```yaml
action:
  type: RestartPod | RollbackDeployment | ScaleDeployment
  target:
    kind: Deployment
    name: checkout-service
    namespace: payments
  params: {}   # e.g. {replicas: 3} for ScaleDeployment
```

Execution rules:
- `RestartPod`: find the pod(s) under `target`, delete them (the Deployment/ReplicaSet controller recreates them). Cap at N pods per action (blast-radius guardrail).
- `RollbackDeployment`: patch the Deployment to the previous revision (equivalent of `kubectl rollout undo`).
- `ScaleDeployment`: patch `spec.replicas` using `params.replicas`.

After executing, immediately set `status.phase = Verifying` and record `status.actionExecutedAt`.

## 5. Verification logic

Poll every few seconds for up to M minutes (configurable, default ~2 min):
- **Resolved**: target pods reach `Ready=true` and restart count has stopped increasing.
- **Rollback**: verification fails and the action was `RollbackDeployment` or a retry is unsafe → mark `Rollback` and stop (a human investigates; do not auto-retry indefinitely for the POC).
- **Escalated**: verification times out without recovery → mark `Escalated`, surface prominently on the dashboard (this is a valid and expected demo outcome for a genuinely hard scenario, not a bug).

## 6. REST API surface (minimal)

```
GET /api/v1/incidents            -> list Incident CRs across watched namespaces
GET /api/v1/incidents/{name}     -> one Incident with its linked RemediationPlan
GET /api/v1/plans                -> list RemediationPlan CRs
GET /api/v1/topology              -> current in-memory topology (from Team 1, or re-derived here if simpler)
GET /api/v1/metrics/summary       -> a few pre-picked PromQL queries proxied from Prometheus (cluster CPU/memory, per-namespace memory saturation) for the Cluster Health view
GET /healthz, /readyz, /metrics
```

Read-only is enough — all mutations happen via `kubectl` against CRs during the demo, not through the API.

## 7. Dashboard (the demo centerpiece)

Minimum viable screens:
1. **Incident list** — table of active/recent incidents with severity and current phase.
2. **Incident detail** — the lifecycle timeline (Observed → Correlated → Analyzing → Remediating → Verifying → Resolved/Escalated) rendered as a horizontal stepper, updating live (poll every 2-3s is fine, no need for websockets).
3. **Cluster Health view (new)** — a small set of live charts (CPU/memory per namespace, per-pod memory-vs-limit) sourced from `/api/v1/metrics/summary`. This is the "observability" half the team asked for — it's useful on its own, with zero incidents happening, which is exactly the point: the platform is a real monitoring tool first, and a self-healer on top of it.
4. *(stretch)* **Topology view** — a simple graph of the affected workload and its dependencies.

Prioritize #1, #2, and #3 — together they tell the whole story live in the demo: here's what's happening in the cluster right now (#3), here's something going wrong (#1), here's the full resolution (#2).

## 8. Week-by-week

| Week | Deliverable |
|---|---|
| 1 | Finalize CRDs with Teams 1 and 2. Build action executor against hand-written sample `RemediationPlan` CRs. |
| 2 | RestartPod + ScaleDeployment implemented and tested against kind. |
| 3 | RollbackDeployment implemented. Verification loop working for all three. |
| 4 | REST API skeleton + incident list screen. |
| 5 | Incident detail/timeline screen; real integration with Team 2's live plans. Cluster Health view wired to Team 1's Prometheus. |
| 6 | Bug-bash, blast-radius guardrails, audit logging polish. |
| 7 | Demo script finalized, full dry-run rehearsal with all 3 teams. |

## 9. Definition of done

- Given a hand-crafted `RemediationPlan` with `action.type: RestartPod`, the target pod is deleted, recreated, and the plan reaches `status.phase: Resolved` within the verification window.
- The dashboard shows an incident's full timeline updating live without a page refresh.
- A deliberately unrecoverable scenario correctly reaches `Escalated` rather than hanging or crashing the controller.
- Blast-radius guardrail demonstrably blocks/limits an action targeting more than N pods.
- The Cluster Health view shows real, live-updating metrics with no incidents in progress — proving the platform stands on its own as an observability tool, not only as a healing loop.

## 10. The full demo script (own this, coordinate with both other teams)

1. Open on the **Cluster Health view** with nothing broken — establishes this is a real observability tool first, not just a healing gimmick.
2. Apply a broken workload (bad image or low memory limit) to the kind cluster.
3. Dashboard shows: Incident appears (Team 1) → correlated → root cause + plan shown (Team 2) → action executed → verifying → Resolved (your team).
4. Second scenario: a bad rollout that requires approval — show the plan waiting in `AwaitingApproval`, a teammate approves via `kubectl patch`, then the rollback executes and resolves.
5. Third scenario (the "why not just Kubernetes" answer): apply a workload that slowly climbs toward its memory limit. Show the `MemoryTrendWarning` Incident appearing on the Cluster Health view **before** the pod ever crashes — side by side with the earlier `OOMKilled` scenario, which only fired after the crash. This contrast is the single strongest live moment in the demo.
6. Total demo time target: under 6-7 minutes end-to-end for all three scenarios combined.
