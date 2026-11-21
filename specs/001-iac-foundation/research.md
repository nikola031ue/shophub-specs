# Research: IaC Foundation (Requirement 5.3)

**Feature**: `001-iac-foundation` | **Date**: 2026-11-21

Inputs fixed by the user: Helm 3, charts published to GHCR as OCI, Docker Desktop Kubernetes,
plain script for bring-up (Flux optional later, layout must stay Flux-compatible), GitHub
Actions in `helm-charts` running `helm lint`, `helm template` and OCI push on `main`.

Local tool check (2026-11-21): `kubectl` (Docker Desktop) and `gh` present; `helm`, `yq`,
`kind`, `bats`, `make` absent. Git remotes: `nikola031ue/helm-charts-spec`,
`nikola031ue/kube-state-spec`.

---

## R1. Format of `helm.yaml`

**Decision**: `helm.yaml` is a two-document YAML file containing real Flux objects:
an `OCIRepository` (`source.toolkit.fluxcd.io/v1`) that names the chart by OCI URL and pins
an exact version in `spec.ref.tag`, and a `HelmRelease` (`helm.toolkit.fluxcd.io/v2`) that
references it via `spec.chartRef` and carries `releaseName`, `targetNamespace`, `dependsOn`
and a `valuesFrom` entry pointing at a ConfigMap generated from the sibling `values.yaml`.
The bring-up script reads only the fields it needs with `yq`; Flux, when added, applies the
same file unchanged.

**Rationale**: satisfies "chart name in OCI format" (5.3), FR-012 (OCI name + exact version +
release name + namespace), and the user's "keep Flux-compatible" with zero translation later.
`dependsOn` encodes FR-020 ordering in a form Flux enforces natively.

**Alternatives considered**:
- Custom minimal schema (`chart:`, `version:`, `namespace:`): simplest to parse, but a Flux
  migration would need every file rewritten.
- Plain chart reference string: does not carry release name / namespace (fails FR-012).
- `HelmRepository` of type `oci`: older Flux pattern; `OCIRepository` + `chartRef` is the
  current recommended way for OCI charts.

## R2. How Flux will later consume `values.yaml` without changing the layout

**Decision**: deferred to the GitOps feature, but the path is fixed now: add
`clusters/local/kustomization.yaml` with `resources: [<release>/helm.yaml …]` and a
`configMapGenerator` per release (`files: [values.yaml=<release>/values.yaml]`,
`disableNameSuffixHash: true`) producing `<release>-values`, which `valuesFrom` already
references. Kustomize's load restrictor allows files in sub-directories, so no file moves.

**Rationale**: release folders keep exactly `helm.yaml` + `values.yaml` (FR-010); 5.3
explicitly allows following a GitOps tool's structure when one is adopted.

**Alternatives considered**: inlining values into `HelmRelease.spec.values` (would delete
`values.yaml`, breaking 5.3); `.sourceignore` + per-folder kustomization (extra files per
release folder).

## R3. Format of `cluster.yaml`

**Decision**: plain YAML, deliberately *not* a Kubernetes object (no `apiVersion`/`kind`), so
no tool ever tries to apply it. Fields: `name`, `context`, `distribution`,
`kubernetesVersion` (informational), `releaseOrder` (list of release folder names). See
`contracts/kube-state-files.md`.

**Rationale**: FR-011 metadata; `releaseOrder` gives the script an explicit, reviewable
install order (FR-020) and CI checks that it is consistent with every `dependsOn`.

**Alternatives considered**: derive order by topologically sorting `dependsOn` in bash (more
code, harder to test); hard-code order in the script (must edit the script for every new
release).

## R4. Bring-up entry point: script vs. Makefile

**Decision**: a single Bash script `bring-up.sh` at the root of `kube-state`, run from Git
Bash (Windows), WSL, macOS or Linux. No Makefile.

**Rationale**: `make` is not installed on the maintainer's Windows machine and is not
bundled with Git Bash; Bash is. One root *file* adds no new top-level *folder*, keeping the
visible layout identical to 5.3 (Constitution Principle I).

