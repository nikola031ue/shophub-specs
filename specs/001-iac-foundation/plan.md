# Implementation Plan: IaC Foundation (Requirement 5.3)

**Branch**: `001-iac-foundation` | **Date**: 2026-11-21 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-iac-foundation/spec.md`

## Summary

Fill the existing `helm-charts` and `kube-state` repositories with the exact Requirement 5.3
layout. `helm-charts` gets four Helm 3 charts (`shop`, `shophub`, `shop-operator`,
`shophub-discord`) as placeholders, validated on every PR (layout, `helm lint`,
`helm template` + kubeconform, SemVer bump, install on kind) and published to
`oci://ghcr.io/nikola031ue/charts` on merge to `main`, never overwriting a version.
`kube-state` gets `clusters/local/` with `cluster.yaml` and three release folders whose
`helm.yaml` is a Flux `OCIRepository` + `HelmRelease` pair pinning an exact chart version.
A root `bring-up.sh` reads only `kube-state`, checks the kube context, installs releases in
`releaseOrder` with skip-if-unchanged idempotency, and waits for any CRDs a release provides
to be `Established` before continuing. Flux can later consume the same files unchanged.

## Technical Context

**Language/Version**: Bash ≥ 4 (bring-up script, CI helpers); YAML (Helm charts, Flux objects,
GitHub Actions)

**Primary Dependencies**: Helm ≥ 3.8 (OCI support), kubectl, yq v4 (mikefarah), jq,
kubeconform, kind (CI only), bats-core (tests), GitHub Actions, GHCR

**Storage**: Git repositories (files) and GHCR (OCI chart artifacts); no database

**Testing**: `helm lint --strict`, `helm template` + kubeconform, bats-core unit tests for
`bring-up.sh` with stubbed `helm`/`kubectl`, kind-based install tests (helm-charts) and e2e
bring-up twice (kube-state)

**Target Platform**: Docker Desktop Kubernetes (context `docker-desktop`) on Windows via Git
Bash; CI on `ubuntu-latest` with kind

**Project Type**: Infrastructure-as-code (two repos: chart library + cluster state) plus a CLI
script

**Performance Goals**: Empty cluster → all releases healthy in < 15 min (SC-002); placeholder
bring-up expected < 3 min

**Constraints**: 5.3 layout immutable (Principle I); no raw `kubectl apply` (Principle V); no
secrets in git, CI auth via `GITHUB_TOKEN` only (Principle VI); chart versions immutable once
published; Windows without `make`

**Scale/Scope**: 4 charts, 1 cluster, 3 releases, 2 CI workflows, 1 script

All Technical Context items resolved in [research.md](research.md); no NEEDS CLARIFICATION
remain.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Pre-research | Post-design |
|-----------|-------|--------------|-------------|
| I. Immutable IaC layout (NON-NEGOTIABLE) | Trees match 5.3 exactly; only additions are `charts/shophub-discord/` (same pattern), hidden `.github/`, `.helmignore`/`.gitkeep`, and one root file `bring-up.sh`; no new top-level folders; CI `layout` jobs enforce it (R4, R11) | PASS | PASS |
| II. Repo-per-service & trunk-based | Work stays in existing `helm-charts` / `kube-state` repos; PRs to `main`; branch protection (required checks, linear history, no direct push, `enforce_admins`) applied **before the first merge** in each repo (T038, T047); charts SemVer-versioned; no `latest` image tag or `appVersion`, enforced by CI `check-images.sh` | PASS | PASS |
| III. Tested before merge | Unit-level: lint, template, kubeconform, schema, bats. Integration-level: kind installs and e2e bring-up. All required checks before merge. Testcontainers N/A (no .NET) (R10) | PASS | PASS |
| IV. Blockchain | Not touched by this feature | N/A | N/A |
| V. Declarative deployment | Platform releases only, via Helm, recorded in `kube-state`; no user CRs in git (FR-015); script never runs `kubectl apply/create/edit` — `kubectl` used read-only (`config current-context`, `wait`); namespaces created by Helm `--create-namespace`. **Not met while charts are placeholders**: "shop-operator chart MUST install the operator and its CRDs; shophub chart MUST consume those CRDs and the prometheus-stack" — recorded in Complexity Tracking | PASS with recorded temporary deviation | PASS with recorded temporary deviation |
| VI. Secrets | No secrets needed by placeholders; GHCR push via `GITHUB_TOKEN`; CI `secrets-scan` on `values.yaml` (R8, contracts) | PASS | PASS |
| VII. Per-Shop observability | Not applicable to placeholder IaC; observability stack arrives as additional release folders in a later feature | N/A | N/A |
| Workflow D-1 (solo) | Branch protection with 0 required approvals; self-review + required CI | PASS (recorded deviation) | PASS |

