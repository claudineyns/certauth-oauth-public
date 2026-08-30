# Machine track — unattended integration

This track describes the OAuth 2.0 Playground as a wire contract. It is written for
an agent or service that integrates without a human in the loop.

There is no code here in any programming language, by design. What an integrator
needs is the exchange itself — method, headers, body, status, response shape — plus
the rules that decide what happens next. Anything else is a translation you would
have to undo.

Where a decision procedure is clearer than prose, it appears as pseudocode.

## Read in this order

| | |
|---|---|
| [01 — Bootstrap](01-bootstrap.md) | What cannot be automated, and how to register a client once you are past it |
| [02 — Authentication](02-authentication.md) | The three client authentication mechanisms, and how the basket picks one |
| [03 — Resources](03-resources.md) | Calling the Resource Server: scopes, subjects, isolation |
| [04 — Errors and lifetimes](04-errors.md) | Error taxonomy, TTLs, and what invalidates what |

## Three things to know before anything else

**1. Setup begins with a manual step, on purpose.**

A client cannot exist until someone opens a browser and assembles an RFC basket. That
step is done by a person, and no automated interface for it is supported. Everything
after it is machine-driven. See [01 — Bootstrap](01-bootstrap.md); it is the first
thing that will block you if you skip it.

**2. The basket decides what your client can do, and it is immutable.**

Each client is registered against a fixed combination of RFCs, locked at creation.
The basket decides how your client authenticates, whether tokens are opaque or JWT,
and whether proof-of-possession applies. It cannot be changed afterward — attempting
to change it returns `rfc_config_immutable`. To use a different combination, register
a different client.

This is the single most important concept in the system. A request that would succeed
for one client fails for another with identical parameters, because their baskets
differ.

**3. Branch on `error`, never on `error_description`.**

Responses carry two fields. `error` is a stable identifier and is part of this
contract. `error_description` is human prose meant for a person reading a terminal:
it is **not** part of the contract, it is not stable, and it will be reworded without
notice. Matching on it produces integrations that break on a copy edit.

```
  {"error":"invalid_client",
   "error_description":"the basket for this client requires mTLS: use the mTLS host and present the certificate"}
             ^ match on this           ^ never parse this
```

Note that four error identifiers are Portuguese words, while the descriptions
around them are English: `interaction_nao_encontrada`, `sem_certificado_de_cliente`,
`as_indisponivel`, `as_timeout`. That is not an oversight — an identifier is an opaque
string, and renaming one would break every integration that matches it. Match them
literally; do not translate them. The full list is in [04 — Errors](04-errors.md).

## Notation

Anything written as `<name>` is a placeholder, not a value to copy: `<client_id>`,
`<access_token>`, `<registration_access_token>`, and so on. Every credential in this
track is shown that way on purpose — an example carrying a real secret teaches the
wrong habit, and every credential here expires within 24 hours anyway.

Values that are **not** placeholders are real captures: entity identifiers, account
and transaction identifiers, and timestamps. They are shown literally because their
format is part of what you need to recognise, and because none of them authenticates
anything.

**They are illustrative, and they will not match yours.** Entities are created fresh
with every basket, and accounts and statement history are derived from the entity
identifier — so a different client sees different accounts. Transaction identifiers
are generated per creation. Never hard-code any of them: read them from the response
that produced them.

## Discovery

Everything below is discoverable at runtime. Prefer reading this document over
hard-coding endpoints:

```http
GET /.well-known/oauth-authorization-server HTTP/1.1
Host: oauth.certauth.dev
```

```json
{
  "issuer": "https://oauth.certauth.dev",
  "authorization_endpoint": "https://oauth.certauth.dev/authorize",
  "token_endpoint": "https://oauth.certauth.dev/token",
  "revocation_endpoint": "https://oauth.certauth.dev/revoke",
  "registration_endpoint": "https://oauth.certauth.dev/register",
  "jwks_uri": "https://oauth.certauth.dev/jwks",
  "scopes_supported": ["accounts:read","transactions:read","transactions:write",
                       "profile:read","profile:write"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code","client_credentials","refresh_token"],
  "token_endpoint_auth_methods_supported": ["client_secret_basic","client_secret_post",
                                            "private_key_jwt"],
  "code_challenge_methods_supported": ["S256"],
  "pushed_authorization_request_endpoint": "https://oauth.certauth.dev/par",
  "request_parameter_supported": true,
  "request_object_signing_alg_values_supported": ["RS256"],
  "authorization_details_types_supported": ["payment_initiation"],
  "dpop_signing_alg_values_supported": ["RS256","ES256"],
  "mtls_endpoint_aliases": {
    "token_endpoint": "https://oauth-mtls.certauth.dev/token",
    "revocation_endpoint": "https://oauth-mtls.certauth.dev/revoke",
    "registration_endpoint": "https://oauth-mtls.certauth.dev/register",
    "pushed_authorization_request_endpoint": "https://oauth-mtls.certauth.dev/par"
  }
}
```

Two absences are deliberate and worth noticing.

There is **no `introspection_endpoint`**. Token introspection exists, but it is
internal infrastructure between the Resource Server and the Authorization Server,
authenticated by a fixed client. It is not part of the surface offered to
integrators, and it is not reachable over mutual TLS.

There is **no `userinfo_endpoint`**. This is OAuth 2.0, not OpenID Connect. Subject
information comes from the Resource Server at `GET /api/profile`.

## What is not available to an unattended integration

| Capability | Why |
|---|---|
| Creating an RFC basket | Manual by design — a person, in the browser. No supported automation; see [01 — Bootstrap](01-bootstrap.md) |
| Rich Authorization Requests (RFC 9396) | Restricted to the authorization code grant, which requires a consent screen. A client using only `client_credentials` cannot exercise RAR at all. |
| Obtaining the first refresh token | Refresh tokens are issued by the authorization code grant. Once you hold one, refreshing is unattended — but acquiring it is not. |

Everything else — registration, maintenance, key issuance, certificate issuance,
token acquisition, resource access, revocation — is reachable over HTTP with no
browser involved.
