# 01 — Bootstrap

[← index](README.md) · next: [02 — Authentication](02-authentication.md)

## The manual step

A client cannot exist until an RFC basket exists, and **assembling the basket is a
manual step**: a person does it in the browser, at
`https://oauth-playground.certauth.dev`. No automated interface is offered for it,
and none is supported.

This is a design decision, not a missing feature. The basket is the pedagogical core
of the playground: choosing the combination *is* the exercise. A supported API for it
would let an integrator skip the one step that requires understanding what is being
assembled.

Treat it as a precondition of your integration — the same way you would treat an
account that someone has to open for you before you can call anything. Anything you
might reach for to shortcut it is outside this contract: unsupported, undocumented,
and subject to change without notice.

So an integration begins with one human action:

```
  human  -> opens the wizard, picks a basket, submits
  system -> returns an initial_registration_token (IRT) and a pair of legal entities
  agent  -> takes the IRT from here on
```

The IRT is the handoff. Once you hold it, nothing else requires a browser.

**The IRT is single-use and lives 24 hours.** Consuming it registers exactly one
client. A second attempt with the same token:

```json
{"error":"invalid_token",
 "error_description":"initial_registration_token invalido, expirado ou ja usado"}
```

If your integration needs to survive past 24 hours, it needs a human to produce a
new basket. Plan for that rather than around it.

### What the wizard returns

For reference — this is what the human hands you:

```json
{
  "initial_registration_token": "<initial_registration_token>",
  "expires_in": 86400,
  "rfc_config": {
    "modo_autorizacao": "query",
    "proof": "nenhum",
    "formato_token": "bearer",
    "pkce": true,
    "rar": false,
    "client_assertion": false
  },
  "pjs": [
    {"id": "B961D002E8DD28", "identificador": "B9.61D.002/E8DD-28", "papel": "titular"},
    {"id": "9C8FB561112A07", "identificador": "9C.8FB.561/112A-07", "papel": "correntista"}
  ]
}
```

The two legal entities (`pjs`) are created together and bound to each other. One is
`titular` — the third party — and the other is `correntista` — the account holder.
The distinction matters when calling the Resource Server; see
[03 — Resources](03-resources.md).

### Reading the basket

`rfc_config` is the contract your client will be held to. Field by field:

| Field | Values | Effect |
|---|---|---|
| `modo_autorizacao` | `query` · `par` · `jar` | How authorization request parameters travel |
| `proof` | `nenhum` · `dpop` · `mtls` | Proof-of-possession mechanism, if any |
| `formato_token` | `bearer` · `jwt` | Opaque token, or a signed JWT (RFC 9068) |
| `alg` | `RS256` · `ES256` | Signing algorithm, only when `formato_token` is `jwt` |
| `pkce` | `true` · `false` | Whether PKCE is required on the code flow |
| `rar` | `true` · `false` | Rich Authorization Requests; requires `authorization_code` |
| `client_assertion` | `true` · `false` | Replaces the client secret with `private_key_jwt` |

The first three are mutually exclusive choices within their group — one value each,
never several. The last three are additive.

For unattended integration, the combination that gives you the most reach with the
fewest moving parts is `proof: mtls` or `client_assertion: true`, both of which
authenticate without a shared secret. See [02 — Authentication](02-authentication.md).

## Registration — RFC 7591

Exchange the IRT for a client.

```http
POST /register HTTP/1.1
Host: oauth.certauth.dev
Authorization: Bearer <initial_registration_token>
Content-Type: application/json

{
  "client_name": "Agente de Integracao",
  "redirect_uris": ["https://cliente.exemplo/callback"],
  "grant_types": ["client_credentials", "refresh_token"],
  "scope": "accounts:read transactions:read"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
```
```json
{
  "client_id": "<client_id>",
  "client_secret": "<client_secret>",
  "client_secret_expires_at": 1788125134,
  "registration_access_token": "<registration_access_token>",
  "registration_client_uri": "https://oauth.certauth.dev/register/<client_id>",
  "client_name": "Agente de Integracao",
  "redirect_uris": ["https://cliente.exemplo/callback"],
  "grant_types": ["client_credentials", "refresh_token"],
  "rfc_config": { "...": "the basket, echoed back" },
  "pjs": [ { "...": "the two entities, now with names and bindings" } ]
}
```

`redirect_uris` is required even for a client that will never use the authorization
code grant. That is RFC 7591 metadata, not a statement about what you intend to do.

### Three credentials, three jobs

The response hands you three secrets. They do not overlap, and confusing them is the
most common integration mistake here.

