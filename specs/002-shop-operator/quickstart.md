# Quickstart & Validation: Shop Operator

Commands assume Git Bash. Resource shapes: [contracts/crds.md](contracts/crds.md).

## Prerequisites

- Feature 001 tooling (Docker Desktop Kubernetes, helm 3.17.3, yq, jq, kubectl, gh).
- For operator development: Go (version from `shop-operator/go.mod`, e.g.
  `winget install GoLang.Go`), Docker Desktop running, `make` optional (all targets have a
  `hack/*.sh` equivalent).
- Published: image `ghcr.io/nikola031ue/shop-operator:0.1.0` (Public), charts `shop-operator`
  and `shophub-discord` `0.2.0` (Public).
- **Required for Scenarios 3–6**: a Discord server you own, a bot with *Manage Channels* and
  *Manage Webhooks* on it, its token and the server (guild) ID. A Shop is Ready only when its
  DiscordChannel is Ready, so SC-001 can't be measured without them. Run Scenario 6 (Discord
  setup) **before** Scenario 3.

## Scenario 1 — Operator tests (US1, US3; Principle III)

```bash
cd shop-operator && go test ./...        # unit + envtest (fake Discord, upstream CRDs)
```

Expected: all pass, without network access to Discord or any blockchain.

## Scenario 2a — Upgrade keeps existing Shops running (US2 scenario 2)

1. **Upgrade from 001**: on the cluster that still runs the feature 001 releases, run
   `./bring-up.sh` from the new `kube-state` `main`. Expected: `cnpg`, `redis-operator`
   installed; `shop-operator` and `shophub-discord` upgraded 0.1.0 → 0.2.0; `shophub`
   unchanged.
2. **Operator upgrade with live Shops** (after Scenario 3 has a Ready Shop in `qs`): change one
   value in `clusters/local/shop-operator/values.yaml` (e.g. `resources.limits.memory`) and run
   `./bring-up.sh`. Expected: only `shop-operator` upgraded, the operator pod restarts, the Shop
   stays Ready, database pod UIDs are unchanged and no duplicate Discord channel appears.
   Revert the value afterwards.

## Scenario 2 — Bring-up installs everything (US2, SC-006)

Reset the Kubernetes cluster in Docker Desktop, then:

```bash
cd kube-state && time ./bring-up.sh
kubectl get crd | grep -E 'shophub.io|cnpg.io|opstreelabs'
```

Expected: `5 installed` (`cnpg`, `redis-operator`, `shop-operator`, `shophub`,
`shophub-discord`); the CRD wait prints the CNPG, Redis and shophub.io CRDs as `Established`;
total < 15 min. A re-run prints `5 unchanged`.

## Scenario 3 — Standard shop lifecycle (US1, SC-001–SC-003)

> In this feature the maintainer plays the backend (no ShopHub yet); applying test resources
> by hand is limited to a throwaway namespace and is cleaned up at the end.

1. Create namespace `qs`, a Wallet without address, a DiscordChannel and a Shop
   (`availability: standard`, `database: standard`) as in the contract example.
2. `kubectl get shop,wallet,dch -n qs -w` → Wallet `generated` Ready, Shop Ready within 5 min,
   `kubectl get deploy -n qs` shows 2/2. (Requires Scenario 6 done first; without a Discord
   token the Shop stays `DiscordChannelNotReady`.)
3. Change `availability: high` → 3/3 within 2 min.
4. Delete the Shop → within 2 min no Deployment, Service, CNPG `Cluster` or pods for it remain
   in `qs`.

## Scenario 4 — Light shop and wallet change (US1)

Shop with `database: light` → an OT `Redis` `<shop>-redis` appears, Shop Ready (this also
proves the operator's RBAC allows reading StatefulSets; `Forbidden` in the Shop message means
a missing RBAC rule). Create a
second Wallet with a registered address and switch `walletRef` → pods roll and `WALLET_ADDRESS`
in the new pods equals that address.

## Scenario 5 — Rejections and protections (SC-004, edge cases)

| Action | Expected |
|--------|----------|
| Shop with `availability: medium` | rejected, message lists allowed values |
| Change a Shop's `database` | rejected, `database is immutable` |
| Wallet `address: 0x123` | rejected by pattern |
| Delete a Wallet still referenced by a Shop | stays, reason `InUse`, `usedBy` lists the Shop |
| Delete a Shop's Deployment by hand | recreated within 30 s |

## Scenario 6 — Discord channel (US3) *(manual, real Discord; run before Scenario 3)*

