# WhatsApp Utility Templates — Guia Completo para Mensagens Dinâmicas

> **Ultima atualizacao:** 2026-03-23
> **Criterio de fontes:** Somente documentacao oficial da Meta/WhatsApp. Informacoes nao verificaveis estao marcadas com [NAO VERIFICADO OFICIALMENTE].

---

## 1. Regras Fundamentais

### 1.1 Categorias de mensagem

Extraido de **business.whatsapp.com/products/platform-pricing**:

| Categoria | Definicao oficial |
|---|---|
| **Marketing** | *"Marketing messages can help businesses achieve a wide range of goals across the funnel, from generating awareness to driving sales to retargeting customers."* |
| **Utility** | *"Utility messages are typically triggered by a user action, like placing an order or submitting a payment."* |
| **Authentication** | *"Authentication messages enable businesses to use one-time passwords (OTPs) for identity verification."* |
| **Service** | *"When users message a business, this opens a 24-hour customer service window during which businesses can respond with service messages, at no charge."* |

### 1.2 Modelo de cobranca

Extraido de **business.whatsapp.com/products/platform-pricing**:

> *"Businesses using our platform are charged on a per-message basis for each message we deliver to users."*
> *"We charge when a message is delivered (not sent)."*
> *"We charge based on who the message is sent to and the category of the message."*

**Service e utility em resposta sao gratis:**
> *"We do not charge for service messages, or for utility messages businesses send in response to users."*

**[NAO VERIFICADO OFICIALMENTE]** Valores exatos por pais (o rate card em business.whatsapp.com e renderizado via JavaScript e nao pode ser extraido estaticamente). Para consultar valores atualizados, acesse: https://business.whatsapp.com/products/platform-pricing e selecione o pais manualmente.

### 1.3 Estimativa de custo mensal (Brasil)

**[NAO VERIFICADO OFICIALMENTE]** Os valores abaixo vem de BSPs (Business Solution Providers) que espelham o rate card da Meta. Duas faixas foram encontradas:

| Categoria | Jul/2025 (USD) | Jan/2026 (USD) |
|---|---|---|
| Marketing | $0.0625/msg | $0.07188/msg |
| Utility | $0.0068/msg | $0.00782/msg |
| Authentication | $0.0068/msg | $0.00782/msg |
| Service | Gratis | Gratis |
| Utility em resposta ao user | Gratis | Gratis |

> Fontes: FlowCall, EngageLab (Jul/2025); SleekFlow (Jan/2026). Todos citam o rate card oficial da Meta.

**Simulacao: 1 mensagem proativa por dia, 30 dias (usando valores Jan/2026):**

| Cenario | Por user/mes | 100 users/mes | 500 users/mes | 1.000 users/mes |
|---|---|---|---|---|
| **Utility proativo** | $0.23 | $23.46 | $117.30 | $234.60 |
| **Marketing proativo** | $2.16 | $215.64 | $1.078,20 | $2.156,40 |
| **Utility em resposta** | **Gratis** | **Gratis** | **Gratis** | **Gratis** |

> Calculo: 30 msgs x $0.00782 = $0.2346/user/mes (utility) | 30 msgs x $0.07188 = $2.1564/user/mes (marketing)

**Comparacao direta:**
- Marketing custa **~9.2x mais** que utility
- Utility em resposta ao user (dentro da janela de servico) e **gratis**
- Usando a Opcao C recomendada (template curto → user responde → free-form gratis), o custo e de **1 utility por dia por user = ~$0.23/user/mes**
- Se o user responder ao template, o restante da interacao (lista completa) sai **gratis**

**Nota sobre volume tiers:** A Meta oferece desconto por volume (*"As businesses send more messages on WhatsApp, they can unlock more attractive pricing via our volume tiers."* — business.whatsapp.com). Os thresholds por pais nao estao publicamente documentados.

### 1.3 Limites tecnicos do template

Extraido de **developers.facebook.com/docs/whatsapp/business-management-api/message-templates/components** (via cache):

