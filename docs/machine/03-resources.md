# 03 — Calling the Resource Server

[← 02 Authentication](02-authentication.md) · [index](README.md) · next: [04 — Errors and lifetimes](04-errors.md)

The Resource Server is a fictional banking API. It exists to give the tokens
something to protect.

Base: `https://resource.certauth.dev` — or `https://resource-mtls.certauth.dev` when
your token is certificate-bound.

## The domain in one paragraph

Two legal entities are created together with every basket and bound to each other:
the `client_owner`, which your client belongs to, and a `third_party`, which it does
not. Both hold accounts. Accounts and balances belong to entities, not to clients.

A token names an entity in its `sub`, and **every response returns the data of the
`sub`, never of whoever is holding the token.** That single rule is what produces
isolation.

Under `client_credentials` the `sub` is always the `client_owner` — that grant
cannot reach any other entity. Reaching the `third_party` requires the authorization
code grant and an explicit consent, which requires a browser and is therefore outside
this track.

## Scopes

| Scope | Grants |
|---|---|
| `accounts:read` | Accounts, balances, and the consent endpoints |
| `transactions:read` | Transaction history and individual transactions |
| `transactions:write` | Creating transactions |
| `profile:read` | Reading the entity profile |
| `profile:write` | Changing the profile display name |

Request them space-separated in `scope` at the token endpoint. You receive what was
granted, echoed in the token response — compare it against what you asked for rather
than assuming.

## Endpoints

| Method | Path | Scope required |
|---|---|---|
| `GET` | `/api/accounts` | `accounts:read` |
| `GET` | `/api/accounts/{accountId}` | `accounts:read` |
| `GET` | `/api/accounts/{accountId}/balance` | `accounts:read` |
| `GET` | `/api/accounts/{accountId}/statement` | `accounts:read` **and** `transactions:read` |
| `GET` | `/api/transactions` | `transactions:read` |
| `GET` | `/api/transactions/{txnId}` | `transactions:read` |
| `POST` | `/api/transactions` | `transactions:write` |
| `GET` | `/api/profile` | `profile:read` |
| `PATCH` | `/api/profile` | `profile:write` |
| `GET` | `/api/consents` | `accounts:read`, authorization code only |
| `POST` | `/api/consents/{consentId}/revoke` | `accounts:read`, authorization code only |

The two consent endpoints are unreachable under `client_credentials`. There is no
delegation in that grant, so there is no consent to list or revoke.

## A call

```http
GET /api/accounts HTTP/1.1
Host: resource.certauth.dev
Authorization: Bearer <access_token>
```

```json
{
  "holder": "B961D002E8DD28",
  "accounts": [
    {"account_id": "AC9C206FD55E", "type": "checking",
     "branch_code": "4338", "account_number": "2891069-9", "currency": "BRL",
     "holder": "B961D002E8DD28"},
    {"account_id": "ACB18C79E081", "type": "savings",
     "branch_code": "6886", "account_number": "8746876-0", "currency": "BRL",
     "holder": "B961D002E8DD28"}
  ]
}
```

Accounts are derived deterministically from the entity identifier: agency, number and
`account_id` are a function of it and nothing else. They are stable for as long as the
entity exists and occupy no storage.

The **balance** is not derived. It starts at zero, and every transaction you create
moves it. A brand-new account has an empty statement and a zero balance — there is no
synthetic history to read.

**The identifiers above belong to one capture and will differ for your client.** Each
basket creates its own entities, and the accounts follow from them. Discover them with
`GET /api/accounts` and carry the values forward; an `account_id` borrowed from another
client is rejected:

```json
{"error":"invalid_request",
 "error_description":"account_id missing or does not belong to the subject of the token"}
```

## Balance and statement

A new account, before anything happens to it:

```http
GET /api/accounts/<account_id>/balance HTTP/1.1
Authorization: Bearer <access_token>
```
```json
{"account_id": "AC618695DA4E", "available_balance": 0, "currency": "BRL",
 "updated_at": "2026-08-30T06:02:49.011Z"}
```

`GET /api/accounts/<account_id>/statement` returns the balance together with the
movements. On a new account, `transactions` is `[]`.

## Creating a transaction

Four fields are required — `account_id`, `amount`, `type`, `direction` — and unknown
fields in the body are **ignored**, not rejected.

| Field | Rule |
|---|---|
| `account_id` | must belong to the `sub` of the token |
| `amount` | positive, at most 2 decimal places |
| `type` | `pix`, `boleto`, `deposit` or `withdrawal` |
| `direction` | `debit` or `credit` |
| `description` | optional, up to 140 characters |

`deposit` only credits and `withdrawal` only debits; `pix` and `boleto` go both ways.
There is no `currency` field — the system has one.

```http
POST /api/transactions HTTP/1.1
Host: resource.certauth.dev
Authorization: Bearer <access_token>
Content-Type: application/json

{"account_id": "AC618695DA4E", "amount": 2500.00, "type": "deposit",
 "direction": "credit", "description": "Aporte inicial"}
```