| Credential | Used for | Lifetime |
|---|---|---|
| `client_id` | Identifying the client. Not a secret. | 24h |
| `client_secret` | Authenticating at `/token` and `/revoke` — **only if the basket says so** | 24h, never regenerated |
| `registration_access_token` | Managing this registration (RFC 7592) and calling the maintenance API | 24h |

The `registration_access_token` — referred to below as the maintenance token — is
**not** an access token. It buys nothing at the Resource Server. It manages the
client record, and it is the credential for issuing key material.

**`client_secret` is never regenerated.** There is no rotation endpoint. If you lose
it, register a new client.

### Validation you may hit

If the basket has `rar: true` but `grant_types` omits `authorization_code`,
registration is refused. A basket that selects RAR and a client that cannot reach a
consent screen would leave the RFC selected and inexercisable — and the basket is
immutable, so the mistake would be permanent.

## Managing the registration — RFC 7592

Read the current record:

```http
GET /register/<client_id> HTTP/1.1
Host: oauth.certauth.dev
Authorization: Bearer <registration_access_token>
```

The `rfc_config` in the response carries a `locked_at` timestamp — the moment the
basket became immutable.

Update it. **Exactly three fields are writable**: `client_name`, `redirect_uris`,
`grant_types`.

```http
PATCH /register/<client_id> HTTP/1.1
Authorization: Bearer <registration_access_token>
Content-Type: application/json

{"client_name": "Novo Nome"}
```

Anything else is rejected. Attempting to touch the basket is a distinct error, not a
silently ignored field:

```http
HTTP/1.1 400 Bad Request
```
```json
{"error":"rfc_config_immutable",
 "error_description":"A cesta de RFC e imutavel. Para usar outra combinacao, registre um novo client."}
```

Delete, which cascades:

```http
DELETE /register/<client_id> HTTP/1.1
Authorization: Bearer <registration_access_token>
```
```http
HTTP/1.1 204 No Content
```

After a delete the maintenance token stops resolving:

```json
{"error":"invalid_token",
 "error_description":"maintenance_token invalido para este client"}
```

The cascade removes the client record, its basket, its registered certificate
thumbprint, its public key, its consents, and the maintenance token itself. It does
not remove the two legal entities — they outlive the client, on their own TTL.

## Issuing key material

Two endpoints, both on the UI host, both authenticated by the maintenance token in
an `X-Maintenance-Token` header. They are ordinary HTTP and need no browser.

### Client key pair — for JAR and Client Assertion

```http
POST /api/admin/client/keys HTTP/1.1
Host: oauth-playground.certauth.dev
X-Maintenance-Token: <registration_access_token>
```

```json
{
  "kid": "<kid>",
  "private_key_pem": "-----BEGIN PRIVATE KEY-----\n...",
  "public_jwk": {"kty": "RSA", "n": "...", "e": "AQAB"},
  "reemissao": false,
  "aviso": "A chave privada e exibida uma unica vez e nao fica no servidor."
}
```

**The private key is returned once and is not stored server-side.** Capture it on the
first response or reissue.

The server keeps only the public JWK. Reissuing replaces it, which invalidates any
assertion signed with the previous key.

If the basket includes neither JAR nor Client Assertion, the request is refused —
there would be nothing to sign:

```json
{"error":"invalid_request",
 "error_description":"a cesta deste client nao inclui JAR nem Client Assertion; nao ha uso para o par de chaves"}
```

### Client certificate — for mutual TLS

```http
POST /api/admin/client/mtls-cert HTTP/1.1
Host: oauth-playground.certauth.dev
X-Maintenance-Token: <registration_access_token>
```

```json
{
  "cert_pem": "-----BEGIN CERTIFICATE-----\n...",
  "key_pem": "-----BEGIN PRIVATE KEY-----\n...",
  "ca_pem": "-----BEGIN CERTIFICATE-----\n...",
  "dn": "CN=<client_id>, O=OAuth Playground",
  "thumbprint_sha256": "...",
  "x5t_s256": "<x5t_s256>",
  "validade_dias": 1
}
```

The certificate's common name is the `client_id`. As with the key pair, the private
key is returned once and never persisted; the server keeps only the DN and the
thumbprint.

**Reissuing invalidates every token bound to the previous certificate**, immediately.
That is correct RFC 8705 behaviour and it is demonstrated on purpose — see
[04 — Errors and lifetimes](04-errors.md).

Requesting a certificate for a client whose basket does not use mutual TLS is
refused for the same reason as the key pair: nothing would consume it.

---

[← index](README.md) · next: [02 — Authentication](02-authentication.md)