**Alternatives considered**: Makefile (needs extra install on Windows); PowerShell script
(not usable in Linux CI); `helmfile` (another tool and another file format that duplicates
`helm.yaml`); putting the script in `scripts/` (new top-level folder — avoided for an
eliminatory requirement).

## R5. Idempotency (FR-019, SC-003)

**Decision**: before each release the script compares desired vs. deployed state —
chart version, release status (`deployed`) and values. The deployed version comes from
`helm list -o json --filter '^<release>$'` field `.chart` (format `<chart name>-<version>`),
with the `<chart name>-` prefix removed; the chart name is the last path segment of the OCI URL
(chart names contain hyphens, so the version is not split on the first `-`). Values compare
`helm get values -o json` with `yq -o json values.yaml`, both normalised with
`jq -S '. // {}'`: Helm returns `null` when no user values were supplied and `yq` returns `{}`
(or `null` for a comments-only file), so without `// {}` an empty `values.yaml` would never
compare equal and every run would upgrade. If identical it
prints `unchanged` and skips; otherwise it runs
`helm upgrade --install --wait --timeout 5m --create-namespace`.

**Rationale**: `helm upgrade` always creates a new revision even when nothing changed; the
skip makes a no-op run produce 0 changes without adding the `helm-diff` plugin.

**Alternatives considered**: `helm-diff` plugin (extra install, more accurate but not
needed for placeholder scope); always upgrade (violates SC-003 by bumping revisions).

## R6. Waiting for CRDs (FR-023)

