---

description: "Task list for 001-iac-foundation"
---

# Tasks: IaC Foundation (Requirement 5.3)

**Input**: Design documents from `/specs/001-iac-foundation/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Included. Constitution Principle III requires unit + integration tests to pass in CI
before merge; plan.md (Testing) and research.md R10 define them. Test-first is recommended, so
test tasks are listed before the implementation they cover within each story.

**Organization**: Tasks are grouped by user story so each story can be implemented and
validated independently.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an incomplete task)
- **[Story]**: User story from spec.md (US1, US2, US3)

## Path Conventions

Work happens in two sibling repositories of `shophub-specs`:

- `../helm-charts/` — chart library (GitHub: `nikola031ue/helm-charts-spec`)
- `../kube-state/` — cluster state (GitHub: `nikola031ue/kube-state-spec`)

Paths below are written as `helm-charts/...` and `kube-state/...` relative to the workspace
root `C:\Git\shophub-workspace\`. OCI base for all charts: `oci://ghcr.io/nikola031ue/charts`
(research.md R8).

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Branches, line endings and local tooling in both repos

- [X] T001 Create branch `001-iac-foundation` from `main` in `helm-charts/` and in `kube-state/` (one branch per repo; trunk-based, Principle II)
- [X] T002 [P] Add `helm-charts/.gitattributes` forcing LF (`* text=auto eol=lf`, `*.sh text eol=lf`, `*.tpl text eol=lf`) so scripts and templates authored on Windows run in Linux CI
- [X] T003 [P] Add `kube-state/.gitattributes` with the same LF rules as T002 (critical for `bring-up.sh` in Git Bash and CI)
- [X] T004 [P] Add `helm-charts/.gitignore` ignoring `*.tgz`, `.cr-release-packages/`, `.idea/`, `.vscode/`
- [X] T005 [P] Add `kube-state/.gitignore` ignoring `.idea/`, `.vscode/`, `*.local.yaml`, `.env*`
- [ ] T006 Install local tools on the maintainer machine per quickstart.md Prerequisites (`helm` ≥ 3.8, `yq` v4 mikefarah, `jq`; `kubectl` and `gh` already present) and confirm `kubectl config get-contexts` lists `docker-desktop`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: CI workflow skeletons that every story adds jobs to

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [X] T007 [P] Create `helm-charts/.github/workflows/charts.yaml` skeleton: triggers `pull_request` and `push` to `main`; top-level `permissions: contents: read`; a reusable "setup tools" step block installing pinned exact versions of helm, yq, jq, kubeconform and bats (versions in `env:`, each release ≥ 2 weeks old); no jobs beyond a placeholder `noop` (removed in T019)
- [X] T008 [P] Create `kube-state/.github/workflows/validate.yaml` skeleton: trigger `pull_request` (and `push` to `main`); `permissions: contents: read`; same pinned tool setup as T007; placeholder `noop` job (removed in T027)
- [X] T009 [P] Create `helm-charts/.github/scripts/lib.sh` with shared Bash helpers (`set -euo pipefail`, `die()`, `log()`, `semver_valid()` implementing strict SemVer 2.0 regex, `semver_gt()`), sourced by later scripts
- [X] T010 [P] Create `kube-state/.github/scripts/lib.sh` with the same helpers as T009 plus `yq_doc()` for selecting a document by `kind` from a multi-doc YAML

**Checkpoint**: Both repos have LF rules, CI skeletons and shared helpers

---

## Phase 3: User Story 1 - Repository layouts match Requirement 5.3 exactly (Priority: P1) 🎯 MVP

**Goal**: Both repos contain exactly the 5.3 trees (plus `charts/shophub-discord/`), enforced
by CI.

**Independent Test**: quickstart.md Scenario 1 — listed trees match `docs/ShopHub.md` §5.3;
`layout` green in helm-charts; `layout`, `schema` and `secrets-scan` green in kube-state.

### Tests for User Story 1

