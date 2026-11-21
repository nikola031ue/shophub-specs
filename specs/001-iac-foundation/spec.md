# Feature Specification: IaC Foundation (Requirement 5.3)

**Feature Branch**: `001-iac-foundation`

**Created**: 2026-11-21

**Status**: Draft

**Input**: User description: "IaC foundation per Requirement 5.3. Create the helm-charts repo
skeleton (README, charts/shop, charts/shophub with the exact template files listed in 5.3, plus
charts/shop-operator) and the kube-state repo skeleton (README, clusters/local/cluster.yaml, and
shop-operator, shophub, shophub-discord folders each with helm.yaml and values.yaml). Charts are
published as OCI artifacts and referenced from kube-state helm.yaml by OCI name. A maintainer
must be able to bring up the platform on an empty local cluster from kube-state alone. Charts
are placeholders now and get real content in later features."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Repository layouts match Requirement 5.3 exactly (Priority: P1)

A reviewer (course examiner or the maintainer self-reviewing) opens the `helm-charts` and
`kube-state` repositories and finds exactly the folder and file structure that Requirement 5.3
prescribes, so the project passes the eliminatory check before anything else is evaluated.

**Why this priority**: Requirement 5.3 is eliminatory. If the layout is wrong, the project is
not reviewed at all, so nothing else in this or later features has value without it.

**Independent Test**: Compare the file trees of both repositories against the trees in
Requirement 5.3 (`docs/ShopHub.md`); every prescribed path exists and no path breaks the
prescribed pattern.

**Acceptance Scenarios**:

1. **Given** the `helm-charts` repository, **When** a reviewer lists its contents, **Then**
   it contains `README.md` and `charts/` with `shop/`, `shophub/`, `shop-operator/` and
   `shophub-discord/`, each containing `Chart.yaml`, `values.yaml` and `templates/`.
2. **Given** `charts/shophub/`, **When** a reviewer lists it, **Then** it additionally
   contains `charts/` and its `templates/` contains exactly `NOTES.txt`, `_helpers.tpl`,
   `deployment.yaml`, `hpa.yaml`, `ingress.yaml`, `service.yaml` and `serviceaccount.yaml`.
3. **Given** the `kube-state` repository, **When** a reviewer lists its contents, **Then** it
   contains `README.md` and `clusters/local/` with `cluster.yaml` and the folders
   `shop-operator/`, `shophub/` and `shophub-discord/`, each containing exactly `helm.yaml`
   and `values.yaml`.
4. **Given** either repository, **When** a reviewer looks for alternative or parallel
   layouts (other top-level folders for charts or cluster state), **Then** none exist.

---

### User Story 2 - Charts are published as versioned OCI artifacts (Priority: P2)

When the maintainer merges a chart change to `helm-charts`, each changed chart is validated
and published to an OCI registry under a semantic version, so `kube-state` can refer to an
exact, immutable chart version by name.

**Why this priority**: `kube-state` references charts by OCI name; without published
artifacts there is nothing for the cluster state to point at, but the layout (P1) is still
valuable on its own for the eliminatory check.

**Independent Test**: Open a PR that changes one chart, observe validation run and block
merge on failure; merge it and confirm the new chart version can be pulled from the registry
by its OCI name without any credentials.

**Acceptance Scenarios**:

1. **Given** a PR to `helm-charts` with a broken chart, **When** CI runs, **Then** the PR
   cannot be merged.
2. **Given** a PR that changes a chart without increasing its version, **When** CI runs,
   **Then** the PR cannot be merged.
3. **Given** a merged change to a chart with a new version, **When** publishing completes,
   **Then** that version is pullable from the registry by its OCI name.
4. **Given** a chart version that is already published, **When** publishing runs again for
   the same version, **Then** the existing artifact is not overwritten.

---

### User Story 3 - Bring up the platform on an empty cluster from kube-state alone (Priority: P3)

A maintainer with an empty local Kubernetes cluster, a clone of `kube-state` and the tools
named in its README follows the README and ends up with every release defined under
`clusters/local/` installed at the chart versions pinned in its `helm.yaml`, configured with
its `values.yaml` — without cloning `helm-charts` or any other repository.

**Why this priority**: It proves the whole chain (layout → published charts → cluster state)
works end to end and is what later features build on, but it depends on P1 and P2.

**Independent Test**: Reset the local cluster to empty, follow the `kube-state` README, and
verify that each release from `clusters/local/` is installed and healthy at its pinned
version.

**Acceptance Scenarios**:

1. **Given** an empty local cluster and a fresh clone of `kube-state`, **When** the
   maintainer follows the README bring-up procedure, **Then** releases `shop-operator`,
   `shophub` and `shophub-discord` are installed and report a successful status.
2. **Given** a platform already brought up, **When** the maintainer runs the procedure again
   with no changes, **Then** nothing changes and no error occurs.
3. **Given** a platform already brought up, **When** the maintainer changes a pinned version
   or a value in `kube-state` and runs the procedure, **Then** only the affected release is
   updated.
4. **Given** the bring-up procedure, **When** it is executed, **Then** no manifest is
   applied to the cluster other than through the releases defined in `kube-state`.

---

### Edge Cases

- A `helm.yaml` points at a chart version that does not exist in the registry: bring-up stops
  for that release with an error naming the release and the missing version.
- The `shop-operator` release succeeds but one of its CRDs never becomes Established:
  bring-up stops before `shophub-discord` and names the CRD (FR-023).
- The registry is unreachable during bring-up: bring-up fails with a clear message and leaves
  already-installed releases untouched.
- The current Kubernetes context is not the local cluster described in `cluster.yaml`:
  bring-up refuses to proceed.
- A release folder is missing `helm.yaml` or `values.yaml`, or the file is malformed:
  validation in the `kube-state` CI fails and the change cannot be merged.
- A chart is renamed or a folder moved in a way that breaks the 5.3 layout: CI in the
  affected repository fails.
- A secret value is placed in a `values.yaml`: it must be caught in review / CI and never
  merged (Constitution Principle VI).

## Requirements *(mandatory)*

### Functional Requirements

**helm-charts repository**

- **FR-001**: The `helm-charts` repository MUST contain a root `README.md` and a `charts/`
  directory, and no other top-level location for charts.
- **FR-002**: `charts/` MUST contain `shop/`, `shophub/`, `shop-operator/` and
  `shophub-discord/`; each MUST contain `Chart.yaml`, `values.yaml` and `templates/`.
- **FR-003**: `charts/shophub/` MUST additionally contain a `charts/` directory, and its
  `templates/` MUST contain exactly `NOTES.txt`, `_helpers.tpl`, `deployment.yaml`, `hpa.yaml`,
  `ingress.yaml`, `service.yaml` and `serviceaccount.yaml`.
- **FR-004**: The `bitcoin/` and `celestia/` charts shown in the Requirement 5.3 example are
  out of scope and MUST NOT be created; the folder pattern they illustrate is followed by all
  charts that are created.
- **FR-005**: Every chart MUST be installable on its own with its default values, even while
  it is a placeholder (installing it creates a successful release).
- **FR-006**: Every chart MUST carry a semantic version; any change to a chart's contents
  MUST increase that version.
- **FR-007**: Every PR to `helm-charts` MUST run validation (lint and render of every chart,
  layout check, version-bump check) and MUST NOT be mergeable if it fails.
- **FR-008**: On merge to `main`, every chart whose version is not yet published MUST be
  published to the OCI registry; already-published versions MUST NOT be overwritten.
- **FR-009**: The `helm-charts` README MUST list each chart, its purpose, its current status
  (placeholder or real), and the OCI name under which it is published.

**kube-state repository**

- **FR-010**: The `kube-state` repository MUST contain a root `README.md` and `clusters/local/`
  with `cluster.yaml` and the release folders `shop-operator/`, `shophub/` and
  `shophub-discord/`, each with exactly `helm.yaml` and `values.yaml`.
- **FR-011**: `cluster.yaml` MUST describe the cluster's metadata: at least its name, the
  Kubernetes context it corresponds to, and its distribution (local Docker Desktop
  Kubernetes).
- **FR-012**: Each `helm.yaml` MUST identify its chart by OCI name and an exact chart version
  (no ranges, no "latest"), plus the release name and target namespace.
- **FR-013**: Each `values.yaml` MUST contain only value overrides for its release and MUST
  NOT contain secrets; secrets are referenced by name only (Constitution Principle VI).
- **FR-014**: `shophub-discord/helm.yaml` MUST reference the `shophub-discord` chart. That
  chart's purpose is to define exactly one Discord notification channel (a `DiscordChannel`
  resource) for platform- and cluster-level alerts (Requirement 4.1), separate from the
  per-Shop channels. It depends on the resource types installed by `shop-operator`. In this
  feature it is a placeholder; the channel resource itself is added in the operator feature.
- **FR-015**: `kube-state` MUST hold platform state only; it MUST NOT contain user-created
  Shop, Wallet, DiscordChannel or database resources (Constitution Principle V).