**Decision**: after a release succeeds, the script collects CRD names provided by it from
both `helm show crds <chart> --version <v>` (charts' `crds/` dir) and
`helm get manifest <release>` filtered to `kind: CustomResourceDefinition` (CRDs in
`templates/`). If the list is non-empty it runs
`kubectl wait --for=condition=Established crd/<name> --timeout=${CRD_WAIT_TIMEOUT:-120s}`
for each and aborts with the CRD name on timeout. Applied after every release (not only
`shop-operator`), which satisfies FR-023 and generalises it.

**Rationale**: `kubectl wait` is read-only (allowed by Principle V). Collecting from both
sources keeps the script correct whichever CRD packaging the operator feature picks.
An empty list skips the wait, so placeholders pass immediately.

**Alternatives considered**: label selector wait (`kubectl wait -l …` errors when nothing
matches, and requires a labelling convention); fixed sleep (flaky).

**Note for the operator feature**: Helm never upgrades CRDs in `crds/`; kubebuilder CRDs
change often, so placing them in `templates/` with `helm.sh/resource-policy: keep` is
likely preferable. Not decided here.

## R7. Context safety (FR-021)

**Decision**: the script reads `context` from `cluster.yaml`, fails if
`kubectl config current-context` differs, and additionally passes `--kube-context` to every
`helm`/`kubectl` call so a context switch mid-run cannot redirect it.

**Alternatives considered**: only passing `--kube-context` (would silently ignore a
mismatched current context, weaker than FR-021).

## R8. GHCR naming, visibility and auth

**Decision**:
- OCI base: `oci://ghcr.io/nikola031ue/charts`; each chart is
  `oci://ghcr.io/nikola031ue/charts/<chart>:<version>`. Independent of the GitHub repo name.
- CI authenticates with the built-in `GITHUB_TOKEN` (`permissions: packages: write`) via
  `helm registry login ghcr.io` — no stored secret needed (Principle VI).
- GHCR creates packages as **private** on first push. After the first publish of each chart,
  the maintainer sets the package visibility to **public** once (GitHub UI → Package
  settings). Documented in `quickstart.md`; CI's anonymous-pull check (SC-005) catches a
  forgotten step.

**Alternatives considered**: Docker Hub (separate account + token secret); repo-name-based
path `ghcr.io/nikola031ue/helm-charts-spec/…` (breaks if repo is renamed).

## R9. Chart version rules in CI (FR-006, FR-007, FR-008)

**Decision**: a bash check in `helm-charts` CI, for each chart whose files changed vs. the PR
base: `version` in `Chart.yaml` must be valid SemVer 2.0 and strictly greater than the
version on `main` (new charts exempt), and `helm show chart oci://…/<chart> --version <v>`
must fail with a *not-found* error (not yet published). Any other failure — notably
unauthorized/denied, which is what GHCR returns for a private package — fails the check, so a
private package can never make a reused version look unpublished (the logged-in `publish` job
would otherwise see it, skip, and silently drop the change).

*Implementation finding (2026-11-21, verified against ghcr.io)*: GHCR answers anonymous pulls
of a package that **does not exist yet** with the same `403 Forbidden` as a private one, so the
two cannot be told apart from the registry answer. Rule used instead: *denied* for a chart that
already exists on `main` → fail (it must have been published, hence it is private); *denied*
for a chart that is new in the PR → not yet published → pass. `publish.sh` pushes on
*not found* or *denied* and refuses on any other error (no blind overwrite). Publish on `main` pushes only versions that are not yet in
GHCR; existing versions are skipped, never overwritten.

**Alternatives considered**: `chart-testing` (`ct lint --check-version-increment`) — does the
same plus maintainers/yamllint rules, but adds a config and container; can replace the bash
check later without changing behaviour.

## R10. Validation and tests (Constitution Principle III)

**Decision**:
- *helm-charts PR*: layout check (incl. chart allow-list) → `helm lint --strict` per chart →
  `helm template` per chart piped to `kubeconform -strict -ignore-missing-schemas` plus an
  image-tag check (no missing/`latest` tags, no `appVersion: latest` — Principle II) →
  version check (R9) →
  install test: create a kind cluster, `helm install --wait` each chart with defaults
  (FR-005).
- *kube-state PR*: layout check; `cluster.yaml` schema + `releaseOrder` covers every folder
  and respects `dependsOn`; each `helm.yaml` parses, has exactly the two expected objects,
  pins an exact SemVer tag, and the chart version is pullable anonymously (SC-005);
  `helm template` of each release with its `values.yaml`; secret-pattern scan of
  `values.yaml`; **bats** unit tests of `bring-up.sh` with stubbed `helm`/`kubectl`;
  **e2e**: kind cluster whose context is renamed to `docker-desktop`, run `bring-up.sh`
  twice and assert the second run reports all releases `unchanged`.
- Testcontainers does not apply (no .NET code in this feature).

**Rationale**: unit level (lint/render/schema/bats) + integration level (real installs on
kind) before merge. Renaming the kind context lets CI exercise the real FR-021 check
instead of bypassing it.

## R11. Where CI tooling lives

**Decision**: CI helper scripts and bats tests live under `.github/` in each repo
(`.github/workflows/`, `.github/scripts/`, `.github/tests/`). The only non-hidden additions
at repository roots are `bring-up.sh` (kube-state) and nothing in helm-charts.

**Rationale**: keeps the visible tree exactly as in 5.3 (Principle I). Hidden CI folders are
universally expected and are not a "parallel layout" for charts or cluster state.

## R12. Placeholder chart contents

**Decision**:
- `shophub`: the standard `helm create` scaffold trimmed to exactly the seven template files
  of 5.3 (the scaffold's `templates/tests/` is removed). Runs one pinned nginx pod so
  bring-up has a real workload to report healthy. `charts/` kept via `charts/.gitkeep`,
  listed in `.helmignore` so Helm does not treat it as a subchart.
- `shop`, `shop-operator`, `shophub-discord`: `Chart.yaml`, `values.yaml`, `.helmignore`,
  `templates/NOTES.txt` + `templates/_helpers.tpl`; render no resources (release installs
  as `deployed` with zero objects).
- All charts start at version `0.1.0`, `apiVersion: v2`, `type: application`.
- Namespaces: `shop-operator` → `shop-operator-system`; `shophub` and `shophub-discord` →
  `shophub`.

**Alternatives considered**: empty `templates/` (Git cannot track empty folders; a stray
`.gitkeep` would be rendered); placeholder pods in every chart (noise, no value).

## R13. Open item outside this plan

GitHub repo names are `helm-charts-spec` / `kube-state-spec`, while Requirement 5.3 calls
them `helm-chart(s)` / `kube-state`. Local folder names match. Renaming the GitHub repos to
`helm-charts` / `kube-state` before the defense is recommended; nothing in this plan
depends on the repo names (R8).
