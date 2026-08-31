# 05 — Mecanismos de segurança

[← 04 Revogação](04-revogacao.md) · [índice](README.md)

Quarto cenário, e o mais longo: exercitar cada mecanismo que a cesta oferece — e, em
cada um, **provocar a falha de propósito**. O valor pedagógico está tanto no caminho
que funciona quanto no que não funciona, e cada mensagem de erro abaixo foi capturada
deste sistema.

Lembre-se de que a cesta é imutável: cada mecanismo abaixo exige **um cliente
próprio**, montado com aquela combinação.

## PKCE — RFC 7636

Coberto em detalhe no [capítulo 02](02-authorization-code.md). O resumo do que quebra:

```json
{"error":"invalid_grant","error_description":"code_verifier does not match the code_challenge"}
```

E a consequência que surpreende: **o código foi consumido na tentativa falha**. Corrigir
o verifier e repetir devolve `code invalid, expired or already used`. Recomece do
`/authorize`.

📖 [RFC 7636 §4](https://datatracker.ietf.org/doc/html/rfc7636#section-4)

## DPoP — RFC 9449

Amarra o token a uma chave que só você tem. A chave pública viaja no cabeçalho da
prova; o AS registra o thumbprint dela no token, e daí em diante cada chamada precisa
de uma prova nova.

```python
# ilustrativo — RFC 9449 secao 4.2
cabecalho = {"typ": "dpop+jwt", "alg": "ES256", "jwk": jwk_publica}
corpo     = {"htm": "POST", "htu": "https://oauth.certauth.dev/token",
             "iat": int(time.time()), "jti": secrets.token_hex(16)}
entrada = f"{b64url_json(cabecalho)}.{b64url_json(corpo)}"
der  = chave.sign(entrada.encode(), ec.ECDSA(hashes.SHA256()))
r, s = utils.decode_dss_signature(der)                 # DER -> P1363
assinatura = r.to_bytes(32, "big") + s.to_bytes(32, "big")
# prova -> <dpop_proof>
```

```js
// ilustrativo — RFC 9449 secao 4.2
const cabecalho = { typ: 'dpop+jwt', alg: 'ES256', jwk: jwkPublica };
const corpo = { htm: 'POST', htu: 'https://oauth.certauth.dev/token',
                iat: Math.floor(Date.now() / 1000),
                jti: crypto.randomBytes(16).toString('hex') };
const entrada = `${b64urlJson(cabecalho)}.${b64urlJson(corpo)}`;
const assinatura = crypto.sign('sha256', Buffer.from(entrada),
  { key: chave, dsaEncoding: 'ieee-p1363' });   // <- sem isto o Node assina em DER
// prova -> <dpop_proof>
```

**Este é o erro mais caro de depurar do playground inteiro.** O JOSE exige a assinatura
ECDSA no formato P1363 — os valores `r` e `s` concatenados, 64 bytes. Tanto o Python
quanto o Node produzem **DER** por padrão: uma assinatura bem-formada, que passa em
qualquer validação sintática, e que o servidor rejeita. Sem `dsaEncoding` no Node, ou
sem converter no Python, tudo parece certo e nada funciona.

```http
POST /token HTTP/1.1
Host: oauth.certauth.dev
DPoP: <dpop_proof>
Authorization: Basic <base64(client_id:client_secret)>

grant_type=client_credentials&scope=accounts:read
```
```json
{"access_token":"<access_token>","token_type":"DPoP","expires_in":3600,"scope":"accounts:read"}
```

O `token_type` muda para `DPoP` — e daí em diante o cabeçalho `Authorization` também
usa o esquema `DPoP`, não `Bearer`.

**A falha:** repita a mesma prova.

```json
{"error":"invalid_dpop_proof","error_description":"DPoP proof already used (replay)"}
```

Cada prova vale **uma vez**, por cinco minutos. Gere uma nova a cada requisição —
inclusive nas retentativas. Reaproveitar a prova numa retentativa transforma um erro
recuperável em outro que parece diferente e não é.

Sem prova alguma: `missing DPoP header`.

**Outra falha, a mais comum de todas:** montar o `htu` a partir da URL completa da
requisição.

```json
{"error":"invalid_dpop_proof","error_description":"htu must not contain a query string"}
```

A requisição pode ter query; o `htu` da prova, não. Para chamar
`/api/accounts?limite=5`, a prova correta traz `htu` como
`https://resource.certauth.dev/api/accounts` — sem a query. O servidor não a remove
por você, e aceitar em silêncio ensinaria um formato que outro servidor recusa.

📖 [RFC 9449 §4.2](https://datatracker.ietf.org/doc/html/rfc9449#section-4.2)

## mTLS — RFC 8705

O certificado substitui o segredo, e o token fica atrelado a ele.

O certificado sai da tela de manutenção, autenticado pelo token de manutenção. **A
chave privada é exibida uma única vez e não fica no servidor** — dele guardam-se apenas
o DN e o thumbprint.

```json
{"dn": "CN=<client_id>, O=OAuth Playground",
 "x5t_s256": "<x5t_s256>",
 "validade_dias": 1}
```

O `x5t#S256` é o SHA-256 do DER do certificado, em base64url. Você pode conferir por
conta própria:

```python
# ilustrativo — RFC 8705 secao 3.1
der = base64.b64decode("".join(l for l in pem.splitlines() if "-----" not in l))
x5t = b64url(hashlib.sha256(der).digest())    # -> <x5t_s256>
```

```js
// ilustrativo — RFC 8705 secao 3.1
const der = Buffer.from(pem.replace(/-----[^-]+-----/g, '').replace(/\s+/g, ''), 'base64');
const x5t = crypto.createHash('sha256').update(der).digest('base64url');  // -> <x5t_s256>
```

Dois enganos comuns aqui: usar o **hex** em vez dos bytes crus, e usar **base64** comum
em vez de base64url. Ambos produzem uma string plausível que não bate com nada.

Use o host `oauth-mtls.certauth.dev` — os endereços estão em `mtls_endpoint_aliases`, no
documento de discovery. **Não há segredo na requisição**: o certificado é a credencial.

**As falhas, três, e cada uma diz algo diferente:**

| Situação | Resposta |
|---|---|
| Host errado (TLS simples) | `the basket for this client requires mTLS: use the mTLS host and present the certificate` |
| Certificado não registrado | `the presented certificate is not the one registered for this client` |
| Token vinculado a certificado anterior | `the certificate for this token is no longer the one registered for the client` |

Conectar ao host mTLS **sem certificado nenhum** falha antes do HTTP existir, no
handshake TLS. Espere erro de transporte, não JSON.

### A reemissão derruba os tokens

Emitiu um certificado novo? **Todos os tokens atrelados ao anterior param de valer**,
na hora.

Isso acontece porque o Resource Server compara o certificado apresentado com **dois**
valores: o `cnf` gravado no token e o thumbprint **atualmente registrado**. A segunda
comparação é o que faz a reemissão ter efeito — sem ela, o par antigo (certificado A +
token A) continuaria funcionando alegremente.

É comportamento correto da RFC 8705, demonstrado de propósito. Se o seu acesso morreu
logo depois de você clicar em "emitir novo certificado", é isto.

📖 [RFC 8705 §3](https://datatracker.ietf.org/doc/html/rfc8705#section-3)

## Client Assertion — RFC 7523

Substitui o `client_secret` por um JWT que você assina. Exige o par de chaves emitido na
tela de manutenção — e, como no mTLS, **a privada aparece uma vez só**.

```python
# ilustrativo — RFC 7523 secao 2.2
agora = int(time.time())
corpo = {"iss": client_id, "sub": client_id,          # os dois sao o cliente
         "aud": "https://oauth.certauth.dev",         # o AS, obrigatoriamente
         "jti": secrets.token_hex(16), "iat": agora, "exp": agora + 300}
entrada = f'{b64url_json({"alg":"RS256","typ":"JWT"})}.{b64url_json(corpo)}'
assinatura = chave.sign(entrada.encode(), padding.PKCS1v15(), hashes.SHA256())
# assertion -> <client_assertion>
```

```js
// ilustrativo — RFC 7523 secao 2.2
const agora = Math.floor(Date.now() / 1000);
const corpo = { iss: clientId, sub: clientId,
                aud: 'https://oauth.certauth.dev',
                jti: crypto.randomBytes(16).toString('hex'),
                iat: agora, exp: agora + 300 };
const entrada = `${b64urlJson({ alg: 'RS256', typ: 'JWT' })}.${b64urlJson(corpo)}`;
const assinatura = crypto.sign('sha256', Buffer.from(entrada), chavePrivada);
// assertion -> <client_assertion>
```

O `aud` **precisa** ser o issuer do AS. Sem essa checagem, uma assertion emitida para
outro servidor poderia ser reapresentada aqui — então ela é verificada, não presumida.

**A tolerância de relógio é de 10 segundos** — vale para `exp`, `iat` e `nbf`, aqui e
também no request object do JAR. Se a máquina que assina estiver dessincronizada além
disso, uma assertion recém-gerada é recusada como `token expirado`, mensagem que não
aponta para a causa. Antes de procurar erro na assinatura, confira o relógio.

`jti` e `exp` são obrigatórios, e **cada `jti` vale uma vez**. Reapresentar a mesma
assertion, byte a byte, falha mesmo dentro da janela de validade:

```json
{"error":"invalid_client","error_description":"client_assertion already presented (jti reused)"}
```

É a mesma proteção que o DPoP tem contra replay — gere um `jti` novo a cada
requisição.

```http
POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&scope=accounts:read
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<client_assertion>
```

Repare que não há `client_id` no corpo: quem identifica o cliente é o `sub` de dentro da
assertion. Identidade e prova chegam juntas.

**A falha:** tente com o `client_secret`, que existe e é válido.

```json
{"error":"invalid_client",
 "error_description":"the basket for this client requires client_assertion (RFC 7523); client_secret is not accepted"}
```

Reemitir o par de chaves invalida toda assertion assinada com a chave anterior — só a
pública corrente fica guardada.

📖 [RFC 7523 §2.2](https://datatracker.ietf.org/doc/html/rfc7523#section-2.2)

## PAR — RFC 9126

Em vez de mandar os parâmetros da autorização pela URL, você os empurra antes, por
POST autenticado, e recebe uma referência opaca.

```http
POST /par HTTP/1.1
Authorization: Basic <base64(client_id:client_secret)>
Content-Type: application/x-www-form-urlencoded

response_type=code&client_id=<client_id>&redirect_uri=...&scope=accounts:read
```

```http
HTTP/1.1 201 Created
```
```json
{"request_uri":"urn:ietf:params:oauth:request_uri:<id>","expires_in":60}
```

O `request_uri` substitui todos os parâmetros:

```
GET /authorize?client_id=<client_id>&request_uri=urn:ietf:params:oauth:request_uri:<id>
```

Duas propriedades que diferenciam o PAR de tudo o mais no playground: ele vale
**60 segundos** — não 5 minutos — e é **consumido na leitura**. Reapresentá-lo falha
mesmo dentro da janela:

```json
{"error":"invalid_request","error_description":"request_uri invalid, expired or already used"}
```

📖 [RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126)

## JAR — RFC 9101

Os parâmetros viram claims de um JWT assinado pelo cliente. Usa o mesmo par de chaves da
Client Assertion.

```python
# ilustrativo — RFC 9101 secao 4
corpo = {"iss": client_id, "aud": "https://oauth.certauth.dev",
         "response_type": "code", "client_id": client_id,
         "redirect_uri": "https://cliente.exemplo/callback",
         "scope": "accounts:read", "state": "a1b2c3",
         "iat": agora, "exp": agora + 300}
# request object -> <request_object>
```

```js
// ilustrativo — RFC 9101 secao 4
const corpo = { iss: clientId, aud: 'https://oauth.certauth.dev',
                response_type: 'code', client_id: clientId,
                redirect_uri: 'https://cliente.exemplo/callback',
                scope: 'accounts:read', state: 'a1b2c3',
                iat: agora, exp: agora + 300 };
// request object -> <request_object>
```

```
GET /authorize?client_id=<client_id>&request=<request_object>
```

O ponto que costuma escapar: quando há `request`, os parâmetros de dentro do JWT são os
que valem — **eles não se somam ao que veio na query**. Mandar `scope` na URL e outro
`scope` no objeto não amplia nada; o da URL é ignorado.

📖 [RFC 9101](https://datatracker.ietf.org/doc/html/rfc9101)

## RAR — RFC 9396

Autorização granular: em vez de um escopo genérico como `transactions:write`, o token
carrega **a transação específica** que foi autorizada.

```json
{"type": "payment_initiation",
 "account_id": "<account_id>",
 "amount": 150.00,
 "currency": "BRL"}
```

Isso vai como `authorization_details` no `/authorize`, e a tela de consentimento mostra
o pagamento concreto — valor, moeda, conta — em vez de uma permissão abstrata.

**RAR só existe no Authorization Code.** É restrição de projeto, e a razão é direta: um
pagamento precisa de tela de confirmação, e `client_credentials` não tem tela nem
segunda parte. Registrar um cliente com `rar: true` sem `authorization_code` nos
`grant_types` é recusado no ato — a cesta é imutável, e a alternativa seria uma RFC
selecionada e inexercível para sempre.

**Duas falhas, e as duas são o ponto:**

Repetir o token:

```json
{"error":"invalid_token",
 "error_description":"this RAR token has already been used; payment initiation tokens are valid once only"}
```

Executar valor diferente do autorizado:

```json
{"error":"invalid_authorization_details",
 "error_description":"the transaction diverges from what was authorized: amount (authorized 150)"}
```

Inverter o sentido:

```json
{"error":"invalid_authorization_details",
 "error_description":"payment_initiation authorizes an outgoing payment: direction must be debit"}
```

Sem a segunda conferência o `authorization_details` seria enfeite — o cliente pediria
autorização para um valor e executaria outro. Sem a terceira, seria pior: o token
autorizado a **pagar** R$ 150 serviria para **receber** R$ 150, e a tela de
consentimento teria mostrado uma saída que virou entrada.

Compare com o mesmo endpoint **sem** RAR, sob `client_credentials`: ali o pagamento é
repetível, porque o cliente é o próprio dono da conta e não há terceiro a proteger.

📖 [RFC 9396 §2](https://datatracker.ietf.org/doc/html/rfc9396#section-2)

## Formato do token — RFC 9068

Escolha da cesta, e muda o que você recebe.

**`bearer`** — string hexadecimal opaca. Não carrega informação; o Resource Server a
resolve internamente.

**`jwt`** — JWT assinado, que você pode inspecionar:

```json
{"alg":"RS256","typ":"at+jwt","kid":"<kid>"}
```
```json
{"iss":"https://oauth.certauth.dev","aud":"https://resource.certauth.dev",
 "sub":"<pj_id>","client_id":"<client_id>","scope":"accounts:read",
 "jti":"<jti>","iat":1788038834,"exp":1788042434}
```

O `typ` é `at+jwt`, não `JWT` — é a RFC 9068 distinguindo um access token de qualquer
outro JWT, e um validador deveria conferir isso.

Verifique contra o `jwks_uri`. O `kid` é derivado do thumbprint da própria chave
(RFC 7638), e é por isso que os dois workloads do AS publicam um JWKS idêntico: token
emitido por um verifica no outro.

Cole o token em [jwt.io](https://jwt.io) para inspecionar — é bancada, não referência.

📖 [RFC 9068](https://datatracker.ietf.org/doc/html/rfc9068) · [RFC 7638](https://datatracker.ietf.org/doc/html/rfc7638)

---

[← 04 Revogação](04-revogacao.md) · [índice](README.md)
