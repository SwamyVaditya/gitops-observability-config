# gitops-observability-config

GitOps config repo for the local GitOps Observability Lab. **This repo
contains no application source code and no Dockerfiles — only declarative
Kubernetes/Helm config that Argo CD reconciles against the cluster.**

Cluster + Argo CD provisioning lives in the separate
`gitops-observability-infra` repo (Terraform). This repo is what Argo CD
watches once that bootstrap is done.

## Architecture

![GitOps observability lab architecture](docs/architecture.svg)

This repo owns the **right-hand side**: everything Argo CD reconciles once
it's watching this repo — the App-of-Apps discovery in `apps/`, the
multi-source `Application` objects that pair each upstream Helm chart with
`environments/local/.../values.yaml`, and the resulting OpenSearch, Fluent
Bit Collector, and OpenSearch Dashboards workloads in the `observability`
namespace. The cluster and Argo CD itself come from
`gitops-observability-infra` — this repo assumes both already exist.

## Verified working

![Argo CD showing all 4 Applications Synced/Healthy](docs/screenshots/01-argocd-apps-synced-healthy.png)
*All 4 Applications (`root-app`, `opensearch`, `opensearch-dashboards`,
`fluent-bit-collector`) reconciled and healthy on a local k3d cluster,
after resolving the `/etc/machine-id` hostVolume issue documented in the
infra repo's `RUNBOOK.md`.*

## Layout

```
apps/                   Argo CD Application manifests only (App-of-Apps)
├── root/                 the one manifest you apply by hand (bootstrap)
├── infra/                platform-level Applications (Traefik, etc. — Step 4)
├── observability/        EFK-equivalent stack: OpenSearch, Dashboards,
│                         Fluent Bit, and the OpenSearch ISM bootstrap Job
└── sampleapp/            the checkout-flow demo app

base/                    Kustomize bases for resources we own (not
│                         third-party charts — those stay external, see
│                         Architecture above)
├── sampleapp/             Deployments/Services for the checkout flow
└── opensearch-ism-bootstrap/  ConfigMap + Job applying the ISM policy

environments/
└── local/                 overlays for the `local` environment
    ├── observability/       one subfolder per Helm release (values.yaml)
    ├── sampleapp/            Kustomize overlay (base/sampleapp/)
    └── opensearch-ism-bootstrap/  Kustomize overlay (base/opensearch-ism-bootstrap/)
```

Adding a `environments/staging/` or `environments/prod/` later means adding
sibling folders here with their own `values.yaml`/overlay — the chart
references and `base/` content stay the same, only the environment-specific
layer changes.

## Bootstrapping (one-time, manual)

After Argo CD is installed (via the infra repo's Terraform) and this repo
is pushed to a real remote:

```bash
# Replace YOUR_GITHUB_ORG in apps/root/root-app.yaml and every apps/observability/*.yaml
# with your actual repo URL first.

kubectl apply -f apps/root/root-app.yaml
```

From here on, everything is automatic: Argo CD watches `apps/` recursively,
discovers `apps/observability/*.yaml`, and syncs each one. Adding a new
Application manifest to this repo and pushing is the entire deployment
workflow — no further `kubectl apply`.

## Verifying versions before you sync

Every child `Application` pins a chart `targetRevision` as of Aug 2026, with
a comment showing how to check for a newer one. Confirm before your first
sync, since these are external charts this repo doesn't control:

```bash
helm repo add opensearch https://opensearch-project.github.io/helm-charts/
helm repo add fluent https://fluent.github.io/helm-charts/
helm repo update
helm search repo opensearch/ fluent/fluent-bit-collector --versions
```

## Known follow-ups (flagged, not yet resolved)

- `curlimages/curl:8.10.1` in `base/opensearch-ism-bootstrap/job.yaml` is
  an unverified tag — confirm it exists before relying on it.
- Whether `config.filters` in `fluent-bit-collector/values.yaml` replaces
  or appends to the chart's default filter chain is unconfirmed — written
  defensively (includes the `kubernetes` enrichment filter explicitly) so
  it works either way, but worth confirming against
  `helm show values fluent/fluent-bit-collector` directly.
- Namespace scoping (`sampleapp` only) means Argo CD's, k3s system
  components', and the observability stack's own logs are not collected.
  If you ever want those back for debugging, widen the `grep` filter's
  namespace regex — but see the incident note below first.
- Security plugins are disabled on both OpenSearch and Dashboards
  (`DISABLE_SECURITY_PLUGIN`, `DISABLE_SECURITY_DASHBOARDS_PLUGIN`) —
  lab-only simplification, not something to carry into shared environments.

Resolved: `opensearchHosts`/`Host` matching the real
`opensearch-cluster-master` Service name, and the `fluent-bit-collector`
`/etc/machine-id` hostVolume mount failing on k3d nodes — see infra repo's
`RUNBOOK.md` for the fix.

## Disk usage / log retention

Two separate real incidents happened here, confirmed on a real run —
worth understanding both since they look similar (disk climbing) but have
different causes and fixes.

**Incident 1 — unbounded ingestion (fixed).** `fluent-bit-collector`
originally matched `kube.*` (every namespace) into a single static
`app-logs` index with no retention policy, and `local-path-provisioner`
doesn't enforce a PVC's declared size as a disk quota, so nothing capped
growth. On Windows this shows up as the WSL2 `.vhdx` growing continuously;
see the infra repo's `RUNBOOK.md` for how to reclaim already-consumed
space.

**Incident 2 — self-ingestion feedback loop (fixed).** The first fix for
Incident 1 scoped collection to `sampleapp|observability` namespaces —
but `observability` is the namespace `fluent-bit-collector` itself runs
in. That meant it tailed its own container's log output, re-processed its
own previous output on the next cycle, which appeared in its own log file
again, compounding every cycle. This was an actively *growing* leak, not
a static one — confirmed by log output showing the same record nested
inside itself dozens of times. Fixed by narrowing collection to
`sampleapp` only, plus an explicit `grep Exclude` filter on
`kubernetes['pod_name'] ^fluent-bit-collector-` as defense in depth, so
this can't recur even if the namespace list is widened again later.

Fixed as of this commit:
- `fluent-bit-collector` now only collects from the `sampleapp` namespace,
  with an explicit self-exclusion filter as a second line of defense.
- Logs land in daily indices (`app-logs-YYYY.MM.DD`) instead of one
  ever-growing index.
- `apps/observability/opensearch-ism-bootstrap.yaml` deploys a Job (Argo
  CD `PostSync` hook, re-runs on every sync) that applies an OpenSearch
  ISM policy deleting `app-logs-*` indices after 3 days, and an index
  template setting `number_of_replicas: 0` (correct for a single-node
  cluster — replicas can never be assigned with only one node, so leaving
  the default just causes perpetual yellow cluster health for no benefit).
