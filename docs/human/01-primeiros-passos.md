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
  "rfc_config": { "modo_autorizacao": "query", "proof": "nenhum",
                  "formato_token": "bearer", "pkce": true,
                  "rar": false, "client_assertion": false },
  "pjs": [
    {"id": "B961D002E8DD28", "identificador": "B9.61D.002/E8DD-28", "papel": "titular"},
    {"id": "9C8FB561112A07", "identificador": "9C.8FB.561/112A-07", "papel": "correntista"}
  ]
}
```

O **IRT** (*initial registration token*) é o que autoriza o registro do cliente. Vale
24 horas e **uma vez só**.

As duas **PJs** nascem juntas e ligadas uma à outra. A `titular` é o terceiro — quem
quer acessar; a `correntista` é a dona da conta — quem autoriza. Essa dupla é o palco
de todos os cenários adiante.

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

## As três credenciais

A resposta do registro devolve três segredos, e confundi-los é o tropeço mais comum:

| Credencial | Serve para | Prazo |
|---|---|---|
| `client_id` | Identificar o cliente. Não é segredo. | 24h |
| `client_secret` | Autenticar em `/token` e `/revoke` — **só se a cesta permitir** | 24h, **nunca regenerado** |
| `registration_access_token` | Manter o registro (RFC 7592) e emitir material criptográfico | 24h |

O `registration_access_token` — o **token de manutenção** — não é token de acesso. Ele
não compra nada no Resource Server: administra o cadastro e é a credencial da tela de
manutenção.

**O `client_secret` nunca é regenerado.** Não há endpoint de rotação. Perdeu, registra
outro cliente.

E atenção ao terceiro caso: se a cesta escolheu `mtls` ou `client_assertion`, o
`client_secret` vem na resposta mas **não é aceito** em lugar nenhum. A cesta manda.

## Administrar o registro

A segunda ação da tela principal pede o token de manutenção e abre a edição. Pela API,
é a RFC 7592 — três campos graváveis e nada mais: `client_name`, `redirect_uris`,
`grant_types`.

Tentar alterar a cesta tem resposta própria, e não silêncio:

```json
{"error":"rfc_config_immutable",
 "error_description":"A cesta de RFC e imutavel. Para usar outra combinacao, registre um novo client."}
```

É também na tela de manutenção que se emite o material que a cesta exigir — certificado
de cliente para mTLS, par de chaves para JAR ou Client Assertion. Ambos reemissíveis,
com uma consequência que vale conhecer antes de clicar; está em
[05 — Mecanismos](05-mecanismos.md).

Apagar o registro remove tudo em cascata: o cliente, a cesta, o thumbprint do
certificado, a chave pública, os consentimentos e o próprio token de manutenção. As
duas PJs sobrevivem, no prazo delas.

---

[← índice](README.md) · a seguir: [02 — Authorization Code](02-authorization-code.md)