| Recurso | Limite | Trecho oficial |
|---|---|---|
| **Body** | 1024 chars | *"1024 character maximum."* |
| **Header (texto)** | 60 chars | *"60 character maximum."* |
| **Footer** | 60 chars | *"60 characters maximum."* |
| **Variaveis no header** | Max 1 | *"Can support 1 parameter."* |
| **Variaveis no body** | Multiplas | *"Can support multiple parameters."* |
| **Variaveis no footer** | 0 | Texto estatico, sem variaveis |
| **Botoes** | Max 10 total | *"Templates can have a mixture of up to 10 button components total"* |
| **Label do botao** | 25 chars | *"25 characters maximum."* |
| **Nome do template** | 512 chars | *"Maximum 512 characters."* |

Extraido de **developers.facebook.com/docs/whatsapp/business-management-api/message-templates** (via cache):

> *"The message template content field is limited to 1024 characters."*

### 1.4 Regras de variaveis

Extraido de **developers.facebook.com/.../components** e **developers.facebook.com/.../guidelines** (via cache):

- Variaveis devem ser **sequenciais**: definir `{{1}}`, `{{2}}`, `{{4}}` sem `{{3}}` sera rejeitado.
- Template **nao pode iniciar ou terminar com uma variavel**: *"dangling parameters are not allowed"* (error code 2388299).
- **Excesso de variaveis** em relacao ao texto: *"Template contains too many variable parameters relative to the message length. You need to decrease the number of variable parameters or increase the message length."* (error code 2388293).
- Variaveis nomeadas: *"Must be lowercase letters and underscores only."*

### 1.5 Newlines na definicao do template

Extraido de **developers.facebook.com/.../creation** (via cache):

> *"Make sure it contains no newlines, tabs, or more than 4 consecutive spaces and meets the length restrictions"*

**Nota importante:** Este trecho refere-se a **definicao do template** (criacao via API). Sobre `\n` em **valores de parametros no envio**, a documentacao oficial e **silenciosa** — nao confirma nem nega explicitamente.

**[NAO VERIFICADO OFICIALMENTE]** Comunidades (n8n, Zapier, Manychat) reportam que a API rejeita `\n` em parametros com o erro: *"Param text cannot have new-line/tab characters or more than 4 consecutive spaces"*. Porem isso nao esta na documentacao oficial.

---

## 2. Janela de Conversa e Custos

### 2.1 Janela de servico (customer service window)

Extraido de **business.whatsapp.com/products/platform-pricing**:

> *"When users message a business, this opens a 24-hour customer service window during which businesses can respond with service messages, at no charge."*
> *"This window resets with each user message."*

**Resumo confirmado oficialmente:**
- Quem abre: somente o **usuario** ao enviar mensagem
- Duracao: **24 horas**
- Reseta a cada nova mensagem do usuario
- Custo: **gratis**

### 2.2 Service e utility em resposta = gratis

Extraido de **business.whatsapp.com/products/platform-pricing**:

> *"We do not charge for service messages, or for utility messages businesses send in response to users."*

Isso confirma: se o usuario mandou mensagem (janela aberta), utility templates enviados em resposta **nao sao cobrados**.

### 2.3 Entry-point gratuito (72h)

Extraido de **business.whatsapp.com/products/platform-pricing**:

> *"When customers send you a message from an Ad that clicks to WhatsApp or Facebook Page call-to-action button, for the following 3 days (72 hours), all of your messages are not charged."*

### 2.4 O que NAO esta confirmado oficialmente sobre janelas

**[NAO VERIFICADO OFICIALMENTE]:**
- Se enviar um template proativo permite ou nao enviar mensagens free-form na mesma janela. A doc oficial so menciona "customer service window" aberta pelo usuario. As paginas de `conversation-types` e `send-messages` sao JS-rendered e nao puderam ser extraidas.

---

## 3. Estrategia: Lista Dinamica em Template Utility

### 3.1 O problema

Templates tem variaveis fixas (`{{1}}`, `{{2}}`). Nao existe loop. Se um user tem 3 lembretes e outro tem 15, o template e o mesmo.

### 3.2 O que sabemos oficialmente