- [X] T011 [P] [US1] Write bats tests for the helm-charts layout checker in `helm-charts/.github/tests/check-layout.bats`: passes on a fixture copy of the correct tree; fails (non-zero, naming the path) when a chart lacks `Chart.yaml`/`values.yaml`/`templates/`, when `charts/shophub/charts/` is missing, when `charts/shophub/templates/` has a missing or extra file, when a `Chart.yaml` exists outside `charts/`, and when `charts/` contains a chart not on the allow-list (`shop`, `shophub`, `shop-operator`, `shophub-discord`), e.g. `charts/bitcoin/` (FR-001–FR-004; `data-model.md` Chart.templates)
- [X] T012 [P] [US1] Write bats tests for the kube-state layout + schema checkers in `kube-state/.github/tests/check-layout.bats` and `kube-state/.github/tests/check-schema.bats`: cover missing/extra files in a release folder, missing `cluster.yaml`, `releaseOrder` missing a folder or listing a dependency after its dependent, `helm.yaml` with ≠ 2 documents or wrong kinds, name mismatches, non-`ghcr.io/nikola031ue/charts/` URL, non-SemVer or `latest` tag, wrong `valuesFrom` name (rules from `contracts/kube-state-files.md`)
- [X] T013 [P] [US1] Write bats tests for the secrets scanner in `kube-state/.github/tests/scan-secrets.bats`: flags `password: abc`, `discordToken: x`; allows `existingSecret: foo`, `tokenSecretName: bar`, empty values (FR-013; `data-model.md` ReleaseDefinition.overrides)

### Implementation for User Story 1 — helm-charts

- [X] T014 [P] [US1] Generate `helm-charts/charts/shophub/` with `helm create shophub`, then delete `templates/tests/`, keep exactly `NOTES.txt`, `_helpers.tpl`, `deployment.yaml`, `hpa.yaml`, `ingress.yaml`, `service.yaml`, `serviceaccount.yaml`; add `charts/.gitkeep` and list `charts/.gitkeep` in `.helmignore`; set `Chart.yaml` `version: 0.1.0`, `apiVersion: v2`, `appVersion` to a pinned full nginx tag (never `latest`), description noting placeholder (research.md R12; `data-model.md` Chart.name, Chart.version, Chart.appVersion, Chart.apiVersion, Chart.templates)
- [X] T015 [P] [US1] Create placeholder chart `helm-charts/charts/shop/` with `Chart.yaml` (name `shop`, version `0.1.0`, `apiVersion: v2`, `type: application`, quoted `appVersion: "0.0.0"`), `values.yaml` (`{}` plus a comment), `.helmignore`, `templates/_helpers.tpl` (name/fullname/labels helpers) and `templates/NOTES.txt` stating it is a placeholder; renders zero resources (`data-model.md` Chart.name, Chart.version, Chart.appVersion, Chart.apiVersion, Chart.defaultValues, Chart.templates)
- [X] T016 [P] [US1] Create placeholder chart `helm-charts/charts/shop-operator/` with the same shape and rules as T015 (name `shop-operator`; NOTES mention CRDs arrive in the operator feature)
- [X] T017 [P] [US1] Create placeholder chart `helm-charts/charts/shophub-discord/` with the same shape and rules as T015 (name `shophub-discord`; NOTES state it will hold one platform/cluster-alert `DiscordChannel`, FR-014)
- [X] T018 [US1] Implement `helm-charts/.github/scripts/check-layout.sh` (sources `lib.sh`) enforcing FR-001–FR-004: the exact seven shophub templates, and an explicit allow-list of chart folders under `charts/` (`shop`, `shophub`, `shop-operator`, `shophub-discord`) defined at the top of the script — adding a chart in a later feature means extending this list in the same PR; make T011 pass; mark executable (`git update-index --chmod=+x`)
- [X] T019 [US1] Add job `layout` to `helm-charts/.github/workflows/charts.yaml` running `check-layout.sh` and `bats .github/tests/check-layout.bats`; remove the `noop` job
- [X] T020 [US1] Write `helm-charts/README.md`: purpose, repository tree, chart table (name, purpose, status `placeholder`, OCI name `oci://ghcr.io/nikola031ue/charts/<name>`), versioning rule (bump on every change, never republish), link to kube-state (FR-009; `data-model.md` Chart.status, PublishedChartArtifact.ociName)

