# Contract: shop-operator 0.2.0 changes (Phase 1)

Changes against [002 contracts/crds.md](../../002-shop-operator/contracts/crds.md) and the 002
resource builders. The CRDs (`shophub.io/v1alpha1`) are **unchanged**. When this ships, 002's
`contracts/crds.md` gets a "0.2.0" note that links here.

## Derived resources per Shop

| Resource | When | Name | Content | Owner |
|----------|------|------|---------|-------|
| Secret | every Shop | `<shop>-admin` | key `password`: 24 random URL-safe chars | Shop (controller) |
| Secret | `database: light` | `<shop>-redis-auth` | key `password`: 32 random URL-safe chars | Shop (controller) |
| OT `Redis` (changed) | `database: light` | `<shop>-redis` | see below | Shop (controller) |

Both Secrets carry `app.kubernetes.io/managed-by=shop-operator` plus the Shop's
`shophub.io/owner` / `shophub.io/shop` labels. They are created once with `Create`. They are never
updated with new values while they exist.

## Application container contract (what the app may rely on)

| Env | `standard` | `light` | Source |
|-----|-----------|---------|--------|
| `SHOP_NAME` | ✓ | ✓ | Shop `spec.shopName` (unchanged) |
| `WALLET_ADDRESS` | ✓ | ✓ | Wallet status (unchanged) |
| `DATABASE_URL` | ✓ | – | `secretKeyRef` `<shop>-db-app` / `uri` (unchanged) |
| `REDIS_URL` | – | ✓ | `redis://<shop>-redis.<ns>.svc:6379`, no credentials (unchanged) |
| `ADMIN_PASSWORD` | ✓ | ✓ | **new**: `secretKeyRef` `<shop>-admin` / `password` |
| `REDIS_PASSWORD` | – | ✓ | **new**: `secretKeyRef` `<shop>-redis-auth` / `password` |

- **Port** `80` (`http`); readiness = TCP connect to `http` (unchanged).
- **Memory** limit 128 Mi (unchanged).
- **New**: pod `securityContext.sysctls` `net.ipv4.ip_unprivileged_port_start=0`, so a non-root
  app can bind 80 (research R5).
- **New**: pod-template annotations `shophub.io/admin-secret-uid` and (light)
  `shophub.io/redis-secret-uid` hold the Secrets' UIDs. A recreated Secret therefore rolls the
  pods. UIDs only, never values or hashes.

## OT `Redis` spec additions (`light`)

```yaml
spec:
  kubernetesConfig:
    redisSecret: {name: <shop>-redis-auth, key: password}
  storage:
    keepAfterDelete: false
    volumeClaimTemplate:
      spec:
        accessModes: [ReadWriteOnce]
        resources: {requests: {storage: 1Gi}}
  redisConfig:
    dynamicConfig: ["appendonly yes"]
```

## Reconcile rules added

1. Ensure `<shop>-admin` exists before applying the Deployment. On a cache miss, confirm with
   `APIReader`. If it is really absent, create it with a new random value. A regenerated admin
   Secret is safe (research R11).
2. `light` only: ensure `<shop>-redis-auth` exists before applying the Redis.
   - If it is absent **and** `<shop>-redis` already exists, do not regenerate. Report `Ready=False`
     with reason `RedisCredentialsLost`.
   - Recovery: delete and recreate the Shop (README).
3. **0.1.0 upgrade**: a live `<shop>-redis` without `spec.storage` is deleted once, with an info
   log `recreating Redis without storage (0.1.0 upgrade)`, and recreated on the next reconcile.
   The Shop shows `DatabaseProvisioning` meanwhile.
4. The Shop controller also `Owns` Secrets, so a deleted admin Secret is noticed and recreated
   without waiting for a resync.

## Status additions

| Kind | Reason | Ready | Meaning |
|------|--------|-------|---------|
| Shop | `RedisCredentialsLost` | False | Light shop's `<shop>-redis-auth` was deleted while its Redis exists; not regenerated |

The operator emits no Kubernetes events (as in 0.1.0). No secret value ever appears in status,
events or logs.

## RBAC and chart

- **RBAC**: no new rules. Secrets already have full verbs, the Redis operator manages the PVC, and
  no events are emitted.
- **Image**: `ghcr.io/nikola031ue/shop-operator:0.2.0`.
- **Chart**: helm-charts `shop-operator` `0.3.0`, appVersion `0.2.0`. kube-state
  `clusters/local/shop-operator/helm.yaml` → `0.3.0`.

## Guarantees (added)

- Deleting a Shop also removes its admin and Redis Secrets and, through the Redis operator, the
  Redis PVC (data lost), within 2 minutes.
- Upgrading from 0.1.0 needs no hand change. Standard shops keep their database. Light shops get a
  fresh, authenticated, persistent Redis.
- An existing admin or Redis password is never replaced while its Secret exists.
