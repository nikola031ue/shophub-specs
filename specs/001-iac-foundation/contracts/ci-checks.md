# Contract: CI checks and branch protection

Both repositories: GitHub Actions, trunk-based on `main`. Required status checks below must
be configured in branch protection together with "require linear history", "disallow
direct pushes" and `enforce_admins: true` (otherwise the repo owner — the only developer —
bypasses everything) **before the first PR is merged** (after its checks have run once, so the check
names can be selected). No required approvals (Constitution deviation D-1).

## helm-charts

Workflow `.github/workflows/charts.yaml`.

| Job (required check name) | Trigger | Does | Fails when |
|---------------------------|---------|------|------------|
| `layout` | PR, push main | Checks tree against 5.3 (FR-001–FR-004); chart folders limited to an allow-list | Missing path, extra/missing shophub template, chart outside `charts/`, chart not on allow-list |
| `lint` | PR, push main | `helm lint --strict` each chart; `scan-secrets.sh` on every chart `values.yaml` | Any lint error/warning; secret literal in chart defaults |
| `template` | PR, push main | `helm template` each chart → `kubeconform -strict -ignore-missing-schemas`; `check-images.sh` | Render error, invalid manifest, image without tag or tagged `latest`, `appVersion: latest` |
| `version` | PR | For changed charts: SemVer valid, > version on `main`, registry answers *not found* (R9) | Version not bumped / already published / registry answers unauthorized (private package) |
| `install` | PR | kind cluster; `helm install --wait` each chart with defaults (FR-005) | Any install fails |
| `publish` | push main only, `needs: [layout, lint, template]` (`version` and `install` are PR-only and already gated the merge) | For each chart: if `<chart>:<version>` not in GHCR → `helm package` + `helm push oci://ghcr.io/nikola031ue/charts`; else skip | Push fails |
| `verify-public` | after `publish` | Anonymous `helm pull` of every chart version (SC-005) | Package not public |

Permissions: `contents: read`; `publish` job adds `packages: write`. Auth via `GITHUB_TOKEN`.
`publish` uses `concurrency: publish` to prevent parallel pushes.

## kube-state

Workflow `.github/workflows/validate.yaml`.

| Job (required check name) | Trigger | Does |
|---------------------------|---------|------|
| `layout` | PR | Tree per 5.3; each release folder has exactly `helm.yaml` + `values.yaml` |
| `schema` | PR | `cluster.yaml` and `helm.yaml` rules from [kube-state-files.md](kube-state-files.md); `releaseOrder` vs `dependsOn` |
| `charts-exist` | PR | Anonymous `helm show chart <url> --version <tag>` for every release (SC-005) |
| `render` | PR | `helm template` each release with its `values.yaml`; `check-images.sh` on the output (no missing/`latest` tags) |
| `secrets-scan` | PR | Secret-key pattern check on every `values.yaml` (FR-013) |
| `unit` | PR | `bats .github/tests/` — `bring-up.sh` with stubbed `helm`/`kubectl` |
| `e2e` | PR | kind cluster, context renamed to `docker-desktop`; run `bring-up.sh` twice; second run must report every release `unchanged` (SC-003) |
