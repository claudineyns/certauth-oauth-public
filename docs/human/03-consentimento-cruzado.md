# 03 — Consentimento cruzado

[← 02 Authorization Code](02-authorization-code.md) · [índice](README.md) · a seguir: [04 — Revogação](04-revogacao.md)

Segundo cenário: **o cliente acessa dados da PJ que não é dona dele**. É o mesmo fluxo
do capítulo anterior, olhado do ângulo de quem exerce acesso delegado.

## Por que as PJs nascem em par

Toda cesta cria duas pessoas jurídicas ligadas entre si:

| `role` | Vínculo com o cliente |
|---|---|
| `client_owner` | é a dona dele; é quem o `client_credentials` alcança |
| `third_party` | nenhum; só é alcançada por Authorization Code, com consentimento |

Ambas têm contas e perfil. A diferença inteira está no vínculo com o `client_id`: com
uma só PJ, um cliente lendo os próprios dados não exercitaria delegação nenhuma.

## Quem o token representa

A regra que organiza tudo no Resource Server: **toda resposta devolve os dados do `sub`
do token, nunca de quem está segurando o token.**

No fluxo do capítulo anterior, quem se autenticou na tela foi a `third_party`. Então:

```
PJ client_owner              A9701183A1D854
PJ third_party (autenticou)  ED01D5E47F3A78
```

```http
GET /api/accounts HTTP/1.1
Authorization: Bearer <access_token>
```
```json
{"holder": "ED01D5E47F3A78", "accounts": [ ... ] }
```

O cliente recebeu dados da `third_party` — porque foi ela quem entrou e consentiu. O
identificador da PJ dona do cliente não aparece em lugar nenhum da resposta.

É daí que sai o isolamento, e ele é **estrutural, não uma checagem**: não existe
endpoint que aceite um identificador de PJ como parâmetro. Não há como pedir os dados
de outra pessoa jurídica, porque não há onde escrever o pedido. O `sub` do token é a
única fonte, e ele foi fixado no momento do login.

## Limites do escopo

No exemplo acima o consentimento foi só `accounts:read`. Peça o perfil:

```http
GET /api/profile HTTP/1.1
Authorization: Bearer <access_token>
```

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope", scope="profile:read"
```
```json
{"error":"insufficient_scope","error_description":"required scope(s): profile:read"}
```

O cabeçalho `WWW-Authenticate` **nomeia o que falta**, e convém lê-lo: em
`GET /api/accounts/{id}/statement`, por exemplo, faltam dois escopos ao mesmo tempo —
`accounts:read` **e** `transactions:read`.

Os cinco escopos:

| Escopo | Alcança |
|---|---|
| `accounts:read` | Contas, saldos e os endpoints de consentimento |
| `transactions:read` | Histórico e transações individuais |
| `transactions:write` | Criação de transações |
| `profile:read` | Leitura do perfil da PJ |
| `profile:write` | Alteração do nome de exibição, e só dele |

Um escopo concedido a mais é permanente até o token expirar — não há como encolher um
token emitido. Peça o mínimo e emita outro quando precisar de mais.

## O objeto de consentimento

O consentimento tem identidade própria e pode ser consultado:

```http
GET /api/consents HTTP/1.1
Authorization: Bearer <access_token>
```

```json
{"consents":[{"consent_id":"<client_id>.677410B4965245",
              "client_id":"<client_id>",
              "legal_entity":"677410B4965245",
              "scopes":["accounts:read"],
              "granted_at":"2026-08-29T22:33:34.830Z"}]}
```

O `consent_id` é a composição `{client_id}.{pj_id}` — as duas partes, no próprio nome.

Esses dois endpoints — listar e revogar — **só existem para tokens de Authorization
Code**. Sob `client_credentials` não há delegação: o cliente age por si, e não há
consentimento a listar. Pedir com um token desses devolve erro, por razão conceitual e
não por restrição arbitrária.

## Registro via API (RFC 7591)

O capítulo 01 registrou o cliente pelo caminho curto. Quem quiser exercitar o
**Dynamic Client Registration** como protocolo tem a RFC 7591 completa, e a RFC 7592
para a manutenção — `GET`, `PATCH` e `DELETE` sobre `/register/{client_id}`,
autenticados pelo token de manutenção.

Os dois caminhos são equivalentes por construção: um cliente criado pela tela e outro
criado por `POST /register`, com a mesma cesta, produzem registros indistinguíveis.

📖 [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591) · [RFC 7592](https://datatracker.ietf.org/doc/html/rfc7592)

---

[← 02 Authorization Code](02-authorization-code.md) · [índice](README.md) · a seguir: [04 — Revogação](04-revogacao.md)
