# 02 — Authorization Code, do começo ao saldo

[← 01 Primeiros passos](01-primeiros-passos.md) · [índice](README.md) · a seguir: [03 — Consentimento cruzado](03-consentimento-cruzado.md)

Primeiro cenário da especificação: **uma PJ consulta o próprio saldo**, por um cliente
que agiu em nome dela — com login e consentimento na tela.

## Antes: o par PKCE

Se a cesta tem `pkce: true`, o fluxo começa antes do primeiro request. Você gera um
segredo aleatório — o `code_verifier` — e envia ao `/authorize` apenas o **resumo**
dele, o `code_challenge`. Só na troca do código o verifier aparece, provando que quem
troca é quem pediu.

```python
# ilustrativo — RFC 7636 secao 4.1 e 4.2
verifier  = b64url(secrets.token_bytes(32))
challenge = b64url(hashlib.sha256(verifier.encode()).digest())
#           ^^^^^^ base64url SEM padding, sobre o hash CRU
# verifier  -> <code_verifier>    (guarde: some usado no /token)
# challenge -> <code_challenge>   (vai no /authorize)
```

```js
// ilustrativo — RFC 7636 secao 4.1 e 4.2
const verifier  = crypto.randomBytes(32).toString('base64url');
const challenge = crypto.createHash('sha256').update(verifier).digest('base64url');
//                                                              ^^^^^^^^^^^ base64url,
//                                                              nunca base64 comum
// verifier  -> <code_verifier>
// challenge -> <code_challenge>
```

> **Erros comuns.** É **base64url sem padding**, não base64; e o hash é calculado
> sobre a string ASCII do verifier, não sobre os bytes aleatórios originais.