- Body maximo: **1024 chars** (texto fixo + variaveis preenchidas)
- Multiplas variaveis no body: **permitido**
- Variaveis devem ser sequenciais e nao podem iniciar/terminar o template
- Excesso de variaveis relativo ao texto = rejeicao

### 3.3 Restricao de `\n` em parametros

**Status: nao documentado oficialmente, mas amplamente reportado por usuarios.**

Comunidades reportam que valores de parametros com `\n` geram erro. Ate confirmacao oficial, trate como **restricao real** e planeje workarounds.

### 3.4 Workarounds

**Opcao A — Multiplas variaveis com quebras no texto fixo:**

Crie o template com quebras de linha JA NO TEXTO FIXO:

```
Ola {{1}}, seus lembretes para hoje:

{{2}}
{{3}}
{{4}}
{{5}}
{{6}}

Responda se precisar alterar algo.
```

Cada `{{N}}` recebe um item. Se o user tem 3 itens, preencha `{{4}}` a `{{6}}` com string vazia.

**Limitacao:** Numero maximo de itens e fixo (definido no template). Lembre que excesso de variaveis relativo ao texto pode causar rejeicao (error 2388293).

**Opcao B — Separador inline em uma variavel so:**

```
Ola {{1}}, seus lembretes: {{2}}
```

No Code Node do n8n:
```javascript
const lembretes = $input.all();
const texto = lembretes
  .map((item, i) => `${i + 1}. ${item.json.descricao}`)
  .join(' | ');

return [{ json: { listaFormatada: texto } }];
```

Resultado: `1. Reuniao 9h | 2. Enviar relatorio | 3. Revisar contrato`

**Opcao C (RECOMENDADA) — Template curto + convite para resposta:**

Template utility:
```
Ola {{1}}, voce tem {{2}} lembrete(s) para hoje. Responda "ver" para a lista completa.
```

Quando o user responder "ver":
- Abre janela de servico (24h, gratis — confirmado oficialmente)
- Voce envia mensagem free-form com a lista formatada
- Na mensagem free-form, `\n` funciona normalmente (nao e parametro de template)
- O utility dentro da janela tambem e gratis (*"or for utility messages businesses send in response to users"*)

### 3.5 Truncamento (para Opcao B)

```javascript
// Code Node no n8n
const MAX_CHARS = 900; // margem segura (body max = 1024)
const lembretes = $input.all();
let texto = '';

for (let i = 0; i < lembretes.length; i++) {
  const item = `${i + 1}. ${lembretes[i].json.descricao}`;
  const separador = i > 0 ? ' | ' : '';

  if ((texto + separador + item).length > MAX_CHARS) {
    const restante = lembretes.length - i;
    texto += ` ... e mais ${restante} item(ns). Responda "ver tudo".`;
    break;
  }
  texto += separador + item;
}

return [{ json: { listaFormatada: texto.trim() } }];
```

---

## 4. Dicas para Template Utility (nao Marketing)

Baseado na definicao oficial de utility:
> *"Utility messages are typically triggered by a user action, like placing an order or submitting a payment."*

**Para ser classificado como utility:**
- Deve estar ligado a uma acao/transacao do usuario (ele cadastrou os lembretes)
- Nao pode ser promocional
- Deve informar, nao persuadir

**[NAO VERIFICADO OFICIALMENTE]:** Regras detalhadas de reclassificacao automatica. As paginas `new-template-guidelines` e `template-categorization` sao JS-rendered e nao puderam ser extraidas.

---

## 5. Fluxo no n8n (Opcao C)

### 5.1 Fluxo proativo (diario)

```
Schedule Trigger (todo dia 8h)
    |
    v
Banco/API -- busca users ativos com lembretes (+ COUNT)
    |
    v
SplitInBatches -- 1 user por vez
    |
    v
IF -- tem lembretes?
    |         |
    v         v
  Sim       Nao -> (ignora)
    |
    v
HTTP Request -- POST template utility
```

### 5.2 Fluxo de resposta (webhook)

```
Webhook -- user respondeu "ver"
    |
    v
Banco/API -- busca lembretes do user
    |
    v
Code Node -- monta lista com \n (funciona em free-form)
    |
    v
HTTP Request -- POST mensagem free-form (type: text)
```