### Implementation for User Story 1 — kube-state

- [X] T021 [P] [US1] Create `kube-state/clusters/local/cluster.yaml` exactly per `contracts/kube-state-files.md` (`name: local`, `context: docker-desktop`, `distribution: docker-desktop`, informational `kubernetesVersion`, `releaseOrder: [shop-operator, shophub, shophub-discord]`); no `apiVersion`/`kind` (research.md R3; `data-model.md` ClusterDefinition.name, ClusterDefinition.context, ClusterDefinition.distribution, ClusterDefinition.kubernetesVersion, ClusterDefinition.releaseOrder)
- [X] T022 [P] [US1] Create `kube-state/clusters/local/shop-operator/helm.yaml` (OCIRepository + HelmRelease per contract; url `oci://ghcr.io/nikola031ue/charts/shop-operator`, tag `0.1.0`, `targetNamespace: shop-operator-system`, no `dependsOn`, `valuesFrom` `shop-operator-values`) and `values.yaml` (`{}`) (`data-model.md` ReleaseDefinition.release, ReleaseDefinition.chart OCI name, ReleaseDefinition.chart version, ReleaseDefinition.namespace, ReleaseDefinition.dependsOn, ReleaseDefinition.overrides)
- [X] T023 [P] [US1] Create `kube-state/clusters/local/shophub/helm.yaml` (url `…/charts/shophub`, tag `0.1.0`, `targetNamespace: shophub`, `dependsOn: [shop-operator]`, `valuesFrom` `shophub-values`) and `values.yaml` (`{}`) with the same citations as T022
- [X] T024 [P] [US1] Create `kube-state/clusters/local/shophub-discord/helm.yaml` (url `…/charts/shophub-discord`, tag `0.1.0`, `targetNamespace: shophub`, `dependsOn: [shophub]`, `valuesFrom` `shophub-discord-values`) and `values.yaml` (`{}`) with the same citations as T022
- [X] T025 [US1] Implement `kube-state/.github/scripts/check-layout.sh` and `kube-state/.github/scripts/check-schema.sh` (source `lib.sh`; use `yq`) enforcing FR-010–FR-012, FR-015 and every validation rule in `contracts/kube-state-files.md`, including `releaseOrder` vs `dependsOn` consistency; make T012 pass; mark executable
- [X] T026 [US1] Implement `kube-state/.github/scripts/scan-secrets.sh` per the pattern in `contracts/kube-state-files.md` § values.yaml; make T013 pass; mark executable
- [X] T027 [US1] Add jobs `layout`, `schema` and `secrets-scan` to `kube-state/.github/workflows/validate.yaml` (each also runs its bats file); remove the `noop` job
- [X] T028 [US1] Write the structure part of `kube-state/README.md`: purpose, repository tree, platform-state-only rule (user Shop/Wallet/DiscordChannel/database CRs never stored here — Principle V), how to add a release folder, `cluster.yaml` and `helm.yaml` field reference linking the contract; leave a "Bring-up" heading for T046

**Checkpoint**: US1 complete on the feature branches — layouts are correct; `layout` is green
on the helm-charts PR and `layout`, `schema`, `secrets-scan` are green on the kube-state PR. The eliminatory check is
validated here, on the branch; US1 is **not merged on its own**: its helm-charts work merges
with US2 (T038) and its kube-state work merges with US3 (T047)

---

## Phase 4: User Story 2 - Charts are published as versioned OCI artifacts (Priority: P2)

**Goal**: Every PR validates charts and blocks un-bumped versions; merges publish new versions
to GHCR, never overwriting, and verify anonymous pull.

