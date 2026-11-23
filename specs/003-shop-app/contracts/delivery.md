# Contract: Delivery (repositories, CI, release, rollout)

## Order of delivery (spec Clarifications)

1. **Operator 0.2.0** ([operator-0.2.0.md](operator-0.2.0.md)). Three PRs, each with green CI and
   merged only after the maintainer approves:
   1. `shop-operator-spec`: code and tests, then tag `v0.2.0`;
   2. `helm-charts-spec`: chart `shop-operator` `0.3.0`;
   3. `kube-state-spec`: helm.yaml → `0.3.0`.

   After step 3, a live 0.1.0 shop on the local cluster must upgrade automatically.
2. **App**: `shop-spec` repository → tag `v0.1.0`.
3. **Rollout**: `kube-state-spec` PR setting the Shop image to the app (research R15).

## `shop` repository (`nikola031ue/shop-spec`)

```text
shop/
├── Shop.slnx  global.json  Directory.Build.props  Directory.Packages.props
├── src/Shop.Api/         # .NET 10 minimal API; wwwroot filled by the image build
│   ├── Program.cs  Settings.cs
│   ├── Endpoints/        # Public.cs, Admin.cs
│   ├── Auth/             # AdminAuth.cs (bootstrap, lockout, cookie), StoreXmlRepository.cs
│   ├── Domain/           # Item, Order, validation, Money
│   └── Storage/          # IShopStore.cs, Postgres/, Redis/ (incl. Lua scripts as resources)
├── src/Shop.Web/         # Angular app (storefront, cart, admin)
├── tests/Shop.Api.Tests/
│   ├── Unit/             # validation, settings, lockout timing
│   └── Integration/      # ShopStoreContract (abstract) + Postgres/Redis subclasses, ApiTests
├── Dockerfile  .dockerignore  .gitignore  README.md
└── .github/workflows/ci.yaml
```

## CI (`.github/workflows/ci.yaml`), mirroring shop-operator

| Job | Trigger | Steps | Required on `main` |
|-----|---------|-------|--------------------|
| `lint` | PR, push to main | `dotnet format --verify-no-changes`; `npm ci && npx ng lint` | ✓ |
| `test` | PR, push to main | `dotnet test` (unit + Testcontainers PostgreSQL 17 and Redis 7.2, images pinned by tag); `npx ng test --watch=false` | ✓ |
| `release` | tag `v*.*.*` | refuse non-SemVer tags; `docker build`; push `ghcr.io/nikola031ue/shop:X.Y.Z` with `GITHUB_TOKEN`; never `latest` | – |

- Actions pinned by full commit SHA, as in shop-operator.
- Before the first merge: branch protection on `main` (required `lint` and `test`, PR required, no
  force pushes, linear history), squash-only and auto-delete branches. These are GitHub settings,
  so **ask the maintainer first**.
- Making the GHCR package `shop` public is also asked first.

## Image

- One image holds the API and the built Angular files (FR-023).
- Non-root chiseled runtime, every base image pinned by digest.
- `ASPNETCORE_HTTP_PORTS=80`.
- OCI labels `org.opencontainers.image.source` and `version`.

## Rollout (kube-state)

```yaml
# clusters/local/shop-operator/values.yaml (added)
shopImage:
  repository: ghcr.io/nikola031ue/shop
  tag: "0.1.0"
```

- `./bring-up.sh` upgrades only `shop-operator`; every Shop rolls to the app.
- Later app releases are a one-line tag change in the same file.
- `helm-charts/charts/shop` is not changed (FR-026).
