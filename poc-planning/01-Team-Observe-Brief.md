# Team 1 Brief — "Observe" (Discovery, Topology, Detection)

**Reference:** Full architecture doc `Observability-POC-SPARK-info.md`, sections 2 (Functional Requirements), 7 (Collector Architecture), 9 (Topology Discovery), 11 (Anomaly Detection).
**Team size:** ~2-3 engineers.
**Language:** Go, controller-runtime / client-go.
**You own:** everything upstream of "we noticed something is wrong."
**You produce:** `Incident` Custom Resources.
**You depend on:** nothing (you are the root of the pipeline).

---

## 1. Mission

Watch a Kubernetes cluster, build a minimal topology of what depends on what, detect a small set of real failure conditions, and turn each detected failure into a structured `Incident` Custom Resource that downstream teams can consume.

## 2. In scope

- Watching core Kubernetes objects via informers: `Pod`, `Deployment`, `ReplicaSet`, `Event`.
- A minimal topology map: ownerReference chains (Deployment → ReplicaSet → Pod) and Service → Pod endpoint mapping.
- 3-5 threshold-based anomaly detectors (see §5).
- Defining and creating the `Incident` CRD (joint with Team 2 in week 1 — do not change the schema unilaterally after week 1 without telling Teams 2 and 3).
- Enriching incidents with basic evidence (namespace, workload name/kind, node, container, recent events).
- Unit tests for detector logic; `envtest`-based controller tests.

## 3. Out of scope

- Logs and traces (no log tailing, no OTLP trace ingestion).
- Anything beyond the 3-5 listed detectors — resist scope creep here, this is the easiest team to over-build.
- Multi-namespace RBAC hardening — use a broad ClusterRole for the POC, note this as a known gap.
- Persisting topology anywhere durable — an in-memory graph rebuilt from informer caches is enough.

## 4. The `Incident` CRD (draft — finalize with Team 2 in week 1)

```yaml
apiVersion: selfheal.io/v1alpha1
kind: Incident
metadata:
  name: incident-<uuid>
  namespace: <affected-namespace>
spec:
  severity: Critical | Warning | Info
  detectedAt: <RFC3339 timestamp>
  detector: CrashLoopBackOff | OOMKilled | RestartSpike | Unready
  affected:
    kind: Deployment
    name: checkout-service
    namespace: payments
  evidence:
    - type: PodStatus
      podName: checkout-service-abc123
      reason: CrashLoopBackOff
      message: "back-off restarting failed container"
    - type: KubernetesEvent
      reason: BackOff
      message: "..."
status:
  phase: Observed | Correlated | Analyzing | AwaitingApproval | Remediating | Verifying | Resolved | Rollback | Escalated | Suppressed
  conditions: []
```

```go
type IncidentSpec struct {
    Severity   string       `json:"severity"`
    DetectedAt metav1.Time  `json:"detectedAt"`
    Detector   string       `json:"detector"`
    Affected   ObjectRef    `json:"affected"`
    Evidence   []Evidence   `json:"evidence"`
}

type IncidentStatus struct {
    Phase      string             `json:"phase"`
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

You own writing `spec` and the initial `status.phase = Observed`. You do **not** write later phases (`Correlated` onward) — that's Team 2/3's job as the CR moves through the lifecycle.

## 5. The 3-5 detectors to build (in priority order)

1. **CrashLoopBackOff** — pod container status `waiting.reason == CrashLoopBackOff`.
2. **OOMKilled** — pod container `lastState.terminated.reason == OOMKilled`.
3. **RestartSpike** — container restart count increases by N within a rolling time window.
4. **Unready too long** — pod `Ready` condition false for longer than N minutes while `Running`.
5. *(stretch)* **Bad rollout** — new ReplicaSet's pods failing while old ReplicaSet was healthy (signals a bad deploy, useful for the "rollback" demo scenario).

Build #1 and #2 first — they're the two demo scenarios everyone will rely on for the week 5 integration test.

## 6. Week-by-week

| Week | Deliverable |
|---|---|
| 1 | Kind cluster setup + "broken app" demo manifests (bad image, low memory limit) shared with all teams. `Incident` CRD finalized with Team 2. Repo scaffold. |
| 2 | Informers running, topology graph builder working, `#healthz`/`#readyz` skeleton. |
| 3 | Detectors #1 and #2 implemented and creating real `Incident` CRs in the kind cluster. |
| 4 | Detectors #3-4, evidence enrichment, unit + envtest coverage. |
| 5 | Hand off to Team 2 for real integration; fix issues found. |
| 6 | Bug-bash, demo scenario #5 (bad rollout) if time allows. |
| 7 | Polish, docs, rehearse your part of the demo. |

## 7. Definition of done

- `kubectl apply -f demo/broken-crashloop.yaml` in the kind cluster produces a matching `Incident` CR within ~30 seconds, with `status.phase = Observed` and evidence populated.
- Same for the OOM scenario.
- Unit tests cover each detector's true-positive and true-negative cases.
- README explains how to run the controller locally against kind.

## 8. Handoff interface to Team 2

Team 2 only needs to know: **watch `Incident` CRs where `status.phase == Observed`**. Give them 2-3 sample `Incident` YAML files in week 1 so they can build against your contract before your controller exists.
