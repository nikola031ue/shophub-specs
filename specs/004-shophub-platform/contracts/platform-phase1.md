# Contract: Phase 1 platform changes

Delivered and deployed before any ShopHub app work (spec Dependencies). CRDs
(`shophub.io/v1alpha1`) are unchanged.

## Entry point (kube-state release `traefik`)

| Item | Value |
|------|-------|
| Chart | upstream `oci://ghcr.io/traefik/helm/traefik` **42.3.1** (Traefik v3.7.17), exact version in `.github/upstream-charts.yaml` |
| Namespace | `traefik` |
| Service | `LoadBalancer`, `externalTrafficPolicy: Local`, ports 80 (`web`) and 443 (`websecure`) |
| Redirect | `web` → `websecure`, permanent |
| IngressClass | `traefik`, default |
| Default certificate | Secret `shophub-tls` (type `kubernetes.io/tls`) in `traefik`, created by hand from mkcert (README). Listed in `cluster.yaml` `requiredSecrets`; `bring-up.sh` checks it exists before installing anything and otherwise prints the steps and exits **7** |
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
| `--trusted-proxy-host` | `trustedProxyHost` | `traefik-proxies.traefik.svc.cluster.local` | Passed to the app as `TRUSTED_PROXY_HOST` (research R5) |
| `--ingress-namespace` | `ingressNamespace` | `traefik` | First namespace allowed by the Shop NetworkPolicy |
| `--shophub-namespace` | `shophubNamespace` | `shophub` | Second namespace allowed by the Shop NetworkPolicy |

### Reconcile additions (per Shop, owned children, server-side apply)

1. `Ingress <shop>`: host `<shop>.<shop-domain>`, path `/` (Prefix) → Service `<shop>` port 80.
   No `tls` block; the entry point's default certificate covers the host.
2. `NetworkPolicy <shop>`: selects the Shop's pods. It allows ingress on TCP 80 only from
   namespaces `traefik` and `shophubNamespace`.
3. Deployment env `TRUSTED_PROXY_HOST`.
4. `status.url = https://<shop>.<shop-domain>`.

The Shop CRD gains one validation rule: `metadata.name` must not contain a dot (FR-021).

Existing 0.2.x Shops are updated by reconciliation alone, if any exist in `shops`.

### RBAC (replaces the manager ClusterRole)

| Object | Namespace | Rules |
|--------|-----------|-------|
| Role `shop-operator-manager` | `shops` | the 0.2.1 rules (Shops, Wallets, DiscordChannels incl. status and finalizers; Deployments; Services; Secrets; CNPG Clusters; Redis; StatefulSets read), plus `networking.k8s.io` `ingresses` and `networkpolicies` (`get list watch create update patch delete`) |
| Role `shop-operator-platform-channels` | operator namespace | `discordchannels` (`get list watch update patch`, status, finalizers) and `secrets` (`get list watch create update patch delete`): the platform alerts channel, its webhook Secret and the Discord token Secret (research R4) |
| Role `…-leader-election` | operator namespace | unchanged |
| Namespace `shops` | – | created by the chart, label `shophub.io/shops: "true"` |

No ClusterRole or ClusterRoleBinding remains in the chart. Shops and Wallets are reconciled only
in `shops`; DiscordChannels in `shops` and in the operator namespace. Everything else is ignored.

### Platform alerts channel move (kube-state)

The `shophub-discord` release moves from namespace `shophub` to `shop-operator-system`
(`clusters/local/shophub-discord/helm.yaml` `targetNamespace`). The chart itself is unchanged.
A Helm release can't change namespace, so the move is an uninstall and a reinstall, done in the
order in quickstart Scenario 1 (step 1): the old CR is deleted while operator 0.2.1 still
watches `shophub`, so its finalizer removes the old Discord channel. Bring-up then installs the
release in the new namespace, and 0.3.0 creates a new channel there.

### Acceptance (envtest + live)

- The Ingress and NetworkPolicy are created, owned, and restored after a manual edit. Status `url`
  is HTTPS.
- A Shop in a namespace other than `shops` gets no children and no status; a DiscordChannel in
  the operator namespace is reconciled; one in any other namespace is not.
- Live: `kubectl auth can-i get secrets -n <ns> --as system:serviceaccount:shop-operator-system:shop-operator`
  answers **no** for every namespace except `shops` (and the token Secret by name).

## shop 0.2.0

| Change | Behaviour |
|--------|-----------|
| `TRUSTED_PROXY_HOST` (optional DNS name) | Resolved every 30 s. `X-Forwarded-For` and `X-Forwarded-Proto` are honoured only when the TCP peer is one of its addresses (the Traefik pods); `ForwardLimit = 1`. Unset or a failed lookup means no proxy is trusted. Other admitted peers (ShopHub) can't set the client address (research R5) |
| Admin lockout | Per (account, client address) 5 failures / 15 min, plus a shop-wide ceiling of 50 failures / 15 min (data-model.md). 429 with `Retry-After` |
| Order rate limit | Unchanged rule (10/min); the key is now the real client address |
| Cookies | Session and anti-forgery cookies always `Secure` |
| `GET /api/shop` | `{name, adminPasswordChanged}`; `adminPasswordChanged` is false while the operator-generated password is still the admin password |

Tests on both stores:

- forwarded header honoured from a trusted peer and ignored from an untrusted one;
- per-(account, address) lockout plus the ceiling; failures for one account from an address
  don't lock a different account from the same address;
- two client addresses each get their own order limit;
- `Secure` flag set;
- `adminPasswordChanged` flips after a password change.
