# Contract: Charts and kube-state changes

## helm-charts

### `charts/shop-operator` 0.1.0 → 0.2.0 (appVersion `0.1.0`)

| File | Content |
|------|---------|
| `templates/crds.yaml` | Shop, Wallet, DiscordChannel CRDs (synced from `shop-operator/config/crd`), annotated `helm.sh/resource-policy: keep` |
| `templates/deployment.yaml` | Operator Deployment, 1 replica, leader election on, non-root, liveness/readiness on `:8081` |
| `templates/serviceaccount.yaml`, `templates/rbac.yaml` | ServiceAccount, ClusterRole (rules synced from `config/rbac`), ClusterRoleBinding, leader-election Role/RoleBinding |
| `values.yaml` | see below — no secret values |

```yaml
image: { repository: ghcr.io/nikola031ue/shop-operator, tag: "" }   # "" → appVersion
shopImage: { repository: nginx, tag: "1.27.3" }                   # placeholder Shop app (FR-007)
discord:
  guildID: ""                         # Discord server ID (not secret)
  tokenSecret: shop-operator-discord  # Secret in the release namespace, key "token"
resources: { requests: { cpu: 50m, memory: 64Mi }, limits: { memory: 128Mi } }
```

Installs with default values on a cluster without CNPG/Redis, without a Discord Secret, and
becomes Ready (research R5, R15) — required by the `install` CI job.

### `charts/shophub-discord` 0.1.0 → 0.2.0 (FR-025)

`templates/discordchannel.yaml`: one `DiscordChannel` named `{{ .Values.channelName }}`
(default `platform-alerts`) in the release namespace. Requires the shop-operator CRDs (kube-state
already orders `shop-operator` before `shophub-discord` and waits for CRDs).

### CI (`charts.yaml`)

`install` job installs `shop-operator` first, then every other chart.

## kube-state

### New release folders

| Folder | `spec.url` | Tag | `targetNamespace` |
|--------|-----------|-----|-------------------|
| `clusters/local/cnpg/` | `oci://ghcr.io/cloudnative-pg/charts/cloudnative-pg` | `0.29.2` | `cnpg-system` |
| `clusters/local/redis-operator/` | `oci://ghcr.io/ot-container-kit/charts/redis-operator` | `0.27.0` | `redis-operator` |

Each folder has `helm.yaml` (OCIRepository + HelmRelease, same shape as feature 001) and
`values.yaml` (`{}`).

### `cluster.yaml`

```yaml
releaseOrder: [cnpg, redis-operator, shop-operator, shophub, shophub-discord]
```

`shop-operator` `dependsOn` `cnpg` and `redis-operator`; `shop-operator` and
`shophub-discord` tags bumped to `0.2.0`.

### Upstream allowlist — `.github/upstream-charts.yaml`

```yaml
# Charts not built in helm-charts that kube-state may reference. Exact url + version only.
charts:
  - url: oci://ghcr.io/cloudnative-pg/charts/cloudnative-pg
    version: 0.29.2
  - url: oci://ghcr.io/ot-container-kit/charts/redis-operator
    version: 0.27.0
```

`check-schema.sh` rule (replaces "url must start with `$OCI_BASE/`"):

| Case | Result |
|------|--------|
| url under `$OCI_BASE/` | pass (existing rules) |
| url equals an allowlist `url` and tag equals its `version` | pass |
| url allowlisted, tag differs | fail: `<file>: <url> is allowlisted only at version <v> (got <tag>); update .github/upstream-charts.yaml in the same PR` |
| any other url | fail: `<file>: spec.url must start with <OCI_BASE>/ or be listed in .github/upstream-charts.yaml` |
| allowlist missing/invalid or an entry without exact SemVer `version` | fail (exit non-zero) |

Covered by new bats cases in `check-schema.bats`.