**Independent Test**: quickstart.md Scenarios 2 and 3.

### Tests for User Story 2

- [X] T029 [P] [US2] Write bats tests in `helm-charts/.github/tests/check-version.bats` for the version checker with stubbed `git show` / `helm show chart`: unchanged chart → skip; changed chart with bumped version → pass; changed chart with same or lower version → fail; invalid SemVer → fail; version already in registry → fail; registry answers unauthorized/denied (e.g. private package) → fail with a message pointing to "Make packages public", never treated as "not published"; registry answers not-found → treated as not published; brand-new chart → pass (FR-006, FR-007; `data-model.md` Chart.version, PublishedChartArtifact.version)
- [X] T030 [P] [US2] Write bats tests in `helm-charts/.github/tests/publish.bats` with stubbed `helm`: version present in registry → no push, logs `skip`; absent → `helm package` + `helm push` to `oci://ghcr.io/nikola031ue/charts` (FR-008)

### Implementation for User Story 2

- [X] T031 [US2] Implement `helm-charts/.github/scripts/check-version.sh`: detect charts changed vs `origin/${GITHUB_BASE_REF}`; enforce research.md R9 rules; classify the anonymous `helm show chart` result by its error output — success → published (fail), "not found"/manifest unknown → not published (pass), anything else including unauthorized/denied/403 → fail; make T029 pass; mark executable (`data-model.md` Chart.version, PublishedChartArtifact.visibility)
- [X] T032 [US2] Implement `helm-charts/.github/scripts/publish.sh`: for each chart, skip if `helm show chart oci://ghcr.io/nikola031ue/charts/<name> --version <v>` succeeds, else package and push; make T030 pass; mark executable (`data-model.md` PublishedChartArtifact.ociName, PublishedChartArtifact.version — terminal `published` state)
- [X] T033 [US2] Add jobs `lint` (`helm lint --strict` each chart, then `helm-charts/.github/scripts/scan-secrets.sh` — same pattern and bats cases as kube-state T013/T026, in `helm-charts/.github/tests/scan-secrets.bats` — over every `charts/*/values.yaml`; `data-model.md` Chart.defaultValues, Principle VI) and `template` (`helm template` each chart piped to `kubeconform -strict -ignore-missing-schemas -summary`) to `helm-charts/.github/workflows/charts.yaml`; the `template` job also runs `helm-charts/.github/scripts/check-images.sh` (new, sources `lib.sh`, executable), which fails if any `Chart.yaml` `appVersion` is `latest`, or if any rendered container `image:` has no tag or the tag `latest` (Constitution Principle II; `data-model.md` Chart.appVersion), with bats cases in `helm-charts/.github/tests/check-images.bats` (pinned tag passes; `nginx`, `nginx:latest`, `appVersion: latest` fail)
- [X] T034 [US2] Add job `version` (PR only; `fetch-depth: 0`; runs `check-version.sh` and its bats file) to `helm-charts/.github/workflows/charts.yaml`
- [X] T035 [US2] Add job `install` (PR only) to `helm-charts/.github/workflows/charts.yaml`: create a kind cluster with a pinned kind action version, `helm install --wait --timeout 5m --create-namespace` each chart with default values into its own namespace, then `helm list -A` shows all `deployed` (FR-005)
- [X] T036 [US2] Add job `publish` (push to `main` only; `needs: [layout, lint, template]`; `permissions: packages: write`; `concurrency: publish`) to `helm-charts/.github/workflows/charts.yaml`: `echo "$GITHUB_TOKEN" | helm registry login ghcr.io -u ${{ github.actor }} --password-stdin`, run `publish.sh`, bats for publish (Principle VI — no stored secret)
- [X] T037 [US2] Add job `verify-public` (`needs: publish`) to `helm-charts/.github/workflows/charts.yaml`: with no registry login, `helm show chart` every chart at its `Chart.yaml` version; fail with a message pointing to quickstart.md "Make packages public" (SC-005; `data-model.md` PublishedChartArtifact.visibility)
- [ ] T038 [US2] Open the helm-charts PR (contains US1 + US2 helm-charts work) and let every check run once; **before merging**, configure branch protection on `main` of `nikola031ue/helm-charts-spec` via `gh api` (required checks `layout`, `lint`, `template`, `version`, `install`; require linear history; block force-push and direct push; `enforce_admins: true` so the repo owner cannot bypass the rules; 0 required approvals per D-1) and record the command in `helm-charts/README.md`; then get all checks green, self-review the full diff (D-1), squash-merge, confirm `publish` pushed four charts at `0.1.0` (Constitution Principle II)
- [ ] T039 [US2] Set each GHCR package `charts/shop`, `charts/shophub`, `charts/shop-operator`, `charts/shophub-discord` to Public (GitHub → Packages → Package settings), re-run `verify-public` until green (research.md R8)

