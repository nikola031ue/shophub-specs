# Contract: ShopHub panel HTTP API

Served by the `shophub` image under `/api`. The Angular panel is served from `/`, and unknown
non-`/api` paths return `index.html`. JSON only.

- Errors are RFC 7807 problem details. Validation problems add `errors: {field: [message]}`.
- A PATCH or DELETE without a JSON body is answered like an empty body (400), never with the
  framework's 404.
- All state-changing requests need the `X-XSRF-TOKEN` header matching the `XSRF-TOKEN` cookie
  (400 otherwise).
- Owner endpoints need a valid session (401 otherwise).
- Every response carries the same security headers as shop 0.1.1.

## Accounts (FR-001–FR-004)

| Method | Path | Request | Responses |
|--------|------|---------|-----------|
| GET | `/api/session` | – | 200 `{signedIn, email?}`. Also (re)issues the `XSRF-TOKEN` cookie |
| POST | `/api/register` | `{email, password}` | 202, empty body, no session cookie, **the same whether or not the email already has an account** (FR-001; an existing account is left unchanged, and a password hash is computed either way so timing matches). The panel then calls `/api/signin` · 400 validation (`email` at most 254 characters with one `@`, `password` 12–200 characters) · 429 more than 10 registrations per hour from one client address |
| POST | `/api/signin` | `{email, password}` | 204 + session cookie · 401 wrong email or password: the same body for both, and an unknown email still verifies a password against a fixed dummy hash so timing matches (FR-001); it counts towards the lockout like any failure · 429 locked (`Retry-After`) |
| POST | `/api/signout` | – | 204; this session ends; a copied cookie gets 401 afterwards |
| POST | `/api/password` | `{currentPassword, newPassword}` | 204; every other session of this owner ends, this one stays (FR-002) · 400 validation (`newPassword` 12–200 characters) · 401 wrong current password (counts towards the lockout like a failed sign-in) |

## Shops (FR-005–FR-016)

`{id}` is the generated identifier. A shop that does not exist and a shop of another owner both
give **404** with the same body (FR-005, SC-004).

| Method | Path | Request | Responses |
|--------|------|---------|-----------|
| GET | `/api/shops` | – | 200 `{shops:[Shop]}`, the caller's shops only, newest first |
| POST | `/api/shops` | `{name, availability, database, walletAddress}` | 201 `Shop` (state `preparing`) · 200 the existing `Shop` when the same create is repeated (same name, settings and wallet; SC-010) · 400 validation · 409 `{reason: "duplicateName", errors: {name}}` for the same name with other settings · 409 `{reason: "limit"}` when the owner already has 5 shops in any state |
| GET | `/api/shops/{id}` | – | 200 `Shop` · 404 |
| PATCH | `/api/shops/{id}` | `{availability?, walletAddress?}`, at least one | 200 `Shop`, also when the values sent are the current ones (nothing is written) · 400 validation by field: `name` or `database` present (even unchanged), any other field, a bad value, or `{}` (key `""`) · 404 · 409 `{reason: "deleting"}` shop is being deleted |
| DELETE | `/api/shops/{id}` | `{confirmName}` | 202 `Shop` (state `deleting`), also for a repeat while it is being deleted · 400 `errors: {confirmName}` when `confirmName` doesn't match the display name (trimmed, exact case) or the body is missing · 404. Order: ownership first (another owner's shop is 404 whatever name is sent), then the name |
| POST | `/api/shops/{id}/admin-password` | – | 200 `{password}` with `Cache-Control: no-store`, only while the shop is ready and the generated password is still in use · 409 `{reason: "changed"}` when the shop admin has changed it · 409 `{reason: "notReady"}` (not ready, being deleted, or no Secret yet) · 404 · 503 when the shop doesn't answer the check. Order: ownership, Ready, the shop's own `adminPasswordChanged`, then Secret `<id>-admin` |

`Shop`:

```json
{
  "id": "s7k2m4qxab",
  "name": "Acme Clothing",
  "availability": "standard",
  "database": "light",
  "walletAddress": "0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed",
  "state": "preparing | ready | attention | deleting",
  "reason": "plain-language text or null",
  "url": "https://s7k2m4qxab.shops.localhost or null until known",
  "readyReplicas": 0,
  "replicas": 2,
  "createdAt": "2026-11-24T10:00:00Z"
}
```

`password` never appears in any other response, in logs or in traces.

## Reason texts (FR-008, R10)

| Shop `Ready` reason (002/003) | Panel text |
|-------------------------------|------------|
| `WalletNotReady` | The wallet address is being checked. |
| `DiscordChannelNotReady` | The notification channel is being created. |
| `DatabaseProvisioning` | The database is being prepared. |
| `DatabaseOperatorMissing` | The platform can't create this database type right now. |
| `Progressing` | The shop is starting (x of y copies running). |
| `ReconcileError` | Something went wrong on the platform; it keeps retrying. |
| none yet | The shop is being created. |
| other | The platform is still working on it. |

A Wallet with `InvalidChecksum` is refused before anything is created (FR-006), so it never
reaches the panel.

A shop being deleted has no reason for its first 2 minutes (SC-003). After that, ShopHub reads
the shop's objects by name and names the first one still there, in removal order:

| Still in the cluster | Panel text |
|----------------------|------------|
| Shop | Deleting is taking longer than usual: the shop and its database are still being removed. The platform keeps retrying. |
| DiscordChannel (e.g. a finalizer that can't finish) | Deleting is taking longer than usual: the notification channel can't be removed yet. The platform keeps retrying. |
| Wallet only | Deleting is taking longer than usual: the wallet record is still being removed. The platform keeps retrying. |
| nothing | no reason; the record goes on the next pass |

## Limits and errors

- 503 "temporarily unavailable" when the ShopHub database or the Kubernetes API does not
  answer. Nothing half-done is left (the record is written in one transaction, and the
  reconciler finishes or cleans up).
- 500 with a correlation id for anything unexpected; details only in the log.
- Never in responses or logs: passwords and hashes, session keys, `DATABASE_URL`, the Kubernetes
  token, the shop admin password (except the 200 body above).
