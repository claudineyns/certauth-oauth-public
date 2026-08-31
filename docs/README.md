# OAuth 2.0 Playground — Documentation

A public sandbox where you assemble your own basket of OAuth 2.0 RFCs and exercise
each flow against a working system. No code required to start; no code required at
all if you only want to watch the protocol work.

Two tracks over the same system. The axis between them is **how you work with it**,
not what you are: one walks you through the flows in a browser, the other states the
contract you implement against.

| Track | Audience | Language | Form |
|---|---|---|---|
| [`human/`](human/) | Someone driving the flows by hand, with a browser | Português | Walkthrough, with illustrative Python and Node snippets |
| [`machine/`](machine/) | Anyone implementing a client — a software engineer, or an agent | English | The wire contract, plus pseudocode |

## Which one do you want

Pick **`human/`** if you want to see the consent screen, understand what each RFC
buys you, and drive the flows yourself. That track covers everything the playground
does, including the parts that require a person in front of a browser.

Pick **`machine/`** if you are implementing a client. It is written for whoever
writes that client — a software engineer building an integration, or an agent driving
one — and describes the system as a contract: request shapes, response shapes, error
codes, lifetimes, and the rules that decide when a token stops working.

It carries no code in any programming language, and that is deliberate. The exchange
on the wire serves an implementation in Java, Go, Python or Rust equally; a sample in
one of those is dead weight for the other three, and tends to be copied rather than
understood.

The two tracks are not translations of each other, and `machine/` is the only one in
English. If you are implementing a client and do not read Portuguese, that is your
track: the Python and Node snippets in `human/` illustrate a walkthrough, they are not
a client library.

One section is worth reading in `machine/` whichever track you chose: [the manual
step](machine/01-bootstrap.md) that begins every integration, and why it is a design
decision rather than a gap.

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
