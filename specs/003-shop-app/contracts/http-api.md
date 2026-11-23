# Contract: Shop HTTP API

Served by the one shop image on port 80, under `/api`. The Angular app is served from `/`, and
unknown non-`/api` paths return `index.html`. JSON only. Errors use RFC 7807 problem details. A
validation problem adds `errors: {field: [message]}`. Money is integer cents of mUSDT
([data-model.md](../data-model.md)).

## Public (no sign-in)

| Method | Path | Request | Responses |
|--------|------|---------|-----------|
| GET | `/api/shop` | – | 200 `{name}` |
| GET | `/api/items?q=&page=1` | `q` optional, case-insensitive substring | 200 `{items:[{id,name,priceCents,stock,available}], page, pageSize:50, total}` sorted by name |
| POST | `/api/orders` | `{lines:[{itemId, quantity, unitPriceCents}]}` (1–50 lines, quantity ≥ 1) | 201 `{reference, status:"PendingPayment", totalCents, createdAt}` · 400 validation · 409 `{problems:[{itemId, kind:"removed"\|"price"\|"stock", currentPriceCents?, currentStock?}]}`, nothing stored · 503 database unavailable |

Duplicate `itemId`s in one request are merged before checking.

## Admin (cookie session, research R11)

All endpoints except `login` need a valid session (401 otherwise). State-changing endpoints also
need the `X-XSRF-TOKEN` header that matches the `XSRF-TOKEN` cookie (400 otherwise).

| Method | Path | Request | Responses |
|--------|------|---------|-----------|
| POST | `/api/admin/login` | `{password}` | 204 + session cookie · 401 wrong password · 429 locked (`Retry-After` seconds) |
| POST | `/api/admin/logout` | – | 204, cookie cleared |
| GET | `/api/admin/me` | – | 200 `{initialPassword: bool}` (UI prompts to change it) |
| POST | `/api/admin/password` | `{currentPassword, newPassword}` | 204 · 400 (`newPassword` < 12 chars) · 401 wrong current password |
| GET | `/api/admin/items?q=&page=1` | – | 200, same shape as the public list |
| POST | `/api/admin/items` | `{name, stock, price}` (`price` decimal string, ≤ 2 decimals) | 201 item · 400 validation · 409 name exists |
| PUT | `/api/admin/items/{id}` | `{name, stock, price}` | 200 item · 400 · 404 · 409 name exists |
| DELETE | `/api/admin/items/{id}` | – | 204 · 404 |
| GET | `/api/admin/orders?page=1` | – | 200 `{orders:[{reference, createdAt, status, totalCents, walletAddress, lines:[{itemId,name,unitPriceCents,quantity}]}], page, pageSize:20, total}`, newest first |

## Lockout (FR-006)

- 5 failed logins within 15 min → every login is refused with 429 for 15 min, even with the
  correct password.
- The counter is shared by all replicas (stored in the database).

## Operational

| Method | Path | Notes |
|--------|------|-------|
| GET | `/healthz` | 200 when the store answers a ping, 503 otherwise. Not used by the operator (its probe is TCP); for local checks |

## Never in responses or logs

Password, password hash, `ADMIN_PASSWORD`, `REDIS_PASSWORD`, `DATABASE_URL` (FR-024). Unhandled
errors return a generic 500 problem with a correlation id; details go only to the log, with
connection strings removed.