**Checkpoint**: US2 complete — four charts pullable anonymously at `0.1.0`

---

## Phase 5: User Story 3 - Bring up the platform on an empty cluster from kube-state alone (Priority: P3)

**Goal**: `./bring-up.sh` installs all releases from `kube-state` only, safely and idempotently.

**Independent Test**: quickstart.md Scenarios 4–7.

**Depends on**: US1 (files) and US2 (published charts) for the `charts-exist`, `render` and
`e2e` jobs; the script and its unit tests can be built in parallel with US2.

### Tests for User Story 3

- [X] T040 [P] [US3] Create stubs `kube-state/.github/tests/stubs/helm` and `kube-state/.github/tests/stubs/kubectl` (Bash; record every invocation to `$STUB_LOG`; return canned output selected by env vars for `config current-context`, `list -o json`, `get values -o json`, `show crds`, `get manifest`, `wait`, `upgrade --install`)
- [X] T041 [US3] Write `kube-state/.github/tests/bring-up.bats` covering `contracts/bring-up-cli.md`: missing tool → exit 2; missing/invalid `cluster.yaml` → exit 3; context mismatch → exit 4 and no helm call; installs follow `releaseOrder`; every helm/kubectl call carries `--kube-context docker-desktop`; matching deployed version+values+status → `unchanged` and no `upgrade`; empty `values.yaml` (`{}` or comments only) with `helm get values` returning `null` → `unchanged` (normalisation case); deployed chart version parsed from `helm list` `.chart` = `shop-operator-0.1.0` (chart name containing hyphens) compares equal to `0.1.0`; differing values → `upgraded`; first-release failure → exit 1 and later releases not attempted; CRDs from `show crds` and from `get manifest` both waited on with `CRD_WAIT_TIMEOUT`; CRD wait timeout → exit 5 naming the CRD; no CRDs → `crds: none` and no `wait`; missing chart version → exit 6; script never calls `kubectl apply|create|edit|delete` (FR-019–FR-023)

### Implementation for User Story 3