```http
HTTP/1.1 201 Created
```
```json
{"transaction_id": "TXABF577E3E504", "account_id": "AC618695DA4E",
 "type": "deposit", "direction": "credit", "amount": 2500,
 "description": "Aporte inicial", "booked_at": "2026-08-30T06:02:50.084Z",
 "origin": "direct", "created_by": "<client_id>"}
```

`credit` raises the balance and `debit` lowers it. After the deposit above and a
`pix` debit of 150.00, the balance reads:

```json
{"account_id": "AC618695DA4E", "available_balance": 2350, "currency": "BRL",
 "updated_at": "2026-08-30T06:02:51.118Z"}
```

**The balance may go negative.** There is no funds check: a debit larger than the
balance succeeds and leaves a negative number, simulating an overdraft line. To hold
funds first, post a `deposit`.

The `account_id` above came from `GET /api/accounts` on the same token — it is not a
constant, and it is not the same one shown earlier in this page.

Under `client_credentials` this is **repeatable**: the client owns the account, so
there is nothing to authorize per payment. `origin` reads `criada`.

With a token carrying `authorization_details` — only obtainable through the
authorization code grant — the same endpoint behaves differently: the token becomes
single-use, the request body must match what was authorized down to the amount, and
`direction` must be `debit`, because a payment initiation is an outflow. `origin` then
reads `criada_via_rar`. An unattended integration cannot reach that path; it is
described here so you recognise the distinction if you see it.

## Errors you will meet here

Validation of the transaction body. All four are `400` with `invalid_request`:

```json
{"error":"invalid_request",
 "error_description":"type is required and must be one of: pix, boleto, deposit, withdrawal"}
```
```json
{"error":"invalid_request",
 "error_description":"direction is required and must be one of: debit, credit"}
```
```json
{"error":"invalid_request",
 "error_description":"type and direction are incompatible: deposit requires direction credit"}
```
```json
{"error":"invalid_request",
 "error_description":"amount must have at most 2 decimal places"}
```

Insufficient scope — note the `WWW-Authenticate` header naming exactly what is
missing:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope", scope="transactions:read"
```
```json
{"error":"insufficient_scope","error_description":"required scope(s): transactions:read"}
```

Unknown, expired or revoked token — all three collapse into one response, and you
cannot tell them apart. Treat them identically: acquire a new token.

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token", error_description="token inactive, expired or revoked"
```

Missing header:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token", error_description="missing Authorization (Bearer or DPoP)"
```

## Certificate-bound tokens

If your token was issued over mutual TLS, it carries `cnf.x5t#S256` and the Resource
Server enforces the binding on **every** call. Three distinct refusals, and the
difference between them is diagnostic:

| Response | Meaning | What to do |
|---|---|---|
| `this token is certificate-bound: use the mTLS host and present the certificate` | You called the plain-TLS host | Retry against `resource-mtls.certauth.dev` |
| `the presented certificate is not the one this token was bound to` | Wrong certificate for this token | Use the certificate the token was issued with |
| `the certificate for this token is no longer the one registered for the client` | The certificate was reissued after this token was minted | Discard every token bound to the old certificate; acquire new ones |

The third exists because the check compares the presented certificate against **two**
values: the token's `cnf`, and the thumbprint currently registered for the client. The
second comparison is what makes reissue take effect immediately. Without it, the old
certificate and its old token would keep working as a matching pair.

Practical consequence: **reissuing a client certificate invalidates all outstanding
tokens**. If your integration reissues on a schedule, acquire fresh tokens right
after, and do not treat the resulting 401s as a fault.

## Consent revocation

Available only to tokens obtained through the authorization code grant.

```http
GET /api/consents HTTP/1.1
Authorization: Bearer <access_token>
```
```json
{"consents":[{"consent_id":"<client_id>.447550F48AFA89",
              "client_id":"<client_id>","legal_entity":"447550F48AFA89",
              "scopes":["accounts:read"],"granted_at":"2026-08-29T21:32:51.670Z"}]}
```

The `consent_id` is `{client_id}.{entity_id}`. Either party may revoke — the client
or the entity — but only a consent the token holder participates in. Revoking someone
else's returns `access_denied`.

```http
POST /api/consents/<client_id>.447550F48AFA89/revoke HTTP/1.1
Authorization: Bearer <access_token>
```
```json
{"revoked":"<client_id>.447550F48AFA89","effect":"immediate"}
```

**Revocation does not revoke the token**, and the refusal that follows is a
*different* error from an invalid token. The access token stays valid at the
Authorization Server; the Resource Server refuses it on the next call because the
consent behind it is gone:

```http
HTTP/1.1 403 Forbidden
```
```json
{"error":"consent_required",
 "error_description":"no active consent from this legal entity for this client; it may have been revoked"}
```

The distinction is operationally important. `invalid_token` (401) means *get a new
token*. `consent_required` (403) means *a human withdrew permission* — a new token
will be issued happily and refused exactly the same way. Do not retry it.

---

[← 02 Authentication](02-authentication.md) · [index](README.md) · next: [04 — Errors and lifetimes](04-errors.md)
