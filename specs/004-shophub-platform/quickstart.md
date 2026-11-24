# Quickstart & Validation: ShopHub Platform (004)

Run on Docker Desktop Kubernetes (context `docker-desktop`) after each rollout step in
[contracts/delivery.md](contracts/delivery.md). Expected results refer to the spec's FRs and SCs.

## Prerequisites

- The platform from 002/003 running through kube-state.
- `mkcert` installed. One time: `mkcert -install` (trusts the local CA in the browser and
  Windows store).
- The TLS Secret, created by hand once per cluster (never committed; README "Local HTTPS"):

  ```bash
  mkcert -cert-file shophub.pem -key-file shophub-key.pem shophub.localhost '*.shops.localhost'
  kubectl create namespace traefik
  kubectl create secret tls shophub-tls -n traefik --cert=shophub.pem --key=shophub-key.pem
  rm shophub.pem shophub-key.pem
  ```

## Scenario 1 — Entry point and shop addresses (US1 scenarios 1–2, FR-020, FR-021)

1. Merge the kube-state PRs for `traefik` and `shop-operator` 0.4.0 and run `./bring-up.sh`.
   Expected: `traefik` installed and `shop-operator` upgraded; nothing else changes. Without
   `shophub-tls`, bring-up stops and prints the steps above.
2. The maintainer creates two test Shops in `shops` (hand change, Complexity Tracking), each with
   a registered Wallet and a DiscordChannel.
3. Expected:
   - each Shop's status `url` is `https://<shop>.shops.localhost`;
   - the browser opens it with a trusted padlock;
   - `http://` redirects to `https://`;
   - the two hosts serve different shops.
4. Delete one Shop. Expected: its host returns 404 from the entry point within 2 minutes.

## Scenario 2 — Least privilege and isolation of shop pods (US1 scenario 7, FR-026, SC-007)

```bash
SA=system:serviceaccount:shop-operator-system:shop-operator
for ns in shops shop-operator-system shophub cnpg-system traefik default; do
  echo "$ns: $(kubectl auth can-i get secrets -n $ns --as $SA)"; done
kubectl auth can-i get secret/shop-operator-discord -n shop-operator-system --as $SA
kubectl get clusterrolebinding -o wide | grep shop-operator   # expected: nothing
```

Expected: `yes` only for `shops` and for the token Secret by name. No ClusterRoleBinding. A Shop
created in another namespace gets no children.

NetworkPolicy check: from a throwaway pod in `default`, `curl http://<shop>.shops.svc` must fail.
From the entry point it works. Record whether the local CNI enforces the policy (research R5).

## Scenario 3 — Real client address in shops (US1 scenarios 3–6, FR-022–FR-025, SC-005)

After shop `v0.2.0` is rolled out:

1. From two throwaway pods (different pod IPs), call the shop **through the entry point
   Service** (`traefik.traefik.svc`, with a `Host:` header).
   - 5 wrong admin sign-ins from pod A, then the right password from pod B: **204**.
   - Pod A gets 429.
2. Pod A sends `X-Forwarded-For: <pod B IP>` itself. Expected: still treated as pod A.
3. Pod A places 10 orders, then pod B places one. Expected: pod B **201**, pod A's 11th **429**.
4. Sign in through the browser. Expected: session and anti-forgery cookies are `Secure`.
5. `GET /api/shop` shows `adminPasswordChanged: false`, then `true` after a password change.

Delete the Phase 1 test resources afterwards.

## Scenario 4 — ShopHub up (FR-027, FR-028, SC-009)

After the `shophub` kube-state PR: run `./bring-up.sh`. Expected:

- `https://shophub.localhost` shows the sign-in page with a trusted certificate.
- The ShopHub pods run as ServiceAccount `shophub`.
- `kubectl auth can-i --list -n shops --as system:serviceaccount:shophub:shophub` lists only the
  three kinds and Secret `get`.
- In every other namespace it lists nothing beyond discovery.

From an empty cluster (optional full run): bring-up through to a working ShopHub in under 15 min.

## Scenario 5 — Accounts (US2, FR-001–FR-005)

1. Register `a@example.test`, sign out, sign in. A duplicate registration (any letter case) gets
   409.
2. Five wrong passwords from one address, then the right one from that address: 429. From
   another address (Scenario 3 method): it works.
3. Copy the session cookie, sign out, replay the cookie. Expected: 401.

## Scenario 6 — Create and open (US3, SC-001, SC-002)

1. As owner A, create `Acme` (standard, PostgreSQL) and `Light` (standard, Redis) with a valid
   wallet address. Expected: both in the list within 5 s as "being prepared", then "ready".
2. Record the time from registration to the first storefront open in a new tab (SC-001 < 5 min).
3. "Show shop admin password", sign in to the shop admin area, change it. Expected: the panel now
   says the password was changed (409 `changed`).
4. Invalid inputs (empty name, 101 characters, duplicate name in other letter case, bad checksum,
   `availability: medium`): field errors, nothing created (`kubectl get shops -n shops`).
5. A 6th shop for owner A: refused with the limit message.

## Scenario 7 — Isolation (FR-005, FR-018, SC-004)

As owner B, call every shop endpoint with owner A's shop id (GET, PATCH, DELETE, admin
password). Expected: 404, the same as an unknown id. Owner B's list doesn't contain A's shops.
The automated isolation suite in CI covers the same matrix.

## Scenario 8 — Change and delete (US4, US5, SC-002, SC-003)

1. Switch `Acme` to high. Expected: 3 pods within 2 min; items and orders unchanged.
2. Change `Acme`'s wallet address.
   - A new order carries the new address; old orders keep theirs.
   - Wallet `<id>-w2` exists and `<id>-w1` is gone.
3. Delete `Light` with the right name.
   - Within 2 min: gone from the list.
   - Shop, Wallet, DiscordChannel, Redis, Ingress and NetworkPolicy are gone.
   - Its host no longer serves the shop.
4. Delete with a mistyped name: 400, nothing deleted.
5. Delete the Shop resource of `Acme` by hand (drift). Expected: ShopHub puts it back within 60
   s. This is a hand change, recorded.

## Scenario 9 — No secret leaks (FR-030, SC-008)

Search for these values:

- owner passwords;
- the ShopHub DB password and URI;
- the shop admin passwords;
- the Redis password;
- the Discord token;
- the TLS key.

Expected **0 matches** in:

- `git grep` over every commit of all repositories;
- ShopHub, shop and operator logs;
- `kubectl get shops,wallets,discordchannels -n shops -o yaml`;
- ShopHub API responses, other than the admin-password response.

Clean up: delete the test owners' shops through the panel. Delete the throwaway pods.
