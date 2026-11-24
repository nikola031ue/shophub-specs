# Contract: Delivery

## Repository `shophub-spec` (folder `shophub`)

```text
shophub/
├── global.json                      # .NET SDK pin, Microsoft.Testing.Platform (as in shop)
├── ShopHub.slnx
├── src/ShopHub.Api/
│   ├── Program.cs, Settings.cs, ErrorHandling.cs, SecurityHeaders.cs, ForwardedHeaders.cs
│   ├── Accounts/        # owners, sessions, sign-in failures, registration limit
│   ├── Auth/            # cookie setup, XSRF, Data-Protection keys in PostgreSQL
│   ├── Shops/           # ShopRecords (SQL), Reconciler (hosted service), IKubeGateway + KubeGateway, ReasonTexts
│   ├── Endpoints/       # Accounts.cs, Shops.cs
│   └── Storage/Scripts/ # 001_initial.sql …
├── src/ShopHub.Web/     # Angular: auth/ (register, sign-in), shops/ (list, create, edit, delete, admin password)
├── tests/ShopHub.Api.Tests/
│   ├── Unit/            # validation, naming, reason mapping, state mapping
│   └── Integration/     # Postgres (Testcontainers), K3s (Testcontainers + operator CRDs), Api (WebApplicationFactory), Isolation
├── Dockerfile           # node build → sdk publish → aspnet chiseled, non-root, port 8080
├── README.md
└── .github/workflows/ci.yaml
```

Settings (environment only):

| Variable | Meaning |
|----------|---------|
| `DATABASE_URL` | From CNPG Secret `shophub-db-app`, key `uri` |
| `SHOPS_NAMESPACE` | `shops` |
| `SHOP_DOMAIN` | `shops.localhost` (for display only; URLs come from Shop status) |
| `TRUSTED_PROXY_CIDRS` | As for shops |
| `MAX_SHOPS_PER_OWNER` | default 5 |

Kubernetes access comes from the in-cluster ServiceAccount token. A kubeconfig is used only in
tests.

## CI (`.github/workflows/ci.yaml`, mirrors shop)

| Job | Trigger | Steps |
|-----|---------|-------|
| `lint` | PR, push main, tags | `dotnet format --verify-no-changes`, `ng lint` |
| `test` | PR, push main, tags | `dotnet test` (unit, Testcontainers PostgreSQL and k3s with the pinned `shop-operator` chart CRDs), `ng test` |
| `release` | `v*.*.*` tag | strict SemVer; tagged commit on `main`; refuse an existing version; push `ghcr.io/nikola031ue/shophub:X.Y.Z` |

Actions pinned by SHA, `permissions: {}` at the top. Branch protection (asked first): required
`lint` + `test`, strict, linear history, squash only, delete branches on merge.

## Chart `helm-charts/charts/shophub` 0.2.0 (replaces the 0.1.0 placeholder)

| Template | Content |
|----------|---------|
| `deployment.yaml` | 2 replicas, image `ghcr.io/nikola031ue/shophub:<appVersion>`, non-root, readiness `/healthz`, memory 128Mi/256Mi, env from values and the CNPG Secret |
| `service.yaml` | ClusterIP 80 → 8080 |
| `ingress.yaml` | class `traefik`, host `shophub.localhost`, no `tls` block (default certificate) |
| `serviceaccount.yaml` | `shophub` |
| `rbac.yaml` (new) | Role + RoleBinding **in `shopsNamespace` only**: `shophub.io` `shops`, `wallets`, `discordchannels`: `get list watch create patch delete` (server-side apply needs `create` and `patch`); `""` `secrets`: `get` only (used only for `<id>-admin`). No ClusterRole |
| `database.yaml` (new) | CNPG `Cluster` `shophub-db`, 1 instance, 1Gi |
| `hpa.yaml` | kept, disabled by default |
| `NOTES.txt`, `_helpers.tpl` | updated |

Values: `image.tag`, `host`, `shopsNamespace`, `shopDomain`, `trustedProxyCIDRs`,
`maxShopsPerOwner`. No secrets.

The shop admin password is fetched over the in-cluster Service of the shop
(`http://<id>.shops.svc.cluster.local/api/shop`), which the Shop NetworkPolicy allows from
namespace `shophub`.

## Rollout order (each step its own PR, CI green, merge asked first)

1. kube-state: `traefik` release + README mkcert steps + `bring-up.sh` precheck + allowlist.
2. shop-operator `v0.3.0` → chart `shop-operator` 0.4.0 → kube-state `helm.yaml` 0.4.0.
3. shop `v0.2.0` → kube-state `shopImage.tag: "0.2.0"`.
4. Live Phase 1 checks (quickstart 1–3); throwaway resources deleted.
5. shophub `v0.1.0` → chart `shophub` 0.2.0 → kube-state `clusters/local/shophub` (`helm.yaml`
   0.2.0, `values.yaml` image tag and host).
6. Quickstart 4–9.