- [X] T042 [US3] Implement `kube-state/bring-up.sh` per `contracts/bring-up-cli.md` and research.md R5–R7: prerequisite checks, load `cluster.yaml`, context guard, ordered loop over `releaseOrder`, skip-if-unchanged comparison — deployed version = `.chart` from `helm list -o json --filter '^<release>$'` with the `<chart name>-` prefix removed (chart name taken from the last path segment of the OCI URL), compared to the `helm.yaml` tag; values = `helm get values -o json` vs `yq -o json values.yaml`, both normalised with `jq -S '. // {}'` so `null` and `{}` compare equal; `helm upgrade --install --wait --timeout $HELM_TIMEOUT --create-namespace`, CRD collection + `kubectl wait --for=condition=Established`, summary line and exit codes; make T041 pass; mark executable (`data-model.md` ClusterDefinition.context, ClusterDefinition.releaseOrder, ReleaseDefinition.chart OCI name, ReleaseDefinition.chart version, ReleaseDefinition.namespace, ReleaseDefinition.overrides)
- [X] T043 [US3] Add job `unit` (runs `bats .github/tests/bring-up.bats` with `PATH` prefixed by the stubs dir) to `kube-state/.github/workflows/validate.yaml`
- [X] T044 [US3] Add jobs `charts-exist` (anonymous `helm show chart <url> --version <tag>` for every release, SC-005) and `render` (`helm template <release> <url> --version <tag> -f values.yaml` for every release, output piped to `kube-state/.github/scripts/check-images.sh` — a copy of the helm-charts script from T033 with its bats cases in `kube-state/.github/tests/check-images.bats` — so a `values.yaml` override to a missing or `latest` image tag fails, Constitution Principle II) to `kube-state/.github/workflows/validate.yaml` (`data-model.md` ReleaseDefinition.chart version, ReleaseDefinition.overrides)
- [X] T045 [US3] Add job `e2e` to `kube-state/.github/workflows/validate.yaml`: create kind cluster (pinned action), `kubectl config rename-context kind-<name> docker-desktop`, run `./bring-up.sh` and assert `3 installed`, run again and assert `0 installed, 0 upgraded, 3 unchanged` and `helm history shophub -n shophub` has one revision (SC-003, research.md R10)
- [X] T046 [US3] Complete the "Bring-up" section of `kube-state/README.md`: prerequisites (Docker Desktop Kubernetes, helm ≥ 3.8, yq v4, jq, kubectl; winget command), the single command `./bring-up.sh`, env vars, exit codes table (link contract), expected output, re-run/update behaviour, how to reset to an empty cluster, note that Flux may be adopted later without changing release folders (FR-017, FR-018)
- [ ] T047 [US3] Open the kube-state PR (contains US1 + US3 kube-state work) and let every check run once; **before merging**, configure branch protection on `main` of `nikola031ue/kube-state-spec` via `gh api` (required checks `layout`, `schema`, `charts-exist`, `render`, `secrets-scan`, `unit`, `e2e`; linear history; no force-push or direct push; `enforce_admins: true`; 0 approvals per D-1) and record the command in `kube-state/README.md`; then get all checks green (incl. `e2e`), self-review (D-1), squash-merge (Constitution Principle II)

**Checkpoint**: All user stories complete and independently validated

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Repository governance and end-to-end validation

- [ ] T048 [P] Verify branch protection on `nikola031ue/helm-charts-spec` (set in T038): `gh api repos/nikola031ue/helm-charts-spec/branches/main/protection` lists the five required checks, linear history and `enforce_admins.enabled: true`; a direct `git push` of a test commit to `main` by the repo owner is rejected
- [ ] T049 [P] Verify branch protection on `nikola031ue/kube-state-spec` (set in T047) the same way as T048, expecting the seven required checks
- [ ] T050 Run quickstart.md Scenarios 1–7 on Docker Desktop after a Kubernetes reset; record timings (SC-002 < 15 min) and results in `shophub-specs/specs/001-iac-foundation/quickstart.md` under a new "Validation log" section
- [ ] T051 [P] Decide on renaming GitHub repos to `helm-charts` / `kube-state` (research.md R13); if renamed, update git remotes and branch-protection commands in both READMEs (OCI paths are unaffected)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: none
- **Foundational (Phase 2)**: after Setup — blocks all stories
- **US1 (Phase 3)**: after Foundational
- **US2 (Phase 4)**: after Foundational; T038 merge needs US1's helm-charts tasks (T014–T020) in the same PR; branch protection is applied inside T038 before the merge
- **US3 (Phase 5)**: T040–T043 and T046 after Foundational; T044, T045, T047 need US1 kube-state files (T021–T028) and US2 publish (T039); branch protection is applied inside T047 before the merge
- **Polish (Phase 6)**: after US2 (T048) / US3 (T049, T050)

### User Story Dependencies

- **US1 (P1)**: independent
- **US2 (P2)**: publishes the charts created in US1; its CI logic is testable on its own
- **US3 (P3)**: needs published charts from US2 for real installs; script logic is unit-tested independently with stubs

### Within Each User Story

