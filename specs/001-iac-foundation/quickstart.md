# Quickstart & Validation: IaC Foundation

Proves the feature end to end. Commands assume Git Bash on Windows (or any Bash shell).

## Prerequisites

- Docker Desktop with Kubernetes enabled (context `docker-desktop`).
- `helm` ≥ 3.8, `yq` v4 (mikefarah), `jq`, `kubectl`, `gh` — e.g.
  `winget install Helm.Helm MikeFarah.yq jqlang.jq`.
- Clones of `helm-charts` and `kube-state` side by side (only `kube-state` is needed for
  bring-up).

## One-time setup

1. **Branch protection** on `main` in both repos, applied while each repo's first PR is open
   (after its checks ran once) and **before** it is merged: required checks from
   [contracts/ci-checks.md](contracts/ci-checks.md), linear history, no direct pushes,
   `enforce_admins: true` (applies the rules to you as repo owner), 0 required approvals (D-1).
2. **First publish**: merge the helm-charts skeleton PR; wait for `publish`.
3. **Make packages public**: GitHub → Profile → Packages → `charts/<name>` → Package
   settings → Change visibility → Public, for each of the four charts. Re-run
   `verify-public`; it must pass.

## Scenario 1 — Layout matches 5.3 (US1, SC-001)

```bash
cd helm-charts && find . -path ./.git -prune -o -path ./.github -prune -o -print | sort
cd ../kube-state && find . -path ./.git -prune -o -path ./.github -prune -o -print | sort
```

Expected: exactly the trees in `docs/ShopHub.md` §5.3 for the in-scope charts/releases, plus
`charts/shophub-discord/`, `bring-up.sh` and `.helmignore`/`.gitkeep` files.
The `layout` CI job reports success in both repos.

## Scenario 2 — Charts are published and pullable anonymously (US2, SC-005)

```bash
helm registry logout ghcr.io 2>/dev/null
for c in shop shophub shop-operator shophub-discord; do
  helm show chart oci://ghcr.io/nikola031ue/charts/$c --version 0.1.0 | grep '^version'
done
```

Expected: four lines `version: 0.1.0`, no auth error.

## Scenario 3 — CI blocks bad changes (US2, SC-004)

Open throwaway PRs in `helm-charts` and confirm each is blocked:

| Change | Failing check |
|--------|---------------|
| Edit `charts/shop/values.yaml` without bumping version | `version` |
| Introduce a template syntax error | `lint` / `template` |
| Delete `charts/shophub/templates/hpa.yaml` | `layout` |

Close the PRs without merging.

## Scenario 4 — Bring-up on an empty cluster (US3, SC-002)

1. Docker Desktop → Settings → Kubernetes → **Reset Kubernetes Cluster**.
2. Run:

   ```bash
   cd kube-state
   time ./bring-up.sh
   helm list -A
   ```

Expected: output as in [contracts/bring-up-cli.md](contracts/bring-up-cli.md) with
`3 installed`; `helm list -A` shows three releases `deployed` at `0.1.0`; the `shophub` pod
is `Running`; total time well under 15 minutes.

## Scenario 5 — Idempotent re-run (SC-003)

```bash
./bring-up.sh
```

Expected: `done: 0 installed, 0 upgraded, 3 unchanged`; `helm history shophub -n shophub`
still shows a single revision.

## Scenario 6 — Change propagates to one release only (US3 scenario 3)

Set `replicaCount: 2` in `clusters/local/shophub/values.yaml`, run `./bring-up.sh`.
Expected: `shophub upgraded`, the other two `unchanged`. Revert afterwards.

## Scenario 7 — Safety checks (edge cases, FR-021, FR-023)

| Action | Expected |
|--------|----------|
| `kubectl config use-context <other>` then `./bring-up.sh` | exit 4, `ERROR: current context … expected docker-desktop` |
| Set a non-existent tag in `shophub/helm.yaml` | exit 6 naming `shophub` and the version; later releases not attempted |
| CRD wait (placeholders) | each release prints `crds: none` |

Switch back with `kubectl config use-context docker-desktop`.
