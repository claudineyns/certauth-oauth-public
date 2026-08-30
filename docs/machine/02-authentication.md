# 02 — Authentication and tokens

[← 01 Bootstrap](01-bootstrap.md) · [index](README.md) · next: [03 — Resources](03-resources.md)

## The basket picks the mechanism — you do not

Three client authentication mechanisms exist. Exactly one applies to your client, and
which one is decided by the basket, not by what you send. Offering a second mechanism
does not create a fallback; it produces an error.

```
  if basket.client_assertion is true:
      mechanism = private_key_jwt          # RFC 7523
  else if basket.proof == "mtls":
      mechanism = tls_client_auth          # RFC 8705
  else:
      mechanism = client_secret_basic | client_secret_post
```

This matters because the failure is explicit rather than silent. A client whose basket
selects assertions, presenting a perfectly valid secret:

```json
{"error":"invalid_client",
 "error_description":"a cesta deste client exige client_assertion (RFC 7523); client_secret nao e aceito"}
```

And the reverse — a client whose basket does not include assertions, sending one —
is refused as well.

## Mechanism 1 — client secret

Applies when the basket has `client_assertion: false` and `proof` other than `mtls`.

```http
POST /token HTTP/1.1
Host: oauth.certauth.dev
Authorization: Basic <base64(client_id:client_secret)>
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&scope=accounts:read
```

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```
```json
{"access_token":"<access_token>",
 "token_type":"Bearer",
 "expires_in":3600,
 "scope":"accounts:read"}
```

`client_secret_post` — `client_id` and `client_secret` in the form body — is accepted
equivalently.

## Mechanism 2 — private_key_jwt (RFC 7523)

Applies when the basket has `client_assertion: true`. Requires a key pair issued
beforehand; see [01 — Bootstrap](01-bootstrap.md).

Build a JWT and sign it with the private key. Claims:

| Claim | Value |
|---|---|
| `iss` | your `client_id` |
| `sub` | your `client_id` |
| `aud` | the issuer — `https://oauth.certauth.dev` |
| `jti` | unique per assertion |
| `iat` | now |
| `exp` | short; 300 seconds is ample |

Header: `{"alg":"RS256","typ":"JWT"}`. RS256 is the only accepted algorithm here.

`aud` must be the issuer. Without that check an assertion minted for a different
server could be replayed against this one, so it is enforced rather than assumed.

```http
POST /token HTTP/1.1
Host: oauth.certauth.dev
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&scope=accounts:read
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<client_assertion>
```

Note there is no `client_id` parameter. The client is identified by the `sub` claim
inside the assertion — the identity and the proof arrive together.

`jti` and `exp` are both required, and **each `jti` is accepted once**. Re-presenting
the same assertion — byte for byte — fails, even inside its validity window:

```json
{"error":"invalid_client","error_description":"client_assertion ja apresentada (jti reutilizado)"}
```

Generate a fresh `jti` per request, exactly as you would a DPoP proof. An assertion
signed with a key that has since been reissued is also rejected: only the current
public JWK is held.

## Mechanism 3 — tls_client_auth (RFC 8705)

Applies when the basket has `proof: mtls`. Requires a client certificate issued
beforehand; see [01 — Bootstrap](01-bootstrap.md).

**Use the mutual-TLS host.** The endpoints are the same paths on a different
hostname, published under `mtls_endpoint_aliases` in the discovery document.

```http
POST /token HTTP/1.1
Host: oauth-mtls.certauth.dev
Content-Type: application/x-www-form-urlencoded
[TLS: client certificate presented]

client_id=<client_id>&grant_type=client_credentials&scope=accounts:read
```

```json
{"access_token":"<access_token>",
 "token_type":"Bearer","expires_in":3600,"scope":"accounts:read"}
```

There is no secret in that request. The certificate is the credential.

Two failure modes, distinguishable and worth handling separately:

Wrong host — the request reached the plain-TLS endpoint, where no certificate can be
presented:
```json
{"error":"invalid_client",
 "error_description":"a cesta deste client exige mTLS: use o host mTLS e apresente o certificado"}
```

Right host, wrong certificate — the thumbprint does not match the registered one,
which is what you see after a reissue:
```json
{"error":"invalid_client",
 "error_description":"o certificado apresentado nao e o registrado para este client"}
```

Connecting to the mutual-TLS host **without** any certificate fails during the TLS
handshake, before HTTP exists. Expect a transport error, not a JSON body.

The issued token carries a confirmation claim binding it to the certificate —
`cnf.x5t#S256`. See [03 — Resources](03-resources.md) for how that is enforced.

### Checking that your certificate arrives

Before debugging an authentication failure, confirm the certificate is reaching the
server at all. `/whoami-cert` echoes what the connection presented:

```http
GET /whoami-cert HTTP/1.1
Host: oauth-mtls.certauth.dev
[TLS: client certificate presented]
```
```json
{"workload": "oauth-as-mtls",
 "certificado_cliente": {
   "sha256_hex": "<sha256_hex>",
   "x5t_s256": "<x5t_s256>",
   "subject": "CN=<client_id>, O=OAuth Playground",
   "issuer": "CN=OAuth Playground Root CA, O=OAuth Playground",
   "valido_ate": "Aug 30 21:27:43 2026 GMT"}}
```

Compare `x5t_s256` against the value returned when the certificate was issued. If
they differ, the client is presenting a different certificate than it thinks.

On a connection with no client certificate the same endpoint answers `400` with
`sem_certificado_de_cliente`. This endpoint is a diagnostic; it authenticates
nothing.

## Token formats

The basket's `formato_token` decides what you receive.

**`bearer`** — an opaque hex string. It carries no information; the Resource Server
resolves it internally.

**`jwt`** — a signed JWT following RFC 9068. You can inspect it, and the Resource
Server validates it locally without a round trip. Real example, decoded:

```json
{"alg":"RS256","typ":"at+jwt","kid":"<kid>"}
```
```json
{
  "iss": "https://oauth.certauth.dev",
  "aud": "https://resource.certauth.dev",
  "sub": "6093D75EF38D79",
  "client_id": "<client_id>",
  "scope": "accounts:read",
  "jti": "<jti>",
  "iat": 1788038834,
  "exp": 1788042434
}
```

`typ` is `at+jwt`, not `JWT` — that is RFC 9068 distinguishing an access token from
any other JWT, and a validator should check it.

Verify against `jwks_uri`. The `kid` is derived deterministically from the key's JWK
thumbprint (RFC 7638), which is why both Authorization Server workloads publish an
identical JWK set: a token minted by one verifies against the other.

**Do not treat an opaque token as a JWT because it happens to be long, or a JWT as
opaque.** The basket tells you which you have, before the first request.

## DPoP (RFC 9449)

Applies when the basket has `proof: dpop`. The token is bound to a key you hold; the
key travels in each proof, and possession is proven on every call.

A proof is a JWT. Header:

| Field | Value |
|---|---|
| `typ` | `dpop+jwt` — exact |
| `alg` | `RS256` or `ES256` |
| `jwk` | your **public** key. A proof carrying private material is rejected. |

Payload:

| Claim | Value |
|---|---|
| `htm` | the HTTP method, uppercase |
| `htu` | the target URL, **without query string or fragment** |
| `iat` | now; must be within ±300 seconds |
| `jti` | unique per proof |

Send it in a `DPoP` header alongside the token request:

```http
POST /token HTTP/1.1
Host: oauth.certauth.dev
DPoP: <dpop_proof>
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&scope=accounts:read
```

The response differs from the Bearer case in one field:

```json
{"access_token":"...","token_type":"DPoP","expires_in":3600,"scope":"accounts:read"}
```

**Every proof is single-use.** The `jti` is recorded for five minutes; presenting it
again returns `invalid_dpop_proof` with `prova DPoP ja usada (replay)`. Generate a
fresh proof per request — including retries. Reusing a proof on a retry is the most
likely way to break a DPoP integration.

When calling the Resource Server, the scheme changes too:

```http
GET /api/accounts HTTP/1.1
Host: resource.certauth.dev
Authorization: DPoP <access_token>
DPoP: <a fresh proof, with htm=GET and htu=https://resource.certauth.dev/api/accounts>
```

Presenting a DPoP token under the `Bearer` scheme is refused, and so is the reverse.

## Grants

`grant_types_supported` is `authorization_code`, `client_credentials`,
`refresh_token`.

**`client_credentials`** is the unattended path. The client acts as the `titular`
entity — the third party — on its own behalf. No consent screen is involved because
there is no second party to consent.

**`authorization_code`** requires a browser: the authorization endpoint redirects to
a login screen and then a consent screen, both rendered by the UI. An agent cannot
complete it. It is the only grant that produces cross-entity access and the only one
that can carry Rich Authorization Requests.

**`refresh_token`** is unattended, but the first refresh token comes from the
authorization code grant. Non-rotating: the same refresh token stays valid for its
full lifetime and is not replaced on use.

```http
POST /token HTTP/1.1
Host: oauth.certauth.dev
Authorization: Basic <base64(client_id:client_secret)>
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=<refresh_token>
```

## Revocation (RFC 7009)

```http
POST /revoke HTTP/1.1
Host: oauth.certauth.dev
Authorization: Basic <base64(client_id:client_secret)>
Content-Type: application/x-www-form-urlencoded

token=<access_token>
```
```http
HTTP/1.1 200 OK
```

The effect is immediate. The next resource call with that token:

```json
{"error":"invalid_token","error_description":"token inativo, expirado ou revogado"}
```

Per RFC 7009, revoking an unknown or already-revoked token also returns `200`.
**Do not use the status code to infer whether a token existed.**

Authenticate this request with the same mechanism the basket dictates — a client
using mutual TLS revokes on the mutual-TLS host.

---

[← 01 Bootstrap](01-bootstrap.md) · [index](README.md) · next: [03 — Resources](03-resources.md)
