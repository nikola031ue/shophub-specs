# Quickstart & Validation: Shop Application

Commands assume Git Bash. References: [operator contract](contracts/operator-0.2.0.md),
[HTTP API](contracts/http-api.md), [delivery](contracts/delivery.md),
[data model](data-model.md).

## Prerequisites

- The 002 platform is running from kube-state (`5 unchanged`), with the Discord token Secret in
  place and `platform-alerts` Ready.
- To develop the app: .NET 10 SDK (version from `shop/global.json`), Node LTS, and Docker Desktop
  (Testcontainers).
- Test resources are applied by hand **only** in the throwaway namespace `qs`, as in 002, and
  deleted at the end (plan: Complexity Tracking).

## Scenario 1 — Operator 0.2.0 tests (Phase 1)

```bash
cd shop-operator && go test ./...
```

Expected: all pass. The new envtest specs cover:

- admin Secret created, owned and referenced, and regenerated after deletion;
- for light shops, Redis Secret, `redisSecret`, storage and `appendonly`;
- a lost Redis password Secret regenerated under a new name, with the same Redis and storage
  (0.2.1; `RedisCredentialsLost` in 0.2.0);
- the 0.1.0 Redis recreate;
- the sysctl, the UID annotations and the memory request/limit (128Mi/256Mi).

## Scenario 2 — Automatic upgrade of 0.1.0 Shops (Phase 1)

1. Before the operator change, on the cluster still at operator 0.1.0: create a standard Shop
   `std` and a light Shop `lite` in `qs`, each with a Wallet and a Ready DiscordChannel as in 002
   Scenario 3, and wait for both to be Ready.
2. Merge the kube-state PR (`shop-operator` 0.3.0) and run `./bring-up.sh`. Expected: only
   `shop-operator` upgraded.
3. Expected within 5 min, with no hand change:
   - Both shops are Ready again and have `<shop>-admin` Secrets.
   - `lite` has `lite-redis-auth`. Its Redis was recreated once (operator log line) and has a
     Bound PVC.
   - `redis-cli -a` without a password gets `NOAUTH`; with the password it gets `PONG`.
   - `CONFIG GET appendonly` returns `yes`.
   - `std` database pod UID is unchanged.
   - The shop pods show memory request `128Mi` and limit `256Mi`.
4. (0.2.1) Write a key with `redis-cli`, then delete the `lite-redis-auth` Secret. Expected, with
   no hand change:
   - `lite-redis-auth-2` appears, owned by `lite`, and `lite-redis` references it;
   - the Redis pod is replaced; the Redis CR and PVC UIDs are unchanged and the key is still there;
   - the `lite` pods roll to the new `REDIS_PASSWORD` reference and `lite` is Ready again;
   - the operator log names both Secrets and contains no password.
   (0.2.0 showed `RedisCredentialsLost` here and needed `lite` to be recreated.)

## Scenario 3 — App tests (both stores, SC-003, SC-004)

```bash
cd shop && dotnet test && (cd src/Shop.Web && npm ci && npx ng test --watch=false)
```

Expected: the store contract suite passes for both PostgreSQL and Redis, including:

- 20 concurrent single-piece orders against 5 in stock → exactly 5 succeed;
- the lockout tests;
- the API tests on both stores.

## Scenario 4 — Release and rollout (US4, SC-008)

1. Tag `v0.1.0` in `shop`. Expected: the `release` job pushes `ghcr.io/nikola031ue/shop:0.1.0`
   within 15 min. After the package is made public, `docker pull` works anonymously.
2. Merge the kube-state PR that sets `shopImage` and run `./bring-up.sh`. Expected: only
   `shop-operator` upgraded; `std` and `lite` roll to the app and stay Ready.

## Scenario 5 — Admin and items (US1, SC-001, SC-006)

```bash
kubectl port-forward -n qs svc/std 8080:80
kubectl get secret -n qs std-admin -o jsonpath='{.data.password}' | base64 -d   # don't paste it anywhere
```

1. Open `http://localhost:8080/admin` and sign in. Expected: a prompt to change the initial
   password. Change it.
2. Create, edit and delete items as in US1 scenarios 2–4. Invalid input is refused per field.
3. Six wrong passwords in a row: the sixth attempt, even if correct, gets "locked" (429).
4. Repeat steps 1–2 on `lite`. Time from Ready to first visible item < 2 min (SC-001).
5. Restart the app pods (`kubectl rollout restart` is a hand change, so do it only in `qs`) and
   sign in again with the **changed** password. Expected: it works, because the changed password
   is not overwritten.

## Scenario 6 — Customer order (US2, SC-002)

On `std` and `lite`:

1. Search `shirt`, add 2 pieces to the cart, reload the page; the cart is kept.
2. Place the order. Expected: a reference is shown and the cart is empty; stock went down by 2.
   Time from opening the shop to the placed order < 1 min.
3. Fill a cart, have the admin change that item's price in another tab, then place the order.
   Expected: a "price changed" problem, no order created. After confirming the corrected cart,
   the order succeeds.
4. **SC-005**: seed 1,000 items through the admin API with a small script (signed-in cookie and
   XSRF header), then search a substring that matches about 10 of them. Expected: results within
   1 s in the browser's network tab.

## Scenario 7 — Orders list (US3)

Sign in as admin and open Orders. Expected:

- the orders are listed newest first with reference, time, PendingPayment, total and lines;
- an order whose item was later renamed still shows the old name and price.

## Scenario 8 — Durability and replicas (SC-009, US4 scenario 6)

