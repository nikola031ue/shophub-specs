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
