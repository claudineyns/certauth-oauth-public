# 03 — Consentimento cruzado

[← 02 Authorization Code](02-authorization-code.md) · [índice](README.md) · a seguir: [04 — Revogação](04-revogacao.md)

Segundo cenário: **um terceiro acessa dados da PJ correntista**. É o mesmo fluxo do
capítulo anterior, olhado de outro ângulo — o de quem *não* é dono da conta.

## As duas PJs, e por que elas nascem em par

Toda cesta cria duas pessoas jurídicas ligadas entre si:

| Papel | Quem é | No enredo |
|---|---|---|
| `titular` | Terceiro Serviços Digitais SA | Quem quer acessar |
| `correntista` | Correntista Indústria e Comércio LTDA | Quem tem a conta e autoriza |

O cliente que você registrou representa o **terceiro**. As contas e o histórico
pertencem à **correntista**. Entre um e outro só existe uma ponte: o consentimento que
a correntista concede na tela, e que o token carrega.

Sem essa dupla não haveria o que demonstrar — um cliente lendo os próprios dados não
exercita delegação nenhuma.

## Quem o token representa

A regra que organiza tudo no Resource Server: **toda resposta devolve os dados do `sub`
do token, nunca de quem está segurando o token.**

No fluxo do capítulo anterior, quem se autenticou na tela foi a correntista. Então:

```
PJ titular    (terceiro)     C9B5DE4DDBF283
PJ correntista (autenticou)  677410B4965245
```

```http
GET /api/accounts HTTP/1.1
Authorization: Bearer <access_token>
```
```json
{"titular": "677410B4965245", "accounts": [ ... ] }
```

O cliente do terceiro recebeu dados da **correntista** — porque foi ela quem entrou e
consentiu. O identificador do terceiro não aparece em lugar nenhum da resposta.

É daí que sai o isolamento, e ele é **estrutural, não uma checagem**: não existe
endpoint que aceite um identificador de PJ como parâmetro. Não há como pedir os dados
de outra pessoa jurídica, porque não há onde escrever o pedido. O `sub` do token é a
única fonte, e ele foi fixado no momento do login.

## O escopo limita de verdade

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

O cabeçalho `WWW-Authenticate` **nomeia o que falta**. Vale ler antes de sair
adivinhando: em `GET /api/accounts/{id}/extract`, por exemplo, faltam dois escopos ao
mesmo tempo — `accounts:read` **e** `transactions:read`.

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

## O consentimento é um objeto, não um efeito colateral

Ele existe, tem identidade e pode ser consultado:

```http
GET /api/consents HTTP/1.1
Authorization: Bearer <access_token>
```

```json
{"consents":[{"consent_id":"<client_id>.677410B4965245",
              "client_id":"<client_id>",
              "pj":"677410B4965245",
              "scopes":["accounts:read"],
              "granted_at":"2026-08-29T22:33:34.830Z"}]}
```

O `consent_id` é a composição `{client_id}.{pj_id}` — as duas pontas da ponte, no
próprio nome.

Esses dois endpoints — listar e revogar — **só existem para tokens de Authorization
Code**. Sob `client_credentials` não há delegação: o cliente age por si, e não há
consentimento a listar. Pedir com um token desses devolve erro, e o motivo é conceitual,
não uma restrição arbitrária.

## Registro por API, se quiser exercitar

O capítulo 01 registrou o cliente pelo caminho curto. Quem quiser exercitar o
**Dynamic Client Registration** como protocolo tem a RFC 7591 completa, e a RFC 7592
para a manutenção — `GET`, `PATCH` e `DELETE` sobre `/register/{client_id}`,
autenticados pelo token de manutenção.

Os dois caminhos são equivalentes por construção: um cliente criado pela tela e outro
criado por `POST /register`, com a mesma cesta, produzem registros indistinguíveis.

📖 [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591) · [RFC 7592](https://datatracker.ietf.org/doc/html/rfc7592)

---

[← 02 Authorization Code](02-authorization-code.md) · [índice](README.md) · a seguir: [04 — Revogação](04-revogacao.md)
