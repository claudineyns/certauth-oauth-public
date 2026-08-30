# OAuth 2.0 Playground — Documentation

A public sandbox where you assemble your own basket of OAuth 2.0 RFCs and exercise
each flow against a working system. No code required to start; no code required at
all if you only want to watch the protocol work.

Two tracks. They cover the same system from opposite ends.

| Track | Audience | Language | Form |
|---|---|---|---|
| [`human/`](human/) | Someone driving the flows by hand, with a browser | Português | Walkthrough, with illustrative Python and Node snippets |
| [`machine/`](machine/) | An agent or service integrating without human intervention | English | HTTP on the wire, plus pseudocode |

## Which one do you want

Pick **`human/`** if you want to see the consent screen, understand what each RFC
buys you, and drive the flows yourself. That track covers everything the playground
does, including the parts that require a person in front of a browser.

Pick **`machine/`** if you are building unattended integration. That track is a
contract: request shapes, response shapes, error codes, lifetimes, and the rules
that decide when a token stops working. It carries no code in any programming
language, because an agent needs the wire format, not a translation of it.

The two tracks are not translations of each other. There is one thing worth reading
in `machine/` even if you are human: [the manual step](machine/01-bootstrap.md) that
begins every integration, and why it is a design decision rather than a gap.

## The system behind both tracks

Five public hostnames, each terminating its own TLS:

| Host | Role |
|---|---|
| `oauth-playground.certauth.dev` | UI and its backend-for-frontend |
| `oauth.certauth.dev` | Authorization Server |
| `oauth-mtls.certauth.dev` | Authorization Server, mutual TLS |
| `resource.certauth.dev` | Resource Server |
| `resource-mtls.certauth.dev` | Resource Server, mutual TLS |

Everything is ephemeral. Every record the sandbox keeps expires within 24 hours,
most of it much sooner. Nothing you create here is meant to survive the day.