- bats tests before the script they cover (recommended, Principle III)
- Chart/file content before the CI job that validates it
- Workflow file edits (`charts.yaml`, `validate.yaml`) are sequential — same file

### Parallel Opportunities

- T002–T005; T007–T010
- US1: T011–T017 and T021–T024 all in parallel (different files); kube-state and helm-charts halves in parallel
- US2: T029, T030 in parallel; US3 T040–T042 can proceed while US2 is in progress
- Polish: T048, T049, T051
- T048/T049 are verification only; branch protection itself is never deferred past the first merge (Principle II)

---

## Parallel Example: User Story 1

```bash
# Tests
Task: "bats tests for layout checker in helm-charts/.github/tests/check-layout.bats"
Task: "bats tests for layout+schema in kube-state/.github/tests/check-layout.bats and check-schema.bats"
Task: "bats tests for secrets scanner in kube-state/.github/tests/scan-secrets.bats"

# Content
Task: "shophub chart from helm create in helm-charts/charts/shophub/"
Task: "placeholder chart in helm-charts/charts/shop/"
Task: "placeholder chart in helm-charts/charts/shop-operator/"
Task: "placeholder chart in helm-charts/charts/shophub-discord/"
Task: "cluster.yaml in kube-state/clusters/local/cluster.yaml"
Task: "release files in kube-state/clusters/local/{shop-operator,shophub,shophub-discord}/"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1 + Phase 2
2. Phase 3 (US1) → both layouts exist and are CI-enforced on the open PRs
3. **STOP and VALIDATE on the branch**: quickstart.md Scenario 1 against the feature branches,
   `layout`/`schema`/`secrets-scan` green — the eliminatory requirement is met
4. The MVP is not merged alone: it lands on `main` together with US2 (helm-charts, T038) and
   US3 (kube-state, T047)

### Incremental Delivery

1. US1 → layouts correct and validated on the branches (eliminatory check safe)
2. US2 → branch protection on helm-charts, merge, charts published at `0.1.0`, public
3. US3 → branch protection on kube-state, merge, one-command bring-up proven on kind and
   Docker Desktop
4. Polish → branch protection verified + recorded validation

---

## Notes

- Commit messages follow Conventional Commits (e.g. `feat(charts): add shophub placeholder chart`, `ci: add publish job`)
- Never put secrets in any file; GHCR auth uses `GITHUB_TOKEN` only
- Mark every `.sh` executable in git (`git update-index --chmod=+x`) — Windows does not preserve the bit
- Do not add top-level folders to either repo beyond those in plan.md (Principle I)

## Implementation log (2026-11-21)

- Local validation ran in a throwaway container with the same pinned tools as CI (helm v3.17.3,
  yq v4.45.1, kubeconform v0.6.7, bats v1.11.1); results: helm-charts 52/52 bats, kube-state
  81/81 bats, all four charts `helm lint --strict` clean and kubeconform-valid. T006 (installing
  tools on the maintainer machine) is still open.
- Workflows were written in final form directly; the `noop` placeholder jobs of T007/T008 were
  never needed. Tool setup is a composite action `.github/actions/setup-tools/` in each repo.
- Added beyond the task text: a top-level-directory check in both layout scripts (only
  `charts/` resp. `clusters/` visible — Principle I), `.github/tests/helpers.bash` (kube-state
  fixture builder) and `.github/tests/stubs/helm` in helm-charts.
- GHCR returns `403` for packages that do not exist yet (verified live); the version check
  therefore treats *denied* as "not published" only for charts new in the PR (research.md R9).
- Bug found by the bring-up tests: `yq eval 'select(.kind == …)'` on multi-doc files can emit
  `---` separators; all multi-doc reads now use `yq eval-all '[…] | .[0] | …'`.
- Files are staged (executable bits set) on branch `001-iac-foundation` in both repos; nothing
  is committed or pushed. Remaining tasks need GitHub/GHCR or the local cluster: T038, T039,
  T047–T051.
