# Contract: kube-state files

Applies to `kube-state/clusters/<cluster>/`. Validated in kube-state CI (FR-016).

## Repository tree (Requirement 5.3, immutable)

```text
kube-state
├── README.md
├── bring-up.sh                 # root file, not a folder (research R4)
├── .github/                    # CI only (research R11)
└── clusters/
    └── local/
        ├── cluster.yaml
        ├── shop-operator/   { helm.yaml, values.yaml }
        ├── shophub/         { helm.yaml, values.yaml }
        └── shophub-discord/ { helm.yaml, values.yaml }
```

Every release folder contains **exactly** `helm.yaml` and `values.yaml`.

## `cluster.yaml`

Not a Kubernetes object (no `apiVersion`/`kind`).

```yaml
name: local                      # == folder name
context: docker-desktop          # bring-up refuses any other current-context
distribution: docker-desktop
kubernetesVersion: "1.31"        # informational
releaseOrder:                    # every release folder exactly once, deps first
  - shop-operator
  - shophub
  - shophub-discord
```

## `helm.yaml`

Exactly two YAML documents, in this order. Example for `shophub`:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: shophub                  # == release folder name
  namespace: flux-system
spec:
  interval: 10m
  url: oci://ghcr.io/nikola031ue/charts/shophub
  ref:
    tag: 0.1.0                   # exact SemVer; no ranges, no latest
  layerSelector:
    mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip
    operation: copy
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: shophub                  # == release folder name
  namespace: flux-system
spec:
  interval: 10m
  releaseName: shophub           # == release folder name
  targetNamespace: shophub
  install:
    createNamespace: true
  chartRef:
    kind: OCIRepository
    name: shophub
  dependsOn:
    - name: shop-operator
  valuesFrom:
    - kind: ConfigMap
      name: shophub-values       # generated from values.yaml when Flux is adopted (R2)
      valuesKey: values.yaml
```

Fields read by `bring-up.sh`: `OCIRepository.spec.url`, `OCIRepository.spec.ref.tag`,
`HelmRelease.spec.releaseName`, `HelmRelease.spec.targetNamespace`. All other fields exist for
Flux and are validated but ignored by the script.

**Validation rules**
- Both objects have `metadata.name` == folder name; `releaseName` == folder name.
- `spec.url` starts with `oci://ghcr.io/nikola031ue/charts/`.
- `spec.ref.tag` matches strict SemVer 2.0 and is anonymously pullable.
- Every `dependsOn[].name` is another release folder, listed earlier in `releaseOrder`.
- `valuesFrom[0].name` == `<folder>-values`.

## `values.yaml`

Plain Helm value overrides for the release's chart. A YAML mapping, `{}` allowed.
Must render with `helm template <url> --version <tag> -f values.yaml`.
Must not contain secret values; CI rejects keys matching
`(?i)(password|token|secret|privatekey|apikey)` with a non-empty literal value unless the
key ends in `SecretName`/`existingSecret` (references by name are allowed).