```bash
kubectl create secret generic shop-operator-discord -n shop-operator-system \
  --from-literal=token="$DISCORD_BOT_TOKEN"          # one-time bootstrap (plan: Complexity Tracking)
```

Set `discord.guildID` in `kube-state/clusters/local/shop-operator/values.yaml`, run
`./bring-up.sh`. Then `platform-alerts` (from `shophub-discord`) and the Scenario 3 channel
appear on the server; posting to the URL in each `<name>-discord-webhook` Secret shows a
message. Deleting the DiscordChannel removes the channel within 2 min.

## Scenario 7 — No secrets leak (SC-007)

`kubectl get shop,wallet,dch -A -o yaml` and `kubectl logs -n shop-operator-system deploy/…`
contain no private key, webhook URL or token. `git grep` in all repos finds none.

Clean up: `kubectl delete ns qs`.

## Validation log

**2026-11-23** — Docker Desktop Kubernetes v1.36.1, Git Bash, helm v3.17.3. shop-operator
image `0.1.0` (`sha256:a229778f…`), charts `shop-operator`/`shophub-discord` `0.2.0`,
kube-state `main` `5f0c1fe` (guild ID via kube-state-spec#4). Discord: project server
`1573413870264131654`, bot token in Secret `shop-operator-discord` (created by the maintainer;
never shown).

| Scenario | Result | Notes |
|----------|--------|-------|
| 1 Operator tests | pass | 52/52 envtest specs + unit packages. `--network none` is not usable (envtest's kube-apiserver needs a non-loopback interface); the tests' only endpoints are local fakes/httptest |
| 2a.1 Upgrade from 001 | pass | Start: kube-state pre-PR `82ecc5f` → `3 unchanged`. New `main`: `cnpg`, `redis-operator` installed; `shop-operator`, `shophub-discord` 0.1.0 → 0.2.0 upgraded; `shophub` unchanged; **75 s**. The operator log shows `Database kinds postgres=true redis=true` |
| 2 Fresh bring-up (SC-006) | pass | After the Kubernetes reset: `5 installed` in **86 s**, shophub.io CRDs `Established`; re-run `5 unchanged` in 23 s |
| 6 Discord (US3) | pass | First attempt: channel created, webhook `403` → `DiscordForbidden` (bot lacked **Manage Webhooks**; fixed by the maintainer). The next retry turned `platform-alerts` Ready after 145 s with the **same channel ID** (no duplicate). Anonymous webhook GET → `channel_id 1574404025443680286`, `guild_id 1573413870264131654`; test message posted (HTTP 204) |
| 3 Standard shop (SC-001, SC-002) | pass | Generated wallet, DiscordChannel `qs-alerts`, Shop `acme`: `DiscordChannelNotReady` → `DatabaseProvisioning` → **Ready in 116 s** (cold Postgres image pull). App env: `SHOP_NAME`, `WALLET_ADDRESS` = wallet status, `DATABASE_URL` = `postgresql://…` from the CNPG Secret. `availability: high` → **3/3 in 3 s** |
| 2a.2 Operator upgrade with live shop | pass | Values change → only `shop-operator` upgraded, operator pod replaced; shop stays Ready, DB pod UID unchanged, Deployment generation unchanged, both channel IDs unchanged. Reverted |
| 4 Light shop + wallet switch | pass | `lite` Ready in 37 s via OT `Redis` `lite-redis` (StatefulSet 1/1; the RBAC statefulsets rule works on a real cluster), no CNPG cluster, `REDIS_URL` set. `walletRef` → registered `reg-wallet`: all new pods have `WALLET_ADDRESS=0xfB69…`; `usedBy` moved accordingly |
| 5 Rejections and protections | pass | `medium` rejected with allowed values; `database` change → `database is immutable`; `0x123` → pattern error; deleting a referenced Wallet held with `InUse`, `usedBy` lists the Shop, released when the Shop is deleted; hand-deleted Deployment recreated in **4 s** |
| 3/6 Deletion (SC-003) | pass | Deleting both Shops: no Deployment/Service/CNPG/Redis/StatefulSet/pods left after **9 s**. Deleting `qs-alerts`: resource and webhook Secret gone in 2 s; Discord answers `404 Unknown Webhook` for its webhook (channel deleted) |
| 7 No secrets leak (SC-007) | pass | Generated private key, webhook URL (and its token part) and the bot token: 0 matches in `shop,wallet,dch -A -o yaml`, cluster events, the operator log, and `git grep` over all commits of shop-operator, helm-charts, kube-state, shophub-specs |

Namespace `qs` deleted afterwards; the platform stays at `5 unchanged`, `platform-alerts` Ready.