No unjustified violations. One temporary deviation (Principle V, placeholder charts) is
recorded and justified in Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/001-iac-foundation/
├── plan.md              # This file
├── research.md          # Phase 0 — decisions R1–R13
├── data-model.md        # Phase 1 — entities and lifecycles
├── quickstart.md        # Phase 1 — validation scenarios
├── contracts/
│   ├── kube-state-files.md   # cluster.yaml / helm.yaml / values.yaml schemas
│   ├── bring-up-cli.md       # bring-up.sh usage, behaviour, exit codes
│   └── ci-checks.md          # required checks per repo, publish rules
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 output (/speckit-tasks — not created here)
```

### Source Code (sibling repositories)

```text
../helm-charts/
├── README.md                       # chart list, purpose, status, OCI name (FR-009)
├── .github/
│   ├── workflows/charts.yaml       # layout, lint, template, version, install, publish, verify-public
│   ├── scripts/                    # lib.sh, check-layout.sh, check-images.sh, scan-secrets.sh, check-version.sh, publish.sh
│   └── tests/                      # bats tests for the scripts above
└── charts/
    ├── shop/
    │   ├── .helmignore
    │   ├── Chart.yaml              # 0.1.0
    │   ├── values.yaml
    │   └── templates/{NOTES.txt,_helpers.tpl}
    ├── shop-operator/              # same shape as shop
    ├── shophub-discord/            # same shape as shop; DiscordChannel added later
    └── shophub/
        ├── .helmignore             # ignores charts/.gitkeep
        ├── Chart.yaml              # 0.1.0, appVersion pinned nginx tag
        ├── values.yaml
        ├── charts/.gitkeep
        └── templates/
            ├── NOTES.txt
            ├── _helpers.tpl
            ├── deployment.yaml
            ├── hpa.yaml
            ├── ingress.yaml
            ├── service.yaml
            └── serviceaccount.yaml

../kube-state/
├── README.md                       # prerequisites + bring-up procedure (FR-017)
├── bring-up.sh                     # contracts/bring-up-cli.md
├── .github/
│   ├── workflows/validate.yaml     # layout, schema, charts-exist, render, secrets-scan, unit, e2e
│   ├── scripts/                    # lib.sh, check-layout.sh, check-schema.sh, scan-secrets.sh, check-images.sh
│   └── tests/                      # bats tests + helm/kubectl stubs
└── clusters/
    └── local/
        ├── cluster.yaml
        ├── shop-operator/{helm.yaml,values.yaml}
        ├── shophub/{helm.yaml,values.yaml}
        └── shophub-discord/{helm.yaml,values.yaml}
```

**Structure Decision**: Two existing sibling repositories, laid out exactly per Requirement
5.3. All CI tooling is under hidden `.github/`; the only visible root addition is
`kube-state/bring-up.sh` (a file, not a folder). No code lands in `shophub-specs` beyond
these design documents.

## Delivery Order

1. `helm-charts`: skeleton + CI on a PR → checks run once → configure branch protection per
   [contracts/ci-checks.md](contracts/ci-checks.md) → all checks green → merge → `publish` →
   set four GHCR packages public → `verify-public` green.
2. `kube-state`: files + `bring-up.sh` + bats tests + CI on a PR (`charts-exist` and `e2e` need
   step 1 published) → checks run once → configure branch protection → green → merge.
3. Run [quickstart.md](quickstart.md) scenarios 1–7 on Docker Desktop and verify branch
   protection rejects direct pushes.

## Risks & Follow-ups

- **GHCR visibility** is a manual one-time step per package; `verify-public` makes a miss
  visible.
- **Repo names** on GitHub are `*-spec` suffixed (research R13); renaming to `helm-charts` /
  `kube-state` before the defense is recommended.
- **CRD packaging** (`crds/` vs `templates/`) is decided in the operator feature; the script
  supports both (R6).
- **Flux adoption** later adds `clusters/local/kustomization.yaml`; release folders stay
  unchanged (R2). However, `flux bootstrap --path=clusters/local` would also create a
  `clusters/local/flux-system/` folder that has neither `helm.yaml` nor `values.yaml`, breaking
  FR-010's "exactly" rule and the `layout` check. The GitOps feature must either bootstrap Flux
  to a path outside `clusters/local/`, or add a layout-check exception justified by the 5.3
  clause that allows following a GitOps tool's structure. Not addressed in this feature.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Principle V (temporary): the `shop-operator` chart does not yet install the operator or its CRDs, and the `shophub` chart does not yet consume those CRDs or the prometheus-stack | This feature delivers only the 5.3 layout, publishing pipeline and bring-up path (spec: "Charts are placeholders now"). Real content needs the operator image (operator feature) and the observability stack (observability feature). Ends when both of those features merge; each of their plans MUST mark this deviation closed | Building the operator and observability now would merge three features into one, delay the eliminatory 5.3 check, and leave nothing to install from the operator chart anyway until the operator image exists |
