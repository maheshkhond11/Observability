# Team POC Architecture Diagrams

Companion diagrams to `01-Team-Observe-Brief.md`, `02-Team-Decide-Brief.md`, `03-Team-Act-Brief.md`.

**Update:** Team 1 and Team 3 diagrams below now include the observability layer (OpenTelemetry Collector → Prometheus) and the new metrics-based Memory Saturation Trend detector, added in response to team feedback — see `06-Platform-Mechanism-And-AI-Integration.md` and the updated team briefs.

---

## Team 1 — Observe

```mermaid
flowchart TB
    subgraph K8S[Kubernetes Cluster]
        Pods[Pods]
        Deploys[Deployments / ReplicaSets]
        Events[Kubernetes Events]
        Kubelet[kubelet / cAdvisor metrics]
    end

    subgraph Obs[Observability layer - new]
        OTel[OpenTelemetry Collector]
        Prom[(Prometheus)]
    end

    subgraph Observe[Team 1: Observe Controller]
        Informers[Informers / Watchers]
        Topo[Topology Builder]
        Detectors[Event-based Detectors]
        MetricDet[Metrics-based Detector\npolls Prometheus every ~30s]
    end

    subgraph Detect[Event-based detectors - reactive]
        D1[CrashLoopBackOff]
        D2[OOMKilled]
        D3[RestartSpike]
        D4[Unready too long]
        D5[Bad Rollout - stretch]
    end

    subgraph DetectM[Metrics-based detector - predictive, new]
        D6[Memory Saturation Trend]
    end

    Pods --> Informers
    Deploys --> Informers
    Events --> Informers
    Kubelet --> OTel --> Prom
    Informers --> Topo
    Informers --> Detectors
    Prom --> MetricDet
    Detectors --> D1
    Detectors --> D2
    Detectors --> D3
    Detectors --> D4
    Detectors --> D5
    MetricDet --> D6

    D1 --> Incident[(Incident CR\nstatus: Observed)]
    D2 --> Incident
    D3 --> Incident
    D4 --> Incident
    D5 --> Incident
    D6 -->|catches it BEFORE the crash| Incident

    Topo -. enriches .-> Incident
    Incident --> Downstream[To Team 2: Decide]
    Prom --> DashFeed[To Team 3: Cluster Health view]
```

---

## Team 2 — Decide

```mermaid
flowchart TB
    Incident[(Incident CR\nstatus: Observed)]

    subgraph Decide[Team 2: Decide Controller]
        Watch[Watch Incidents]
        Correlate[Correlation Engine\ngroup by workload + time window]
        RCA[Rule-based RCA\ndecision table]
        Policy[Policy Engine]
    end

    PolicyCR[(SelfHealingPolicy CR)]

    Watch --> Correlate --> RCA --> Policy
    PolicyCR --> Policy

    Policy -->|allowed, no approval needed| Plan[(RemediationPlan CR\nstatus: Pending)]
    Policy -->|requires approval| Wait[(RemediationPlan CR\nstatus: AwaitingApproval)]
    Wait -->|human: kubectl patch approve| Plan

    Incident --> Watch
    Plan --> Downstream[To Team 3: Act & Show]

    subgraph Stretch[Optional stretch]
        LLM[LLM summary call]
    end
    RCA -.->|summary text only - never decides| LLM
    LLM -.->|writes summary field only| Plan
```

---

## Team 3 — Act & Show

```mermaid
flowchart TB
    Plan[(RemediationPlan CR\nstatus: Pending / Remediating)]
    Prom[(Prometheus\nfrom Team 1)]

    subgraph Act[Team 3: Act Controller]
        Watch[Watch RemediationPlans]
        Guard[Guardrails\ndry-run + blast-radius check]
        Exec[Action Executor]
    end

    subgraph Actions[Action types]
        A1[RestartPod]
        A2[RollbackDeployment]
        A3[ScaleDeployment]
    end

    Watch --> Guard --> Exec
    Exec --> A1
    Exec --> A2
    Exec --> A3

    A1 --> KAPI[Kubernetes API]
    A2 --> KAPI
    A3 --> KAPI

    KAPI --> Verify[Verification Loop\npoll health up to M minutes]
    Verify -->|recovered| Resolved[(status: Resolved)]
    Verify -->|unsafe retry| Rollback[(status: Rollback)]
    Verify -->|timeout| Escalated[(status: Escalated)]

    Resolved --> API[REST API]
    Rollback --> API
    Escalated --> API
    Plan --> API
    Prom --> API

    API --> Dashboard1[Dashboard: Incident list + live timeline]
    API --> Dashboard2[Dashboard: Cluster Health view - new]
```

---

## Combined view — how the three plug together

```mermaid
flowchart LR
    subgraph T1[Team 1: Observe + Observability layer]
        direction TB
        O1[Watchers + Topology + Detectors]
        O1b[OTel Collector + Prometheus]
    end
    subgraph T2[Team 2: Decide]
        direction TB
        O2[Correlation + RCA + Policy]
    end
    subgraph T3[Team 3: Act and Show]
        direction TB
        O3[Action + Verification + Dashboard]
    end

    Cluster[(Kubernetes Cluster)] --> T1
    T1 -->|Incident CR| T2
    T2 -->|RemediationPlan CR| T3
    T3 -->|kubectl actions| Cluster
    T1 -->|live metrics| T3
    T3 --> UI1[Dashboard: Incident timeline]
    T3 --> UI2[Dashboard: Cluster Health view]
```
