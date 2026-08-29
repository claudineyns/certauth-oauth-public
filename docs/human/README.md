# Trilha humana — operando o playground

Esta trilha é para quem vai dirigir os fluxos à mão: abrir o browser, montar a cesta,
ver a tela de consentimento e acompanhar o que acontece no fio a cada passo.

Pressupõe familiaridade com OAuth 2.0. Não é um curso sobre o protocolo — é o manual
de operação **deste** playground, que é onde a teoria encontra um servidor de verdade
respondendo.

## Percurso

| | |
|---|---|
| [01 — Primeiros passos](01-primeiros-passos.md) | Montar a cesta, registrar o cliente, entender as três credenciais |
| [02 — Authorization Code](02-authorization-code.md) | A dança de interação, PKCE, e o primeiro saldo consultado |
| [03 — Consentimento cruzado](03-consentimento-cruzado.md) | Um terceiro acessando dados de outra PJ, e o que o escopo limita |
| [04 — Revogação](04-revogacao.md) | Derrubar um token e derrubar um consentimento — são coisas diferentes |
| [05 — Mecanismos de segurança](05-mecanismos.md) | PKCE, DPoP, mTLS, Client Assertion, PAR, JAR, RAR — e o que acontece quando falham |

Se o seu objetivo é integração automatizada, sem ninguém na frente da tela, a trilha
[`machine/`](../machine/) descreve o mesmo sistema como contrato HTTP.

## Como ler os exemplos

**Os trechos de código são ilustrativos.** Mostram a parte que interessa, não um
programa completo: não há imports, tratamento de erro nem `main()`. Copiar e colar não
vai executar, e não é para executar — é para você ver como aquilo se monta na sua
linguagem e reproduzir no seu próprio código.

**Os valores calculados aparecem como marcador.** Onde um trecho produz um
`code_challenge`, uma prova DPoP ou uma assinatura, o resultado é mostrado como
`<code_challenge>`, `<dpop_proof>` e assim por diante.

Isso é deliberado, por dois motivos. Um valor de 43 caracteres em base64url não ensina
nada — o que ensina é a **chamada que o produziu**, e ela está sempre visível. E um
`code_challenge` real num documento convida à cópia, o que quebra o fluxo: ele só vale
com o `code_verifier` que o gerou.

**As trocas HTTP são reais**, capturadas deste sistema. Credenciais nelas aparecem como
`<client_id>`, `<access_token>` e afins; identificadores de PJ, conta e transação
aparecem literais — mas são ilustrativos e **serão diferentes para você**, porque cada
cesta cria as próprias PJs e as contas derivam delas.

## Os cinco endereços

| Host | Papel |
|---|---|
| `oauth-playground.certauth.dev` | A interface e o backend que a serve |
| `oauth.certauth.dev` | Authorization Server |
| `oauth-mtls.certauth.dev` | Authorization Server, TLS mútuo |
| `resource.certauth.dev` | Resource Server |
| `resource-mtls.certauth.dev` | Resource Server, TLS mútuo |

Os certificados são de CA pública, então `curl` funciona sem `-k` e o browser abre sem
aviso.

**Tudo expira.** Nenhum registro do sandbox passa de 24 horas, e a maior parte morre
bem antes. Nada do que você criar aqui sobrevive ao dia seguinte — o que é liberador:
não há como estragar nada.
