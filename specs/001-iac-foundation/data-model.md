# Data Model: IaC Foundation (Requirement 5.3)

**Feature**: `001-iac-foundation` | **Date**: 2026-11-21

This feature has no database. Its "data" is files in two Git repositories and artifacts in an
OCI registry. Exact file schemas are in [contracts/kube-state-files.md](contracts/kube-state-files.md).

## Entities

### Chart

A deployable unit in `helm-charts/charts/<name>/`.

| Field | Source | Rules |
|-------|--------|-------|
| name | `Chart.yaml: name` | Equals folder name; lowercase, `[a-z0-9-]+` |
| version | `Chart.yaml: version` | SemVer 2.0; strictly increases on any change (FR-006) |
| appVersion | `Chart.yaml: appVersion` | Quoted string; never `latest` |
| apiVersion | `Chart.yaml: apiVersion` | `v2` |
| defaultValues | `values.yaml` | No secrets (FR-013 / Principle VI) |
| templates | `templates/` | `shophub`: exactly the 7 files of FR-003; others: ≥1 file |
| status | `helm-charts/README.md` | `placeholder` or `real` (FR-009) |

In-scope instances: `shop`, `shophub`, `shop-operator`, `shophub-discord`.

### PublishedChartArtifact

An immutable chart version in GHCR.

| Field | Rules |
|-------|-------|
| ociName | `oci://ghcr.io/nikola031ue/charts/<chart.name>` |
| version | Equals `Chart.version` at publish time |
| visibility | `public` (anonymous pull, SC-005) |

**Lifecycle**: `unpublished` → (merge to `main` with new version) → `published`.
`published` is terminal: re-publishing the same version is skipped, never overwritten (FR-008).

### ClusterDefinition (`clusters/<cluster>/cluster.yaml`)

| Field | Rules |
|-------|-------|
| name | Equals folder name (`local`) |
| context | kubeconfig context bring-up must run against (`docker-desktop`) (FR-021) |
| distribution | `docker-desktop` |
| kubernetesVersion | Informational |
| releaseOrder | Lists every release folder exactly once; each release appears after all its `dependsOn` (FR-020) |

### ReleaseDefinition (`clusters/<cluster>/<release>/`)

| Field | Source | Rules |
|-------|--------|-------|
| release | folder name | Equals `HelmRelease.metadata.name` and `spec.releaseName` |
| chart OCI name | `OCIRepository.spec.url` | Must start with `oci://ghcr.io/nikola031ue/charts/` |
| chart version | `OCIRepository.spec.ref.tag` | Exact SemVer, no ranges / `latest`; must exist in GHCR |
| namespace | `HelmRelease.spec.targetNamespace` | Non-empty |
| dependsOn | `HelmRelease.spec.dependsOn[].name` | Names of other releases in the same cluster |
| overrides | `values.yaml` | Valid YAML mapping (may be `{}`); no secrets |

In-scope instances:

| release | chart | namespace | dependsOn |
|---------|-------|-----------|-----------|
| shop-operator | shop-operator | shop-operator-system | — |
| shophub | shophub | shophub | shop-operator |
| shophub-discord | shophub-discord | shophub | shophub |

**Relationships**: ClusterDefinition 1 — * ReleaseDefinition; ReleaseDefinition * — 1
PublishedChartArtifact; PublishedChartArtifact * — 1 Chart.

### Deployed release (cluster-side, observed by `bring-up.sh`)

**States per run**: `absent` → `installed`; `deployed (matching)` → `unchanged` (skipped);
`deployed (differs)` / `failed` → `upgraded`; any helm or CRD-wait error → run aborts,
later releases untouched.
