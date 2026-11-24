# Contract: Phase 1 platform changes

Delivered and deployed before any ShopHub app work (spec Dependencies). CRDs
(`shophub.io/v1alpha1`) are unchanged.

## Entry point (kube-state release `traefik`)

| Item | Value |
|------|-------|
| Chart | upstream `oci://ghcr.io/traefik/helm/traefik`, exact version in `.github/upstream-charts.yaml` |
| Namespace | `traefik` |
| Service | `LoadBalancer`, `externalTrafficPolicy: Local`, ports 80 (`web`) and 443 (`websecure`) |
| Redirect | `web` → `websecure`, permanent |
| IngressClass | `traefik`, default |
| Default certificate | Secret `shophub-tls` (type `kubernetes.io/tls`) in `traefik`, created by hand from mkcert (README); `bring-up.sh` stops with the steps if it is missing |
| Hosts | `shophub.localhost` (ShopHub), `<shop>.shops.localhost` (shops) |
| Proxy discovery | Headless Service `traefik-proxies` (`clusterIP: None`) selecting the Traefik pods, created through the chart's `extraObjects`. ShopHub resolves it by DNS to trust forwarded headers only from Traefik (research R7) |
| Release order | `cnpg`, `redis-operator`, **`traefik`**, `shop-operator`, `shophub`, `shophub-discord` |

## shop-operator 0.3.0 (chart `shop-operator` 0.4.0)

### New flags and chart values

| Flag | Chart value | Default | Meaning |
|------|-------------|---------|---------|
| `--watch-namespace` | `shopsNamespace` | `shops` | The only namespace reconciled and cached |
| `--shop-domain` | `shopDomain` | `shops.localhost` | Host suffix: `<shop>.<shop-domain>` |
| `--ingress-class` | `ingressClassName` | `traefik` | `spec.ingressClassName` of every Shop Ingress |
| `--trusted-proxy-cidrs` | `trustedProxyCIDRs` | `10.244.0.0/16` | Passed to the app as `TRUSTED_PROXY_CIDRS` |
| – | `shophubNamespace` | `shophub` | Second namespace allowed by the Shop NetworkPolicy |

### Reconcile additions (per Shop, owned children, server-side apply)

1. `Ingress <shop>`: host `<shop>.<shop-domain>`, path `/` (Prefix) → Service `<shop>` port 80.
   No `tls` block; the entry point's default certificate covers the host.
2. `NetworkPolicy <shop>`: selects the Shop's pods. It allows ingress on TCP 80 only from
   namespaces `traefik` and `shophubNamespace`.
3. Deployment env `TRUSTED_PROXY_CIDRS`.
4. `status.url = https://<shop>.<shop-domain>`.

Existing 0.2.x Shops are updated by reconciliation alone, if any exist in `shops`.

### RBAC (replaces the manager ClusterRole)

| Object | Namespace | Rules |
|--------|-----------|-------|
| Role `shop-operator-manager` | `shops` | the 0.2.1 rules (Shops, Wallets, DiscordChannels incl. status and finalizers; Deployments; Services; Secrets; CNPG Clusters; Redis; StatefulSets read), plus `networking.k8s.io` `ingresses` and `networkpolicies` (`get list watch create update patch delete`) |
| Role `shop-operator-discord-token` | operator namespace | `get` on Secret `resourceNames: [<discord token Secret>]` |
| Role `…-leader-election` | operator namespace | unchanged |
| Namespace `shops` | – | created by the chart, label `shophub.io/shops: "true"` |

No ClusterRole or ClusterRoleBinding remains in the chart. Shops in other namespaces are ignored.

### Acceptance (envtest + live)

- The Ingress and NetworkPolicy are created, owned, and restored after a manual edit. Status `url`
  is HTTPS.
- A Shop in a namespace other than `shops` gets no children and no status.
- Live: `kubectl auth can-i get secrets -n <ns> --as system:serviceaccount:shop-operator-system:shop-operator`
  answers **no** for every namespace except `shops` (and the token Secret by name).

## shop 0.2.0

| Change | Behaviour |
|--------|-----------|
| `TRUSTED_PROXY_CIDRS` (optional, comma-separated CIDRs) | `X-Forwarded-For` and `X-Forwarded-Proto` are honoured only when the TCP peer is inside one of these networks; `ForwardLimit = 1`. Unset or empty means forwarded headers are ignored. Invalid CIDRs stop the start with a message naming the variable |
| Admin lockout | Per client address 5 failures / 15 min, plus a shop-wide ceiling of 50 failures / 15 min (data-model.md). 429 with `Retry-After` |
| Order rate limit | Unchanged rule (10/min); the key is now the real client address |
| Cookies | Session and anti-forgery cookies always `Secure` |
| `GET /api/shop` | `{name, adminPasswordChanged}`; `adminPasswordChanged` is false while the operator-generated password is still the admin password |

Tests on both stores:

- forwarded header honoured from a trusted peer and ignored from an untrusted one;
- per-address lockout plus the ceiling;
- two client addresses each get their own order limit;
- `Secure` flag set;
- `adminPasswordChanged` flips after a password change.
