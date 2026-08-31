# 01 — Primeiros passos

[← índice](README.md) · a seguir: [02 — Authorization Code](02-authorization-code.md)

## A cesta de RFCs

Tudo começa em `https://oauth-playground.certauth.dev`, na ação **Configurar Cenário
OAuth**. Você monta ali a combinação de RFCs que o seu cliente vai exercitar.

Este é o passo que define o playground. A cesta tem três grupos **mutuamente
exclusivos** — uma escolha em cada — e três opções **aditivas**:

| Grupo | Opções | O que muda |
|---|---|---|
| Modo de autorização | `query` · `par` · `jar` | Como os parâmetros chegam ao `/authorize` |
| Prova de posse | `nenhum` · `dpop` · `mtls` | Se o token fica atrelado a uma chave ou certificado |
| Formato do token | `bearer` · `jwt` | Opaco, ou JWT assinado (RFC 9068) |

| Aditivo | Efeito |
|---|---|
| `pkce` | Exige o par verifier/challenge no fluxo de código |
| `rar` | Habilita `authorization_details`; **exige** `authorization_code` |
| `client_assertion` | Substitui o `client_secret` por JWT assinado (RFC 7523) |

**A cesta é imutável.** Depois de fechada, nenhum caminho a altera — nem a UI, nem a
API RFC 7592. Para experimentar outra combinação, monte outra cesta. Isso é deliberado:
um cliente que muda de mecanismo no meio deixa de demonstrar qualquer coisa.

Ao salvar, três coisas nascem:

```json
{
  "initial_registration_token": "<initial_registration_token>",
  "expires_in": 86400,
  "rfc_config": { "authorization_mode": "query", "proof": "none",
                  "token_format": "bearer", "pkce": true,
                  "rar": false, "client_assertion": false },
  "legal_entities": [
    {"id": "A9701183A1D854", "identifier": "A9.701.183/A1D8-54", "role": "client_owner"},
    {"id": "ED01D5E47F3A78", "identifier": "ED.01D.5E4/7F3A-78", "role": "third_party"}
  ]
}
```

O **IRT** (*initial registration token*) é o que autoriza o registro do cliente. Vale
24 horas e **uma vez só**.

As duas **PJs** nascem juntas e ligadas uma à outra. A `client_owner` é a dona do
cliente que você acabou de registrar; a `third_party` não tem vínculo com ele. Ambas
têm contas.

A regra prática, e a única que importa guardar: **`client_credentials` age sempre
como a `client_owner`**, e não alcança a outra. Chegar à `third_party` exige o
Authorization Code, com login e consentimento na tela.

## Registrar o cliente

Duas portas, mesmo resultado. A UI oferece o registro logo depois do wizard; ou você
faz por API, exercitando a RFC 7591:

```http
POST /register HTTP/1.1
Host: oauth.certauth.dev
Authorization: Bearer <initial_registration_token>
Content-Type: application/json

{
  "client_name": "Painel do Contador",
  "redirect_uris": ["https://cliente.exemplo/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "accounts:read transactions:read"
}
```

```python
# ilustrativo
corpo = {
    "client_name": "Painel do Contador",
    "redirect_uris": ["https://cliente.exemplo/callback"],
    "grant_types": ["authorization_code", "refresh_token"],
}
req = urllib.request.Request(
    "https://oauth.certauth.dev/register",
    data=json.dumps(corpo).encode(),
    headers={"Authorization": f"Bearer {irt}",
             "Content-Type": "application/json"})
registro = json.load(urllib.request.urlopen(req))     # -> <registro>
```

```js
// ilustrativo
const registro = await fetch('https://oauth.certauth.dev/register', {
  method: 'POST',
  headers: { Authorization: `Bearer ${irt}`,
             'Content-Type': 'application/json' },
  body: JSON.stringify({
    client_name: 'Painel do Contador',
    redirect_uris: ['https://cliente.exemplo/callback'],
    grant_types: ['authorization_code', 'refresh_token'],
  }),
}).then((r) => r.json());                             // -> <registro>
```

Os dois caminhos produzem registros **idênticos** — mesmos campos, mesmo esquema de
hash do segredo, mesmos prazos. Escolher pela UI ou pela API é conveniência, não
diferença de comportamento.

> `redirect_uris` é obrigatório mesmo para um cliente que nunca vai redirecionar. É
> metadado da RFC 7591, não declaração de intenção.

Cada URI precisa usar **`https`** — ou `http` em loopback (`localhost`, `127.0.0.1`,
`::1`), que é o que a RFC 8252 reserva para desenvolvimento de app nativo. Qualquer
outro esquema é recusado, tanto no registro quanto na edição. Fragmento também.

## As três credenciais

A resposta do registro devolve três segredos, com finalidades distintas:

| Credencial | Serve para | Prazo |
|---|---|---|
| `client_id` | Identificar o cliente. Não é segredo. | 24h |
| `client_secret` | Autenticar em `/token` e `/revoke` — **só se a cesta permitir** | 24h, **nunca regenerado** |
| `registration_access_token` | Manter o registro (RFC 7592) e emitir material criptográfico | 24h |

O `registration_access_token` — o **token de manutenção** — não é token de acesso. Ele
não compra nada no Resource Server: administra o cadastro e é a credencial da tela de
manutenção.

**O `client_secret` nunca é regenerado.** Não há endpoint de rotação; perdido o
segredo, o caminho é registrar outro cliente.

> **Ponto de atenção.** Se a cesta escolheu `mtls` ou `client_assertion`, o
> `client_secret` é devolvido no registro mas **não é aceito** em nenhum endpoint. A
> autenticação passa a ser a que a cesta define.

## Administrar o registro

A segunda ação da tela principal pede o token de manutenção e abre a edição. Pela API,
é a RFC 7592 — três campos graváveis e nada mais: `client_name`, `redirect_uris`,
`grant_types`.

Tentar alterar a cesta é recusado explicitamente:

```json
{"error":"rfc_config_immutable",
 "error_description":"The RFC basket is immutable. To use another combination, register a new client."}
```

É também na tela de manutenção que se emite o material que a cesta exigir — certificado
de cliente para mTLS, par de chaves para JAR ou Client Assertion. Ambos são
reemissíveis, e a reemissão invalida o material anterior — o efeito está descrito em
[05 — Mecanismos](05-mecanismos.md).

Apagar o registro remove em cascata o cliente, a cesta, o thumbprint do certificado, a
chave pública, os consentimentos e o próprio token de manutenção. As duas PJs
permanecem até o fim do prazo delas.

---

[← índice](README.md) · a seguir: [02 — Authorization Code](02-authorization-code.md)
