# Team 2 Brief — "Decide" (Correlation, Root Cause, Policy, Decision)

**Reference:** Full architecture doc `Observability-POC-SPARK-info.md`, sections 10 (Correlation Engine), 12 (Root Cause Analysis), 13 (AI Engine — stretch only), 14 (Decision Engine).
**Team size:** ~2-3 engineers.
**Language:** Go, controller-runtime / client-go.
**You own:** turning "something is wrong" into "here's what we should do about it, and whether we're allowed to."
**You produce:** `RemediationPlan` Custom Resources.
**You depend on:** `Incident` CRs from Team 1 (build against hand-written sample CRs until their controller is ready).

---

## 1. Mission

Watch `Incident` CRs, group related ones together, rank the most likely root cause using simple deterministic rules (no ML), check a declarative policy for what's allowed, and produce a `RemediationPlan` CR describing exactly one safe action to take.

## 2. In scope

- A controller watching `Incident` CRs where `status.phase == Observed`.
- Correlation: group Incidents referencing the same workload within a rolling time window (e.g. 2 minutes) into one working unit — avoid creating 5 separate plans for 5 crashing pods of the same Deployment.
- Rule-based RCA: a small decision table mapping `detector` + evidence → a root-cause label (see §5). No machine learning.
- Policy engine reading a `SelfHealingPolicy` CR: which actions are allowed/denied per namespace, and which require human approval.
- Producing `RemediationPlan` CRs with the chosen action, risk tier, and approval requirement.
- An approval-stub: when `requiresApproval: true`, the plan sits in `AwaitingApproval` until a human patches its status (simulates an approval UI without building one).

## 3. Out of scope

- Any real ML/statistical anomaly scoring — plain if/else or a lookup table is correct for this POC.
- OPA/Rego — a policy CR read into a plain Go struct and matched with simple logic is enough.
- A real approval UI — `kubectl patch` is the approval mechanism for the demo.
- The stretch AI-summary feature (§6) is optional and must never gate or choose the action — it only writes a human-readable string field.

## 4. The `RemediationPlan` CRD (draft — finalize with Team 1 and Team 3 in week 1)

```yaml
apiVersion: selfheal.io/v1alpha1
kind: RemediationPlan
metadata:
  name: plan-<uuid>
  namespace: payments
spec:
  incidentRef:
    name: incident-<uuid>
    namespace: payments
  rootCause:
    type: OOMKilled | BadRollout | CrashLoop | Unknown
    confidence: 0.0-1.0
  action:
    type: RestartPod | RollbackDeployment | ScaleDeployment
    target:
      kind: Deployment
      name: checkout-service
      namespace: payments
    params: {}
  riskTier: Safe | Moderate | HighRisk
  requiresApproval: true | false
status:
  phase: Pending | AwaitingApproval | Remediating | Verifying | Resolved | Rollback | Escalated
  approvedBy: ""
  approvedAt: null
```

```go
type RemediationPlanSpec struct {
    IncidentRef      corev1.ObjectReference `json:"incidentRef"`
    RootCause        RootCause              `json:"rootCause"`
    Action           PlannedAction          `json:"action"`
    RiskTier         string                 `json:"riskTier"`
    RequiresApproval bool                   `json:"requiresApproval"`
}
```

And the policy CRD it reads:

```yaml
apiVersion: selfheal.io/v1alpha1
kind: SelfHealingPolicy
metadata:
  name: payments-default
  namespace: payments
spec:
  mode: AutoSafe
  actions:
    allowed: [RestartPod, RollbackDeployment, ScaleDeployment]
    denied: []
  approvalRequiredFor: [RollbackDeployment]
```

(This mirrors the example already in the source architecture doc, §38.5 — reuse it rather than inventing a new shape.)

## 5. RCA decision table (v1 — extend if time allows)

| Detector (from Incident) | Root cause label | Chosen action |
|---|---|---|
| OOMKilled | `OOMKilled` | RestartPod (stretch: ScaleDeployment with higher memory request — needs a plan param) |
| CrashLoopBackOff | `CrashLoop` | RestartPod |
| CrashLoopBackOff + recent rollout (from Team 1's stretch detector) | `BadRollout` | RollbackDeployment |
| RestartSpike | `CrashLoop` | RestartPod |
| Unready too long | `Unknown` | RestartPod (lowest-risk default) |

## 6. Stretch: AI summary (optional, do last)

If time allows, add a single call to an LLM that reads the Incident evidence and root cause and writes a one-paragraph human-readable summary into `RemediationPlan.status.summary`. This is purely cosmetic for the dashboard — it must never influence `action` or `riskTier`. This demonstrates the source doc's core safety principle (§13, §38.2): AI assists, it never decides or executes.

## 7. Week-by-week

| Week | Deliverable |
|---|---|
| 1 | `RemediationPlan` + `SelfHealingPolicy` CRDs finalized with Teams 1 and 3. Build against hand-written sample `Incident` CRs. |
| 2 | Correlation logic + RCA decision table for detectors #1-2 (OOMKilled, CrashLoop). |
| 3 | Policy engine reading `SelfHealingPolicy`, producing `RemediationPlan` with correct riskTier/approval. |
| 4 | Approval-stub flow; RCA table extended to detectors #3-4. |
| 5 | Real integration with Team 1's live `Incident` CRs; hand off to Team 3. |
| 6 | Bug-bash, stretch AI summary if time allows. |
| 7 | Polish, docs, rehearse your part of the demo. |

## 8. Definition of done

- Given a hand-crafted `Incident` CR with `detector: OOMKilled`, your controller produces a `RemediationPlan` with `action.type: RestartPod`, `riskTier: Safe`, `requiresApproval: false` within a few seconds.
- Given a `RollbackDeployment`-triggering scenario, the plan correctly sits in `AwaitingApproval` until manually patched.
- Two Incidents for the same Deployment within the correlation window produce exactly one `RemediationPlan`, not two.
- Unit tests cover the RCA table and policy matching logic.

## 9. Handoff interface to Team 3

Team 3 only needs to know: **watch `RemediationPlan` CRs where `status.phase` is `Pending` or `Remediating`** (i.e., approval has cleared or wasn't required). Give them 2-3 sample `RemediationPlan` YAML files in week 1.
