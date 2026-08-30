# Security Policy

*Relatos são aceitos em português ou inglês. This document is in English because a
public service is probed by people from anywhere.*

## What this is

The OAuth 2.0 Playground at `certauth.dev` is a **public sandbox for exercising OAuth
2.0 flows**. It exists to be poked at: you assemble a basket of RFCs, register a
client, and drive the protocol by hand.

That makes the usual scope question sharper than normal. Several things that would be
findings elsewhere are **deliberate choices here**, documented and assessed. Others are
genuine defects. This document draws the line so you do not spend time re-discovering a
decision, and so we do not spend time re-explaining one.

No real personal or financial data exists in this system. Every legal entity, account
and transaction is synthetic, and every record expires within 24 hours.

## Reporting

**Report privately, through GitHub:**

> https://github.com/claudineyns/certauth-oauth-public/security/advisories/new

Please do not open a public issue for anything exploitable. The service is live, and a
public description gives a working recipe to anyone reading before a fix exists.

Useful in a report: the request and response that show it, the basket configuration of
the client you used, and what an attacker gains. A `client_id` is enough to identify
the scenario — never include a `client_secret` or a private key.

**What to expect.** This is a personal teaching project, not a funded product. Reports
are read and answered on a best-effort basis. There is no service level, no bounty, and
no guaranteed timeline. What we do commit to is an honest answer: either the finding is
real and gets fixed, or we explain why it is a decision rather than a defect.

## In scope

Anything that breaks a guarantee the system claims to make:

- **Obtaining a token you should not have** — a `client_secret` accepted when the
  basket requires mutual TLS or a client assertion, PKCE bypassed, a scope granted
  beyond what was consented, a token issued without the consent screen.
- **Reaching another legal entity's data** — any response returning data for a subject
  other than the token's `sub`, or beyond the consent that backs it.
- **Replaying something meant to be used once** — the authorization code, the initial
  registration token, a DPoP proof, a RAR payment token, a client assertion `jti`, a
  PAR `request_uri`.
- **Defeating certificate binding** — using a token bound to one client certificate
  while presenting a different one, or after a reissue.
- **Touching another client's registration** — reading, editing or deleting a client
  with a maintenance token that does not belong to it.
- **Injection in the UI origin** — anything that executes script on
  `oauth-playground.certauth.dev`, including through a `redirect_uri`.
- **Crashing a workload** — an input that terminates a process rather than returning an
  error. Unhandled exceptions count.
- **Reading anything the server should not disclose** — a private key after it was
  issued, the root CA key, another user's material.

## Out of scope — deliberate design decisions

These are recorded choices, not oversights. Reporting them is welcome as discussion,
but they will not be treated as vulnerabilities.

| Choice | Why |
|---|---|
| **`Access-Control-Allow-Origin: *`** | There is no cookie and no session; every credential is an explicit header the caller must already hold. With no ambient authority, a restrictive origin policy would remove nothing from an attacker who already has the token, and would break legitimate use from your own pages. Reassessed after a confirmed XSS and kept. |
| **No rate limiting** | Out of scope for this phase. Resource exhaustion is bounded instead — memory caps with eviction, per-service CPU and memory limits, and concurrency caps on the expensive endpoints, which answer `503` with `Retry-After`. |
| **Self-service registration** | Anyone can build a basket and register a client. That is the product. |
| **The sandbox password is `goldsmith`** | Fixed, identical for every legal entity, and printed on the login screen. Authenticating a synthetic entity is not a secret worth protecting. |
| **`client_secret` is never regenerated** | Lose it and you register a new client. There is no rotation endpoint by design. |
| **Everything expires in 24 hours** | Clients, tokens, consents, certificates, entities. Short-lived by construction; "the credential still worked an hour later" is expected. |
| **Certificates from the sandbox CA** | The mTLS root is this project's own. Certificates it issues are meaningless anywhere else, and that is intentional. |
| **`error_description` is not a contract** | It is prose for a human, reworded freely. Integrations must branch on the `error` code. |
| **No audit log** | The project keeps no record of who did what. Deliberate: nothing here is worth auditing. |
| **Missing hardening that does not apply** | The service holds no session, sets no cookie, and serves no third-party content. Findings that assume otherwise do not apply. |

## Testing guidelines

The sandbox runs on a small shared machine. A few requests, honestly:

- **No load or denial-of-service testing.** Concurrency caps exist and will answer
  `503`; please treat that as the answer rather than something to push past. If you
  believe you have found an amplification vector, describe it — do not demonstrate it
  at scale.
- **Use your own clients.** Build your own basket; every basket creates its own pair of
  legal entities. Do not operate on registrations you did not create.
- **Do not upload anything real.** No real names, documents, account numbers or keys.
  There is nowhere for them to belong.
- **Stay in the application.** Attempts to reach the host operating system, the
  container runtime or the certificate authority's private key are out of bounds —
  report the path you found instead of walking it.

## Acknowledgments

The first round of security findings on this project came from an independent QA
review, which identified a stored `javascript:` `redirect_uri` leading to script
execution on the UI origin, and a missing replay control on client assertions. Both
were fixed. The review, and the response to it, are part of the project history.