### 5.3 HTTP Request — Template utility

**URL:**
```
https://graph.facebook.com/v25.0/SEU_PHONE_NUMBER_ID/messages
```

> **Versao da API:** v25.0 (18 Fev 2026).
> **Fonte:** developers.facebook.com/docs/graph-api/changelog

**Body:**

```json
{
  "messaging_product": "whatsapp",
  "to": "{{$json.telefone}}",
  "type": "template",
  "template": {
    "name": "lembrete_diario",
    "language": {
      "code": "pt_BR"
    },
    "components": [
      {
        "type": "body",
        "parameters": [
          {
            "type": "text",
            "text": "{{$json.nome}}"
          },
          {
            "type": "text",
            "text": "{{$json.totalLembretes}}"
          }
        ]
      }
    ]
  }
}
```

### 5.4 HTTP Request — Mensagem free-form (apos user responder)

```json
{
  "messaging_product": "whatsapp",
  "to": "{{$json.telefone}}",
  "type": "text",
  "text": {
    "body": "{{$json.listaFormatada}}"
  }
}
```

Aqui `\n` funciona normalmente (nao e parametro de template).

---

## 6. Resumo de Limites

| Item | Valor | Status da fonte |
|---|---|---|
| Body total | 1024 chars | OFICIAL — *"1024 character maximum."* |
| Header (texto) | 60 chars, max 1 variavel | OFICIAL |
| Footer | 60 chars, sem variaveis | OFICIAL |
| Botoes | Max 10 total, 25 chars cada | OFICIAL |
| `\n` em parametros de template | Provavelmente nao funciona | NAO VERIFICADO OFICIALMENTE |
| `\n` em mensagem free-form | Funciona | Comportamento padrao da API |
| Service messages | Gratis | OFICIAL — *"We do not charge for service messages"* |
| Utility em resposta ao user | Gratis | OFICIAL — *"or for utility messages businesses send in response to users"* |
| Janela de servico | 24h, aberta pelo user | OFICIAL |
| Entry-point (ads) | 72h gratis | OFICIAL |
| API version atual | v25.0 (Fev 2026) | OFICIAL |
| Precos por pais | Nao extraivel (JS-rendered) | CONSULTAR MANUALMENTE |
| Formatacao (*bold*, _italic_) | Nao extraivel (JS-rendered) | CONSULTAR MANUALMENTE |

---

## 7. Fontes Oficiais Utilizadas

| # | Fonte | URL | Metodo de acesso |
|---|---|---|---|
| 1 | Template Components | developers.facebook.com/docs/whatsapp/business-management-api/message-templates/components | Cache/Wayback |
| 2 | Message Templates | developers.facebook.com/docs/whatsapp/business-management-api/message-templates | Cache/Wayback |
| 3 | Template Guidelines | developers.facebook.com/docs/whatsapp/message-templates/guidelines | Cache/Wayback |
| 4 | Template Creation | developers.facebook.com/docs/whatsapp/message-templates/creation | Cache/Wayback |
| 5 | Platform Pricing | business.whatsapp.com/products/platform-pricing | Fetch direto |
| 6 | Graph API Changelog | developers.facebook.com/docs/graph-api/changelog | Fetch direto |

**Paginas que NAO puderam ser acessadas** (JS-rendered SPA, sem conteudo no HTML):
- developers.facebook.com/docs/whatsapp/pricing
- developers.facebook.com/docs/whatsapp/pricing/updates-to-pricing
- developers.facebook.com/docs/whatsapp/conversation-types
- developers.facebook.com/docs/whatsapp/cloud-api/guides/send-messages
- developers.facebook.com/docs/whatsapp/updates-to-pricing/new-template-guidelines
- developers.facebook.com/docs/whatsapp/message-templates/template-categorization
- faq.whatsapp.com/539178204879377 (formatting rules)

> **Recomendacao:** Para verificar informacoes marcadas como [NAO VERIFICADO OFICIALMENTE], acesse as URLs acima em um navegador real e confira o conteudo renderizado.