📖 [RFC 7636 §4.1–4.2](https://datatracker.ietf.org/doc/html/rfc7636#section-4.1)

## Passo 1 — `/authorize`

```http
GET /authorize?response_type=code
   &client_id=<client_id>
   &redirect_uri=https://cliente.exemplo/callback
   &scope=accounts:read%20transactions:read
   &state=a1b2c3
   &code_challenge=<code_challenge>
   &code_challenge_method=S256 HTTP/1.1
Host: oauth.certauth.dev
```

```http
HTTP/1.1 302 Found
Location: https://oauth-playground.certauth.dev/login?interaction=<interaction>
```

O AS não renderizou nada. Ele guardou os parâmetros, criou uma **interação** com prazo de
5 minutos, e mandou o browser para a interface — que é quem tem o HTML.

Essa separação é o princípio central do playground: **a UI renderiza, o AS decide**. O
identificador da interação é a única coisa que atravessa os dois domínios, e ele é
opaco.

No browser, você vê a tela de identificação da PJ. Por baixo, a submissão vai da página
para o backend da UI (mesma origem) e dele para o AS (servidor a servidor) — nunca
direto do browser para o AS. É o que dispensa cookie entre domínios.

## Passo 2 — autenticar a PJ

A senha de todas as PJs deste sandbox é `goldsmith`.

```http
POST /login HTTP/1.1
Host: oauth.certauth.dev
Content-Type: application/json

{"interaction": "<interaction>", "pj_id": "<pj_id>", "password": "goldsmith"}
```

Com senha errada:

```json
{"error":"access_denied","error_description":"invalid legal entity identifier or password"}
```

Com a senha certa, o AS avança o estado e diz para onde ir:

```json
{"interaction":"<interaction>",
 "next":"/consent?interaction=<interaction>",
 "authenticated_legal_entity":"677410B4965245"}
```

A interação **muda de identificador** ao avançar; o anterior deixa de valer.

## Passo 3 — consentir

Peça, de propósito, um escopo que **não** foi solicitado no `/authorize`:

```http
POST /consent HTTP/1.1
Content-Type: application/json

{"interaction": "<interaction>", "scopes": ["accounts:read", "profile:write"]}
```

```json
{"redirect_to":"https://cliente.exemplo/callback?code=<code>&state=a1b2c3",
 "granted_scopes":["accounts:read"]}
```

O `profile:write` não aparece na resposta. Não houve erro nem recusa: o AS
intersectou o que a tela enviou com o que ele havia guardado no `/authorize`, e
concedeu apenas o que constava de lá.

É a garantia central: **o AS nunca confia no que volta da UI**. Ele não valida o que
chega; ele reconstrói a decisão a partir do que ele mesmo armazenou. Uma UI adulterada
não consegue ampliar escopo, porque o escopo nunca esteve nas mãos dela.

## Passo 4 — trocar o código pelo token

```http
POST /token HTTP/1.1
Host: oauth.certauth.dev
Authorization: Basic <base64(client_id:client_secret)>
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=<code>
&redirect_uri=https://cliente.exemplo/callback
&code_verifier=<code_verifier>
```

```json
{"access_token":"<access_token>",
 "token_type":"Bearer",
 "expires_in":3600,
 "scope":"accounts:read transactions:read",
 "refresh_token":"<refresh_token>"}
```

### Erros comuns na troca do código

```json
{"error":"invalid_grant","error_description":"code_verifier does not match the code_challenge"}
```

**O código foi consumido na tentativa falha.** Repetindo com o verifier correto:

```json
{"error":"invalid_grant","error_description":"code invalid, expired or already used"}
```

Um authorization code vale **uma vez**, com sucesso ou sem. Corrigir o verifier e
repetir não resolve: é preciso recomeçar do `/authorize`.

## Passo 5 — o saldo

```http
GET /api/accounts/<account_id>/balance HTTP/1.1
Host: resource.certauth.dev
Authorization: Bearer <access_token>
```

```json
{"account_id": "AC618695DA4E",
 "available_balance": 0,
 "currency": "BRL",
 "updated_at": "2026-08-30T06:02:49.011Z"}
```

Descubra o `account_id` com `GET /api/accounts` — ele deriva da PJ e será diferente do
mostrado aqui.

A conta nasce vazia: o saldo é exatamente a soma do que foi lançado, e nada foi
lançado ainda. Não há histórico fictício — o extrato de uma conta nova vem `[]`.

Para haver saldo, registre um depósito:

```http
POST /api/transactions HTTP/1.1
Host: resource.certauth.dev
Authorization: Bearer <access_token>
Content-Type: application/json

{"account_id": "<account_id>", "amount": 2500.00,
 "type": "deposit", "direction": "credit", "description": "Aporte inicial"}
```

Consultado de novo, o saldo terá se movido. Até 30/08/2026 as transações eram
gravadas sem alterar o saldo, e a lista divergia do total.

Os quatro campos obrigatórios, e a regra que liga dois deles:

| Campo | Valor |
|---|---|
| `amount` | positivo, no máximo duas casas |
| `type` | `pix`, `boleto`, `deposit` ou `withdrawal` |
| `direction` | `debit` ou `credit` |
| `description` | opcional |

`deposit` só credita e `withdrawal` só debita; pedir o contrário é recusado. `pix` e
`boleto` aceitam os dois sentidos. Campos que o servidor não conhece são ignorados,
não recusados.

> **Ponto de atenção.** Não há checagem de fundos. Um débito maior que o saldo é
> aceito e deixa a conta negativa — é o limite especial da conta, deliberado.

## Renovação do token

O `refresh_token` é o que torna a operação seguinte desassistida:

```http
POST /token HTTP/1.1
Authorization: Basic <base64(client_id:client_secret)>
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=<refresh_token>
```

Ele é **não rotativo**: continua valendo depois do uso, pelas 24 horas de vida. Muitos
servidores devolvem um refresh novo a cada troca; este não, e é escolha consciente —
menos peça móvel para um sandbox.

---

[← 01 Primeiros passos](01-primeiros-passos.md) · [índice](README.md) · a seguir: [03 — Consentimento cruzado](03-consentimento-cruzado.md)
