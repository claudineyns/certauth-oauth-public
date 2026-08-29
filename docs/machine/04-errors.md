# 04 — Errors and lifetimes

[← 03 Resources](03-resources.md) · [index](README.md)

## The matching rule

Every error response has the same shape:

```json
{"error": "<identifier>", "error_description": "<human prose, in Portuguese>"}
```

**Branch on `error`. Never parse `error_description`.** The description exists for a
person reading a terminal; it is written in Portuguese, it is not part of this
contract, and it changes.

Some identifiers are themselves Portuguese words. They are identifiers all the same —
opaque strings to be matched literally, never translated:

```
  interaction_nao_encontrada     sem_certificado_de_cliente
  as_indisponivel                as_timeout
```

## Error identifiers

### From the Authorization Server

| `error` | HTTP | Meaning | Action |
|---|---|---|---|
| `invalid_request` | 400 | Malformed request, or a parameter the basket does not allow | Fix the request. Not retryable. |
| `invalid_client` | 401 | Client authentication failed, or the wrong mechanism was used | Check the basket; see [02](02-authentication.md). Not retryable. |
| `invalid_grant` | 400 | Authorization code, refresh token or PKCE verifier rejected | Restart the grant. Not retryable as-is. |
| `invalid_scope` | 400 | Scope unknown or not permitted for this client | Fix the scope set. |
| `unsupported_grant_type` | 400 | Grant not in `grant_types_supported`, or not registered for this client | Fix, or register a client that has it. |
| `invalid_token` | 401 | Bearer credential rejected — includes a consumed or expired IRT and an invalid maintenance token | Depends on which token; see below. |
| `invalid_client_metadata` | 400 | Registration payload rejected | Fix the metadata. |
| `invalid_redirect_uri` | 400 | `redirect_uris` malformed or absent | Fix. Required even for clients that never redirect. |
| `invalid_dpop_proof` | 400/401 | Proof malformed, expired, replayed, or not matching the token key | Generate a **fresh** proof. See below. |
| `rfc_config_immutable` | 400 | Attempted to change the basket | Register a new client. Never retryable. |
| `interaction_nao_encontrada` | 404 | Interaction expired or unknown — 5-minute lifetime | Restart the authorization request. |
| `sem_certificado_de_cliente` | 400 | Only from `/whoami-cert`: no client certificate on this connection | Use the mutual-TLS host and present one. |

### From the Resource Server

| `error` | HTTP | Meaning | Action |
|---|---|---|---|
| `invalid_token` | 401 | Unknown, expired or revoked token | Acquire a new token. |
| `consent_required` | 403 | The consent behind this token was withdrawn | **Not retryable.** A new token is issued normally and refused the same way. |
| `insufficient_scope` | 403 | Token valid, scope missing. `WWW-Authenticate` names it. | Request a token with the scope. Not retryable as-is. |
| `invalid_dpop_proof` | 401 | Proof rejected on a resource call | Fresh proof per request. |
| `invalid_authorization_details` | 403 | RAR token: request does not match what was authorized, or already used | Not retryable — the token was single-use. |
| `access_denied` | 403 | Consent belongs to other parties | Not retryable. |
| `not_found` | 404 | Account, transaction or entity does not exist | Not retryable. |

### From the UI backend

| `error` | HTTP | Meaning |
|---|---|---|
| `as_indisponivel` | 502 | The backend could not reach the Authorization Server |
| `as_timeout` | 504 | The Authorization Server did not answer in time |

These two — and only these two — indicate a transport condition rather than a
rejected request. They are the ones worth retrying with backoff.

## Lifetimes

Everything expires. Nothing here is permanent.

| Artifact | TTL | Notes |
|---|---|---|
| `initial_registration_token` | 24h | **Single-use.** Consumed by registration. |
| Client registration | 24h | The whole client disappears |
| Basket (`rfc_config`) | 24h | Immutable while it lives |
| `registration_access_token` | 24h | Dies with the client |
| Legal entities | 24h | **Renewed at registration** |
| Access token | 1h | |
| Refresh token | 24h | Non-rotating — the same value keeps working |
| Authorization code | 5min | Single-use |
| Interaction | 5min | Login/consent session |
| PAR `request_uri` | **60s** | Single-use |
| DPoP proof `jti` | 5min | Replay window |
| Consent | 24h | |
| Client certificate | 1 day validity | Thumbprint kept 24h |
| Client public key | 24h | |

The practical ceiling is **24 hours for an entire integration**. After that the
client is gone and a human must create a new basket. Build for that, not around it.

## What invalidates what

Cause and effect, in the order you are likely to trip over them.

| Event | Consequence |
|---|---|
| Registering a client | Consumes the IRT permanently |
| Revoking an access token | That token only. Refresh tokens survive. |
| Revoking a consent | Every token for that client/entity pair is refused with `consent_required` (403) **at the Resource Server**, while still appearing active at the Authorization Server. Issuing a new token does not help. |
| Reissuing a client certificate | **All** tokens bound to the previous certificate stop working immediately |
| Reissuing a client key pair | Every assertion signed with the previous key is rejected |
| `DELETE /register/{id}` | Cascades: client, basket, certificate thumbprint, public key, consents, maintenance token. The legal entities survive on their own TTL. |
| Entity TTL expiring | Profile reads fail; derived accounts and history become unreachable |

## An unattended loop

```
  acquire_token():
      response = POST /token  with the mechanism the basket dictates
      if response.error == "invalid_client":
          stop — configuration problem, retrying will not help
      return response.access_token, response.expires_in

  call(request):
      if token is absent or expires within 60s:
          acquire_token()
      if basket.proof == "dpop":
          attach a FRESHLY generated proof        # never reuse, not even on retry
      if basket.proof == "mtls":
          send to the -mtls host with the client certificate
      response = send(request)

      if response.status == 401 and response.error == "invalid_token":
          if this is the first 401 for this request:
              acquire_token(); retry once
          else:
              stop — the client itself is gone, not the token
      if response.status == 403:
          stop — scope, consent or authorization problem
          # consent_required in particular: a fresh token is refused identically
      if response.error in ("as_indisponivel", "as_timeout"):
          retry with backoff
      return response
```

Four rules that catch most integration bugs here:

**Retry a 401 exactly once.** A second identical 401 after a fresh token means the
cause is not the token — the certificate was reissued, or the client expired.
Retrying past that point produces a loop.

**Do not confuse 401 with 403 here.** `invalid_token` (401) is the one case where
acquiring a new token is the right move. `consent_required` (403) looks similar and
is the opposite situation: a human withdrew permission, and the Authorization Server
will keep issuing tokens that the Resource Server keeps refusing.

**Never retry a 403.** Scope and authorization failures are deterministic.

**Generate a new DPoP proof for every attempt, including retries.** Proofs are
single-use for five minutes. Reusing one on a retry turns a recoverable 401 into
`prova DPoP ja usada (replay)`, which looks like a different fault and is not.

## Idempotency

Nothing in this API is idempotent by key — there is no idempotency header.

`POST /api/transactions` under `client_credentials` is **repeatable**: retrying
creates a second transaction. If the connection dropped and you do not know whether
the first succeeded, list transactions before retrying rather than sending it again.

With a RAR token the same endpoint is single-use and a second attempt returns
`invalid_authorization_details` — safe, but only for that path.

`POST /revoke` is safe to repeat: revoking an unknown or already-revoked token
returns `200` per RFC 7009. That also means **the status code tells you nothing about
whether the token existed.**

---

[← 03 Resources](03-resources.md) · [index](README.md)
