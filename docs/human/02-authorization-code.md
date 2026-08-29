# 02 — Authorization Code, do começo ao saldo

[← 01 Primeiros passos](01-primeiros-passos.md) · [índice](README.md) · a seguir: [03 — Consentimento cruzado](03-consentimento-cruzado.md)

Primeiro cenário da especificação: **a PJ correntista consulta o próprio saldo**, por
um cliente que agiu em nome dela.

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

Dois detalhes que derrubam integrações: é **base64url sem padding**, não base64; e o
hash é sobre a string ASCII do verifier, não sobre os bytes aleatórios originais.

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

O AS não desenhou nada. Ele guardou os parâmetros, criou uma **interação** com prazo de
5 minutos, e mandou o browser para a interface — que é quem tem o HTML.

Essa separação é o coração do playground: **a UI renderiza, o AS decide**. O
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

{"interaction": "<interaction>", "pj_id": "<pj_id>", "senha": "goldsmith"}
```

Com senha errada:

```json
{"error":"access_denied","error_description":"identificador de PJ ou senha invalidos"}
```

Com a senha certa, o AS avança o estado e diz para onde ir:

```json
{"interaction":"<interaction>",
 "proximo":"/consent?interaction=<interaction>",
 "pj_autenticada":"677410B4965245"}
```

Note que a interação **muda de identificador** ao avançar. Não guarde o antigo.

## Passo 3 — consentir

Aqui está a demonstração mais elegante do desenho. Peça, de propósito, um escopo que
**não** foi solicitado no `/authorize`:

```http
POST /consent HTTP/1.1
Content-Type: application/json

{"interaction": "<interaction>", "scopes": ["accounts:read", "profile:write"]}
```

```json
{"redirect_to":"https://cliente.exemplo/callback?code=<code>&state=a1b2c3",
 "scopes_concedidos":["accounts:read"]}
```

O `profile:write` **sumiu**. Não houve erro, não houve recusa — o AS simplesmente
intersectou o que a tela mandou com o que ele havia guardado no `/authorize`, e
concedeu só o que constava de lá.

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

### Se o verifier estiver errado

```json
{"error":"invalid_grant","error_description":"code_verifier nao corresponde ao code_challenge"}
```

E aqui vem o detalhe que custa tempo a quem não sabe: **o código foi consumido na
tentativa falha**. Repetindo com o verifier correto:

```json
{"error":"invalid_grant","error_description":"code invalido, expirado ou ja usado"}
```

Não é bug. Um authorization code vale **uma vez**, com sucesso ou sem. Errou o
verifier, recomece do `/authorize` — não adianta corrigir e repetir.

## Passo 5 — o saldo

```http
GET /api/accounts/<account_id>/balance HTTP/1.1
Host: resource.certauth.dev
Authorization: Bearer <access_token>
```

```json
{"account_id": "ACA5F94A33A3",
 "saldo_disponivel": 86714.29,
 "moeda": "BRL",
 "atualizado_em": "2026-08-29T22:31:21.289Z"}
```

Descubra o `account_id` com `GET /api/accounts` — ele deriva da PJ e será diferente do
mostrado aqui.

## Renovar sem incomodar ninguém

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