1. `std` with `availability: high` (3 replicas): sign in and click through items several times.
   The session keeps working on every replica.
2. Delete the `lite-redis` pod (a hand change in `qs`). After it returns, all items and orders
   are still there.
3. Upgrade the app (tag `v0.1.1` → kube-state tag change) or restart the pods. All data is kept on
   both shops.

## Scenario 9 — No secret leaks (SC-007, FR-024)

Scan for the admin and Redis passwords and the DB URI. Expected 0 matches in all of:

- `kubectl get shop -n qs -o yaml`;
- the app logs (`kubectl logs -n qs deploy/std`);
- the operator log;
- `git grep` over all repositories.

Clean up: `kubectl delete ns qs`.

## Validation log

### 2026-11-24 — Scenarios 3–9 on Docker Desktop (T024)

Cluster: Docker Desktop Kubernetes. Software:

- shop-operator 0.2.1 (chart 0.3.1);
- shop 0.1.0 (`sha256:ad30c64d…`);
- CNPG with PostgreSQL 18.7, and redis-operator 0.27.0 with Redis 7.2.16.

Shops `std` (standard, PostgreSQL) and `lite` (light, Redis) ran in namespace `qs`.

Scenarios 5–9 ran through `kubectl port-forward` with a script that drives the HTTP API the way
the Angular app does: cookie session plus the `X-XSRF-TOKEN` header.

- Admin passwords were read from the Secrets into shell variables only and never printed.
- Times marked "scripted" are API times, not human click times.
- The browser UI was clicked through by the maintainer against the same image on Postgres before
  the release (T022). The browser-only behaviour (cart kept on reload, the price-change dialog)
  is covered by the `ng` tests.

| Scenario | Result |
|----------|--------|
| 3 App tests | Pass. Release run 45391109635 on the `v0.1.0` commit: 154/154 .NET tests (store contract on PostgreSQL 18.7 and Redis 7.2.16, including 20 concurrent orders against 5 in stock, lockout and API suites) and 17/17 `ng` tests |
| 4 Release and rollout | Pass. The `release` job finished **2 min 42 s** after the tag (SC-008 ≤ 15 min); anonymous pull works; no `latest`. kube-state#7 + `./bring-up.sh`: only `shop-operator` upgraded, and `std` and `lite` were on the app and Ready in **27 s** |
| 5 Admin and items | Pass on both, 18/18 checks each: see details below |
| 6 Customer order | Pass on both: see details below |
| 7 Orders list | Pass on both: newest first; after renaming and repricing the item, the old order still shows `Blue shirt`, 15.00 × 2, PendingPayment, with the wallet address |
| 8 Durability and replicas | Pass, 11/11 checks: see details below |
| 9 No secret leaks | Pass: 0 matches. See details below |

**Scenario 5 (admin and items)**

- Sign-in with the generated password; the app asks for a new one; a password under 12 characters
  is refused; the change keeps the browser signed in.
- Create, edit and delete items. An invalid item gets per-field errors (`name`, `price`,
  `stock`). A case-insensitive duplicate name gets 409.
- Admin API without a session: 401.
- Step 3, lockout: after 5 wrong passwords, the correct password gets **429** with
  `Retry-After: 900` on both shops.
- Step 5, after `rollout restart`, is recorded under Scenario 8.
- SC-001: the first item was visible in the storefront within 1 s of signing in (scripted).
  Both shops were Ready before the run.

**Scenario 6 (customer order)**

- Case-insensitive search; 2 pieces ordered (SC-002: shop opened to order placed in **1.0 s /
  0.8 s**, scripted); stock went from 10 to 8.
- The admin changed the price, so the order at the old price got 409 `price` with
  `currentPriceCents` 1600 and no order was stored. The order at the confirmed price succeeded.
- Quantity 1,001: 400.
- SC-005: seeded **1,000 items** through the admin API (about 240 s per shop, 0 failures). A search
  for `widget 01` matched 10 of 1,003 items.
  - Through port-forward: best 10/12 ms, worst **18/22 ms** (std/lite).
  - In-cluster: ≤ 82 ms on the first request, then 5–20 ms.

**Scenario 8 (durability and replicas)**

- `std` at `availability: high` (3 pods): one session, signed in through the Service, worked on
  every pod directly (read, plus a write with XSRF).
- `lite-redis-0` deleted: the pod was replaced, and all 1,003 items and 2 orders were still
  there.
- `rollout restart` of both shops:
  - the **changed** admin password still works and `initialPassword` stays false (Scenario 5
    step 5);
  - data kept on both (1,003 items, 2 orders each);
  - `std` was set back to `standard` afterwards.

**Scenario 9 (no secret leaks)**

Searched for 7 values: both generated admin passwords, both changed admin passwords, the Redis
password (`lite-redis-auth-2`), and the CNPG URI and password. **0 matches** in:

- `kubectl get shop -o yaml`;
- the events in `qs`;
- the logs of all app pods of both shops;
- the operator log;
- `git grep` over every commit of shop, shop-operator, helm-charts, kube-state and shophub-specs.

A positive control (the same search over the Secret's own value) matched, so the search works.

Limit: pods replaced during the run take their logs with them. The current app logs date from
the Scenario 8 restart. Earlier log scans:

- each phase's image check;
- the T026 drill (0 hits for the new Redis password);
- the T022 click-through instance.

Also covered in this run: the T026 drill on `lite`. The deleted `lite-redis-auth` was replaced
by `lite-redis-auth-2`, the Redis restarted, and the key survived (tasks.md log).

Cleanup: `kubectl delete ns qs`. The script and its temporary password files were deleted.