- **FR-016**: Every PR to `kube-state` MUST run validation (layout check, each `helm.yaml`
  well-formed and referencing a published chart version, each `values.yaml` renders against
  its chart) and MUST NOT be mergeable if it fails.

**Bring-up**

- **FR-017**: The `kube-state` README MUST document prerequisites and a single bring-up
  procedure that, starting from an empty local cluster and a clone of `kube-state` only,
  installs every release under `clusters/local/`.
- **FR-018**: Bring-up MUST be a one-shot procedure run by the maintainer that reads only
  `kube-state` and installs/upgrades its releases. Continuous reconciliation by a GitOps
  controller (Argo CD / Flux) is out of scope for this feature and MAY be added in a later
  feature, provided it keeps the Requirement 5.3 layout unchanged (Constitution Principle I).
- **FR-019**: Bring-up MUST be idempotent: re-running it with unchanged `kube-state` makes no
  changes; re-running after a change updates only affected releases.
- **FR-020**: Bring-up MUST install releases in the fixed order `shop-operator` →
  `shophub` → `shophub-discord`, and MUST NOT start a release until the previous one has
  reported success.
- **FR-021**: Bring-up MUST refuse to run against a Kubernetes context other than the one
  named in `cluster.yaml`.
- **FR-022**: Bring-up MUST change cluster state only by installing/upgrading the releases
  defined in `kube-state`; no separately applied manifests (Constitution Principle V).
- **FR-023**: The bring-up script MUST wait for the shop-operator CRDs to be Established
  before installing `shophub-discord`. The CRDs waited on are those the installed
  `shop-operator` release provides (none while it is a placeholder, so the wait passes
  immediately). If any CRD is not Established within a bounded timeout, bring-up MUST stop
  before installing `shophub-discord` and name the CRD that is not ready.

### Key Entities

- **Chart**: a packaged, versioned deployable unit in `helm-charts/charts/<name>/`; has a name,
  semantic version, default values, templates, and a publication status in the registry.
- **Published chart artifact**: an immutable chart version in the OCI registry, addressed by
  OCI name + version.
- **Cluster definition** (`cluster.yaml`): metadata of one target cluster (name, context,
  distribution).
- **Release definition** (`helm.yaml` + `values.yaml` in a release folder): which chart
  version is installed under which release name and namespace, and with which overrides.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of the paths prescribed by Requirement 5.3 for the in-scope charts and
  releases exist, and 0 paths deviate from the prescribed pattern, verified by an automated
  layout check in both repositories.
- **SC-002**: A maintainer can go from an empty local cluster to all three releases reported
  healthy in under 15 minutes by following only the `kube-state` README, on the first attempt.
- **SC-003**: Running the bring-up procedure a second time with no changes results in 0
  changed resources.
- **SC-004**: 100% of PRs that break chart validity, the layout, or the version-bump rule are
  blocked from merging.
- **SC-005**: Every chart version referenced from `kube-state` can be pulled from the
  registry anonymously.

## Assumptions

- The two sibling repositories `helm-charts` and `kube-state` already exist (each with only a
  README) and are reused; this feature fills them.
- The OCI registry is public so that an empty cluster and anonymous CI jobs can pull charts
  without credentials; publishing uses CI credentials stored in GitHub Secrets. The specific
  registry is chosen in planning.
- Placeholder charts install a minimal, harmless release (e.g., no workloads or a trivial
  one); real content arrives in later features. The `shop` chart is published but not
  installed from `kube-state`, because Shops are runtime state created via the operator.
- The `shophub-discord` placeholder renders no `DiscordChannel` resource yet, so it installs
  even though the placeholder `shop-operator` does not install CRDs; the fixed bring-up order
  already guarantees the CRDs exist once real content arrives.
- The `shophub-discord` chart and release are platform state (one maintainer-owned channel),
  so keeping them in git does not conflict with Principle V, which excludes only
  user-created DiscordChannel resources.
- The `bitcoin`/`celestia` exclusion and the addition of `shophub-discord` both follow the
  5.3 folder pattern; 5.3 shows example charts, not a closed list.
- Platform dependencies not listed here (observability stack, database operators, GitOps
  tooling) are added in later features as additional release folders following the same
  pattern; they are not needed to install the placeholder releases.
- "From kube-state alone" means no other repository is cloned; it does not exclude tool
  prerequisites listed in the README or secrets that later features may require to be
  created out-of-band (none are needed for placeholders).
- Only the `local` cluster is in scope.
