# Team 1 Brief — "Observe" (Discovery, Topology, Detection)

**Reference:** Full architecture doc `Observability-POC-SPARK-info.md`, sections 2 (Functional Requirements), 7 (Collector Architecture), 9 (Topology Discovery), 11 (Anomaly Detection).
**Team size:** ~2-3 engineers.
**Language:** Go, controller-runtime / client-go.
**You own:** everything upstream of "we noticed something is wrong" — including the cluster's basic observability layer (metrics), not just detection.
**You produce:** `Incident` Custom Resources, plus a running metrics pipeline (OpenTelemetry Collector → Prometheus) that Team 3's dashboard also reads from.
**You depend on:** nothing (you are the root of the pipeline).

> **Update (post team review):** the team asked for two things this brief now covers: (1) why this platform adds value beyond Kubernetes' own restart/replace healing, and (2) a real observability layer, not just a self-healing loop. See §5a and §6a below — this is the direct answer to both, and it's the headline addition to this team's scope.

---

## 1. Mission

Watch a Kubernetes cluster, build a minimal topology of what depends on what, detect a small set of real failure conditions, and turn each detected failure into a structured `Incident` Custom Resource that downstream teams can consume.

## 2. In scope

- Watching core Kubernetes objects via informers: `Pod`, `Deployment`, `ReplicaSet`, `Event`.
- A minimal topology map: ownerReference chains (Deployment → ReplicaSet → Pod) and Service → Pod endpoint mapping.
- 4-6 anomaly detectors, both event-based and metrics-based (see §5 and §5a).
- Standing up the observability layer: an **OpenTelemetry Collector** (official Helm chart, deployed as a DaemonSet) plus a lightweight **Prometheus** (via the community `kube-prometheus-stack` chart as a subchart — do not hand-build a metrics store).
- Defining and creating the `Incident` CRD (joint with Team 2 in week 1 — do not change the schema unilaterally after week 1 without telling Teams 2 and 3).
- Enriching incidents with basic evidence (namespace, workload name/kind, node, container, recent events).
- Unit tests for detector logic; `envtest`-based controller tests.

## 3. Out of scope

- Logs and traces (OTel Collector is configured for **metrics only** in this POC — no log tailing, no OTLP trace ingestion).
- Anything beyond the 4-6 listed detectors — resist scope creep here, this is the easiest team to over-build.
- Application-level custom metrics / app instrumentation — stick to metrics Kubernetes already exposes for free (kubelet/cAdvisor: CPU, memory, restart counts). No changes required to demo apps.
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
  detector: CrashLoopBackOff | OOMKilled | RestartSpike | Unready | MemoryTrendWarning
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

## 5. The event-based detectors to build (in priority order)

These all work the same way: watch the Kubernetes API, react to a status that already changed. This is the **reactive** half of detection.

1. **CrashLoopBackOff** — pod container status `waiting.reason == CrashLoopBackOff`.
2. **OOMKilled** — pod container `lastState.terminated.reason == OOMKilled`.
3. **RestartSpike** — container restart count increases by N within a rolling time window.
4. **Unready too long** — pod `Ready` condition false for longer than N minutes while `Running`.
5. *(stretch)* **Bad rollout** — new ReplicaSet's pods failing while old ReplicaSet was healthy (signals a bad deploy, useful for the "rollback" demo scenario).

Build #1 and #2 first — they're the two demo scenarios everyone will rely on for the week 5 integration test.

## 5a. The metrics-based detector — Memory Saturation Trend (new)

This one works differently on purpose, and it's the direct answer to "why do we need this if Kubernetes already self-heals": it's **predictive**, not reactive. Instead of watching the Kubernetes API for a status change, it periodically runs a query against Prometheus (every ~30s) asking: *is this pod's memory usage above ~85% of its limit, and has it been climbing for the last N minutes?*

If yes, raise an `Incident` with `detector: MemoryTrendWarning` and `severity: Warning` — **before** the container actually gets OOMKilled, while Kubernetes still considers the pod perfectly healthy. This is the one detector in the whole platform that catches a problem Kubernetes structurally cannot see coming, because Kubernetes only reacts after the container process has already died.

Implementation notes:
- Data source: `container_memory_working_set_bytes` vs the container's configured memory limit — both already exposed by kubelet/cAdvisor, scraped automatically once the OTel Collector + Prometheus are running. No app changes needed.
- Mechanism: a simple ticker (`time.NewTicker`) inside the controller, not an informer watch — this is polling a metrics store, not watching the API server. Document this distinction clearly in code comments; it's a genuinely different mechanism from detectors #1-5 and worth being explicit about when explaining the system.
- PromQL sketch: `container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.85`, sustained via Prometheus's own range/`for` semantics or re-checked on each poll.

This detector is what should sit next to an `OOMKilled` incident in the demo — same underlying problem, but caught early instead of after the crash.

## 6. Week-by-week

| Week | Deliverable |
|---|---|
| 1 | Kind cluster setup + "broken app" demo manifests (bad image, low memory limit) shared with all teams. `Incident` CRD finalized with Team 2. Repo scaffold. Install OTel Collector + `kube-prometheus-stack` subchart. |
| 2 | Informers running, topology graph builder working, `#healthz`/`#readyz` skeleton. Confirm Prometheus is scraping cluster metrics. |
| 3 | Detectors #1 and #2 implemented and creating real `Incident` CRs in the kind cluster. |
| 4 | Detectors #3-4, evidence enrichment, unit + envtest coverage. Start Memory Saturation Trend detector (§5a). |
| 5 | Hand off to Team 2 for real integration; fix issues found. Memory Saturation Trend detector working end-to-end. |
| 6 | Bug-bash, demo scenario #5 (bad rollout) if time allows. |
| 7 | Polish, docs, rehearse your part of the demo. |

## 7. Definition of done

- `kubectl apply -f demo/broken-crashloop.yaml` in the kind cluster produces a matching `Incident` CR within ~30 seconds, with `status.phase = Observed` and evidence populated.
- Same for the OOM scenario.
- A pod approaching its memory limit produces a `MemoryTrendWarning` Incident **before** it gets OOMKilled — demonstrable side-by-side with the reactive OOM scenario.
- Prometheus is reachable and returning real cluster metrics (verify with a simple PromQL query via its UI or API).
- Unit tests cover each detector's true-positive and true-negative cases.
- README explains how to run the controller locally against kind, and how to install the observability layer.

## 8. Handoff interface to Team 2

Team 2 only needs to know: **watch `Incident` CRs where `status.phase == Observed`**. Give them 2-3 sample `Incident` YAML files in week 1 so they can build against your contract before your controller exists.
