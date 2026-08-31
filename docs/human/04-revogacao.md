# 04 — Revogação

[← 03 Consentimento cruzado](03-consentimento-cruzado.md) · [índice](README.md) · a seguir: [05 — Mecanismos](05-mecanismos.md)

Terceiro cenário. **Existem duas revogações diferentes**, com efeitos diferentes, e
confundi-las produz falhas difíceis de diagnosticar.

| | Quem faz | O que deixa de valer |
|---|---|---|
| Revogar o **token** (RFC 7009) | O cliente | Aquele token, e só ele |
| Revogar o **consentimento** | Qualquer das duas partes | A autorização inteira — todo token presente e futuro daquele par |

## Revogar um token

```http
POST /revoke HTTP/1.1
Host: oauth.certauth.dev
Authorization: Basic <base64(client_id:client_secret)>
Content-Type: application/x-www-form-urlencoded

token=<access_token>
```
```http
HTTP/1.1 200 OK
```

O efeito é imediato:

```json
{"error":"invalid_token","error_description":"token inactive, expired or revoked"}
```

### Pontos de atenção

**O `200` não significa que o token existia.** Revogar token inexistente ou já
revogado também devolve `200`. É deliberado: a resposta não pode servir de oráculo
sobre quais tokens são válidos. O código de status não permite deduzir existência.

**O refresh token continua válido.** Revogar um access token não invalida o refresh
que o acompanhava. Para encerrar o acesso, revogue os dois — ou revogue o
consentimento, adiante.

📖 [RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009)

## Revogar um consentimento

Qualquer uma das partes pode: o cliente que recebeu a autorização, ou a PJ que a
concedeu. O que ninguém pode é revogar consentimento alheio — tentar devolve
`access_denied`.

```http
POST /api/consents/<client_id>.677410B4965245/revoke HTTP/1.1
Host: resource.certauth.dev
Authorization: Bearer <access_token>
```

```json
{"revoked":"<client_id>.677410B4965245","effect":"immediate"}
```

A próxima chamada com o **mesmo token que ainda não expirou**:

```http
HTTP/1.1 403 Forbidden
```
```json
{"error":"consent_required",
 "error_description":"no active consent from this legal entity for this client; it may have been revoked"}
```

## Separação de responsabilidades entre AS e RS

Depois de revogar o consentimento, consulte o Authorization Server sobre aquele
token:

```
active: True | scope: accounts:read
```

**O token continua ativo.** Não foi revogado nem expirou, e o AS emitiria outro
igual. Quem recusa é o Resource Server, por outro motivo: a autorização que
justificava aquele acesso não existe mais.

Não é inconsistência, e sim a separação de responsabilidades ficando visível. O AS
responde *"este token é autêntico?"*; o RS responde *"esta autorização ainda vale?"*.
São perguntas diferentes, e só a segunda depende do consentimento.

A consequência prática:

- `invalid_token` (401) → **obter um token novo** resolve.
- `consent_required` (403) → **obter outro token não resolve**. O AS emitirá outro, e
  o RS o recusará igual. Só um novo consentimento, concedido na tela, restaura o
  acesso.

Um cliente que trate os dois casos como equivalentes entra em laço: pede token, é
recusado, pede outro, é recusado.

## O que invalida um acesso

Revogação não é o único caminho:

| Evento | Efeito |
|---|---|
| Revogar o access token | Só ele |
| Revogar o consentimento | Todo acesso daquele par cliente/PJ, presente e futuro |
| Reemitir o certificado de cliente | **Todos** os tokens atrelados ao certificado anterior |
| Reemitir o par de chaves do cliente | Toda assertion assinada com a chave anterior |
| `DELETE /register/{client_id}` | O cliente inteiro, em cascata |
| Passar 24 horas | Tudo |

As duas reemissões estão em [05 — Mecanismos](05-mecanismos.md). Em ambas, o material
anterior é invalidado, e com ele os tokens que dependiam dele.

---

[← 03 Consentimento cruzado](03-consentimento-cruzado.md) · [índice](README.md) · a seguir: [05 — Mecanismos](05-mecanismos.md)
