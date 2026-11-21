# Contract: `bring-up.sh` CLI

Location: `kube-state/bring-up.sh`. Bash ≥ 4. Run from the repository root.

## Usage

```text
./bring-up.sh [CLUSTER]          # CLUSTER defaults to "local"
./bring-up.sh --help
```

| Env var | Default | Meaning |
|---------|---------|---------|
| `HELM_TIMEOUT` | `5m` | `--timeout` for each `helm upgrade --install --wait` |
| `CRD_WAIT_TIMEOUT` | `120s` | Per-CRD timeout for `Established` (FR-023) |

Prerequisites checked at start (exit 2 if missing): `helm` ≥ 3.8, `kubectl`, `yq` v4
(mikefarah), `jq`.

## Behaviour

1. Load `clusters/<CLUSTER>/cluster.yaml`. Fail if missing or malformed.
2. Fail unless `kubectl config current-context` == `cluster.yaml: context` (FR-021).
   All later `helm`/`kubectl` calls pass `--kube-context <context>`.
3. For each release in `releaseOrder`, strictly in order (FR-020):
   1. Read chart URL, version, release name, namespace from `helm.yaml`.
   2. If deployed chart version, status (`deployed`) and values all match desired →
      print `unchanged`, skip (FR-019). Deployed version = `helm list` `.chart` minus the
      `<chart name>-` prefix (chart name = last segment of the OCI URL). Values are compared
      after `jq -S '. // {}'` on both sides, so an empty `values.yaml` matches Helm's `null`.
   3. Else `helm upgrade --install <release> <url> --version <tag> -n <ns>
      --create-namespace -f values.yaml --wait --timeout $HELM_TIMEOUT` →
      print `installed` or `upgraded`.
   4. Collect CRDs provided by the release; for each, wait for `Established` with
      `$CRD_WAIT_TIMEOUT` (FR-023). None → no wait.
4. Print a summary table: release, chart version, namespace, result.

The script never runs `kubectl apply/create/edit/delete` (FR-022, Principle V).

## Output (stdout)

```text
[cluster] local (context docker-desktop)
[1/3] shop-operator   0.1.0  shop-operator-system  installed
[1/3] shop-operator   crds: none
[2/3] shophub         0.1.0  shophub               installed
[2/3] shophub         crds: none
[3/3] shophub-discord 0.1.0  shophub               installed
[3/3] shophub-discord crds: none
done: 3 installed, 0 upgraded, 0 unchanged
```

Errors go to stderr, prefixed `ERROR:` and naming the release (and CRD, chart version or
context where relevant).

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | All releases installed/upgraded/unchanged |
| 1 | A release failed to install/upgrade (later releases not attempted) |
| 2 | Prerequisite tool missing or wrong version |
| 3 | `cluster.yaml` / `helm.yaml` missing or invalid |
| 4 | Current context does not match `cluster.yaml` |
| 5 | A CRD did not become `Established` within `CRD_WAIT_TIMEOUT` |
| 6 | Chart version not found / registry unreachable |
