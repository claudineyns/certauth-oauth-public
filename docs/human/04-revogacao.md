# 04 — Revogação

[← 03 Consentimento cruzado](03-consentimento-cruzado.md) · [índice](README.md) · a seguir: [05 — Mecanismos](05-mecanismos.md)

Terceiro cenário. E o ponto do capítulo é que **existem duas revogações diferentes**,
com efeitos diferentes, e confundi-las produz um daqueles bugs que ninguém acha.

| | Quem faz | O que morre |
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

Duas armadilhas da RFC 7009 que valem conhecer:

**O `200` não significa que o token existia.** Revogar token inexistente ou já revogado
também devolve `200`. É proposital — a resposta não pode virar um oráculo que confirma
quais tokens são válidos. Nunca use o código de status para deduzir existência.

**O refresh token sobrevive.** Revogar um access token não derruba o refresh que o
acompanhava. Se a intenção é encerrar o acesso, revogue os dois — ou revogue o
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
{"revogado":"<client_id>.677410B4965245","efeito":"imediato"}
```

A próxima chamada com o **mesmo token que ainda não expirou**:

```http
HTTP/1.1 403 Forbidden
```
```json
{"error":"consent_required",
 "error_description":"no active consent from this legal entity for this client; it may have been revoked"}
```

## O detalhe que revela o desenho

Depois de revogar o consentimento, pergunte ao Authorization Server o que ele acha
daquele token:

```
active: True | scope: accounts:read
```

**O token continua ativo.** Não foi revogado, não expirou, e o AS o emitiria de novo
sem hesitar. Quem recusa é o Resource Server, e por outro motivo: a autorização que
justificava aquele acesso não existe mais.

Isso não é inconsistência — é a separação de responsabilidades ficando visível. O AS
responde *"este token é autêntico?"*. O RS responde *"esta autorização ainda vale?"*.
São perguntas diferentes, e só a segunda depende do consentimento.

A consequência prática é a que interessa:

- `invalid_token` (401) → **pegue um token novo**, é o caminho certo.
- `consent_required` (403) → **não adianta**. O AS emitirá outro token com prazer, e o
  RS o recusará exatamente igual. Só um novo consentimento — pela tela, com alguém
  presente — restaura o acesso.

Um cliente que trate os dois como a mesma coisa entra em laço: pede token, é recusado,
pede outro, é recusado, para sempre.

## O que mais derruba um acesso

Revogação não é o único caminho. Vale ter o mapa:

| Evento | Efeito |
|---|---|
| Revogar o access token | Só ele |
| Revogar o consentimento | Todo acesso daquele par cliente/PJ, presente e futuro |
| Reemitir o certificado de cliente | **Todos** os tokens atrelados ao certificado anterior |
| Reemitir o par de chaves do cliente | Toda assertion assinada com a chave anterior |
| `DELETE /register/{client_id}` | O cliente inteiro, em cascata |
| Passar 24 horas | Tudo |

As duas reemissões estão em [05 — Mecanismos](05-mecanismos.md); são as que mais
surpreendem, porque a pessoa clica em "emitir novo" sem imaginar que está derrubando o
que já estava em uso.

---

[← 03 Consentimento cruzado](03-consentimento-cruzado.md) · [índice](README.md) · a seguir: [05 — Mecanismos](05-mecanismos.md)
