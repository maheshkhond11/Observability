# Repository Naming and Layout

## Suggested name: `selfheal-poc`

Reasons:
- Matches the Helm chart/install name already used in the source architecture doc (`helm install selfheal ./charts/selfheal`), so anyone cross-referencing the doc and the repo isn't confused by a different name.
- The `-poc` suffix is honest about scope to anyone browsing your org's GitHub — sets correct expectations before they open a single file, and makes it easy to later rename/fork into a production repo (e.g. `selfheal`) without ambiguity.
- Clear in a company org repo list next to other "-poc" or "-platform" naming.

## Alternatives, if you want something punchier for the community demo

| Name | Read |
|---|---|
| `cluster-medic` | Playful, memorable, communicates "fixes clusters" instantly |
| `k8s-autopilot` | Communicates autonomy clearly; slightly generic (many projects use "autopilot") |
| `sentinel-selfheal` | Sounds more "platform-grade"; a bit long |
| `aegis-k8s` | Short, brandable; less immediately descriptive of what it does |

If the architect community likes to name these efforts memorably for internal tracking, `cluster-medic` is my second choice after `selfheal-poc` — pick one and stay consistent across the repo, the demo slides, and any internal wiki page.

## Suggested repo structure

Single monorepo, not 3 separate repos — this avoids the 3 teams' CRD schemas drifting out of sync with each other.

```
selfheal-poc/
  api/v1alpha1/              # Shared CRD Go types — Incident, RemediationPlan, SelfHealingPolicy
                              # Changes here need sign-off from all 3 team leads after week 1
  internal/
    observe/                 # Team 1
    decide/                  # Team 2
    act/                     # Team 3
  cmd/
    observe-controller/
    decide-controller/
    act-controller/
    api-gateway/
  charts/selfheal/           # Helm chart to install all 3 controllers + CRDs
  config/
    crd/
    rbac/
    samples/                 # Hand-written sample Incident / RemediationPlan CRs for cross-team dev
  web/dashboard/              # Team 3's frontend
  deployments/kind/           # kind cluster config + demo "broken app" manifests
  docs/
    architecture/             # Observability-POC-SPARK-info.md lives here
    team-briefs/               # the 3 team brief docs
    00-Team-Lead-Summary.md
  test/
    envtest/
    e2e/
  Makefile
  go.mod
  README.md
```

## GitHub setup suggestions

- **Branch protection on `main`**, PRs required, at minimum 1 reviewer.
- **CODEOWNERS**: require a review from all 3 team leads for changes under `api/v1alpha1/` (the shared contract) — this is the one directory where an unreviewed change can silently break another team.
- **GitHub Projects board** with one column per lifecycle stage (Todo / In Progress / In Review / Done) and a label per team (`team-observe`, `team-decide`, `team-act`, `shared-contract`). Turn each work-item row from the 3 brief docs into an issue.
- **Labels**: `team-observe`, `team-decide`, `team-act`, `shared-contract`, `stretch-goal`, `demo-blocker` (use this last one to flag anything that would break the live demo — triage these first every standup).
- README at repo root should link to `docs/architecture/Observability-POC-SPARK-info.md` and `docs/00-Team-Lead-Summary.md` so anyone landing on the repo cold understands the scope and the "why" in under a minute.
