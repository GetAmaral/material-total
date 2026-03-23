# WhatsApp Utility Templates — Guia Completo para Mensagens Dinâmicas

> **Última atualização:** 2026-03-23
> **Todas as informações foram verificadas contra fontes oficiais da Meta e BSPs autorizados.**

## Contexto

Ao usar a **WhatsApp Business API (Cloud API)**, mensagens proativas (business-initiated) precisam de **templates aprovados pela Meta**. Templates de **utility** são significativamente mais baratos que marketing e servem para: confirmações, lembretes, atualizações de pedido, alertas de conta, etc.

Este guia foca em como enviar **conteúdo dinâmico de tamanho variável** (ex: lista de lembretes que muda por usuário) dentro de um único template utility.

---

## 1. Regras Fundamentais

### 1.1 Categorias de template

| Categoria | Uso | Custo relativo |
|---|---|---|
| **Utility** | Transações, lembretes, atualizações, alertas | Mais barato (~$0.0068/msg no BR) |
| **Marketing** | Promoções, ofertas, novidades | ~9x mais caro (~$0.0625/msg no BR) |
| **Authentication** | OTPs / códigos de verificação | Similar ao utility |
| **Service** | Respostas a mensagens do usuário | **Grátis (desde Nov/2024)** |

> **Fonte (categorias):** [Meta — Pricing on the WhatsApp Business Platform](https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing)
> *"Marketing, Utility, Authentication, and Service conversations each have different rates."*

> **Fonte (valores BR):** [Rate Card via flowcall.co](https://www.flowcall.co/blog/whatsapp-business-api-pricing-2026) — Marketing: $0.0625, Utility: $0.0068 por mensagem (Brasil).

> **⚠️ IMPORTANTE — Modelo de cobrança mudou em Jul/2025:** Desde 1º de julho de 2025, o WhatsApp migrou de **cobrança por conversa** para **cobrança por mensagem entregue**.
> **Fonte:** [Meta — Updates to Pricing](https://developers.facebook.com/docs/whatsapp/pricing/updates-to-pricing/)
> *"Charges apply when a message is delivered [...] based on who the message is sent to and the category of the message."*

### 1.2 Classificação automática e reclassificação

A Meta classifica o template automaticamente com base no conteúdo. **Desde 9 de abril de 2025**, a Meta pode reclassificar templates **sem aviso prévio**.

> **Fonte:** [Meta — New Template Guidelines](https://developers.facebook.com/docs/whatsapp/updates-to-pricing/new-template-guidelines/)
> *"For any business detected abusing the template categorization system, WhatsApp will no longer provide 24-hour notice if a utility template should be marketing. The category will be updated with no advance notice."*

> **Fonte:** [yCloud — Template Category Guidelines Update](https://www.ycloud.com/blog/whatsapp-api-message-template-category-guidelines-update/)
> Definição atualizada (Jul/2025): para ser utility, o template deve ser **(a)** não-promocional, E **(b)** específico a uma ação/transação do usuário OU essencial/crítico para o usuário.

### 1.3 Limites técnicos do template

| Recurso | Limite | Fonte |
|---|---|---|
| Caracteres no **body** | **1024 chars** (texto fixo + variáveis preenchidas) | Wati.io (BSP oficial), 8x8, Webex Connect |
| Caracteres no **header** (texto) | 60 chars | 8x8 Developer Portal, indigitall docs |
| Caracteres no **footer** | 60 chars | 8x8 Developer Portal, indigitall docs |
| Variáveis no **header** | Máx **1** | 8x8 Developer Portal |
| Variáveis no **body** | Sem limite de contagem (limitado pelos 1024 chars) | 8x8 Developer Portal |
| Variáveis no **footer** | **0** (texto estático) | 8x8 Developer Portal |
| Botões | Máx **10** total | Múltiplas fontes BSP |

> **Fonte (body 1024):** [Wati.io — Template Message Guidelines](https://support.wati.io/en/articles/11463458-whatsapp-template-message-guidelines-naming-formatting-and-translations)
> *"Marketing, Utility and Authentication templates can have up to 1024 characters."*

> **Fonte (header/footer):** [8x8 Developer Portal — Template Components Reference](https://developer.8x8.com/connect/docs/whatsapp/template-components-reference/)
> *"Header Max 60 characters [...] Footer Max Length: 60 characters"*

> **Nota:** A documentação oficial da Meta (developers.facebook.com/docs/whatsapp/*) é renderizada via JavaScript SPA e não pode ser extraída diretamente. Os limites acima são confirmados por múltiplos BSPs oficiais (Wati.io, 8x8, Infobip, Webex Connect) que espelham as restrições da API da Meta.

### 1.4 O que conta no limite de 1024

O limite é do **body renderizado final**, ou seja:

```
texto fixo do template + conteúdo das variáveis preenchidas = máx 1024 chars
```

Exemplo: se o texto fixo tem 120 chars, sobram **~900 chars** para as variáveis.

---

## 2. Estratégia: Lista Dinâmica em Uma Variável

### 2.1 O problema

Templates têm variáveis **fixas** (`{{1}}`, `{{2}}`). Não existe loop ou variável dinâmica. Se um user tem 3 lembretes e outro tem 15, o template é o mesmo.

### 2.2 A solução

Montar toda a lista como **uma única string** antes de enviar, e injetar numa variável só.

### ⚠️ RESTRIÇÃO CRÍTICA: `\n` NÃO funciona em variáveis de template

A API do WhatsApp **rejeita caracteres de quebra de linha (`\n`) e tabulação dentro de parâmetros/variáveis** de template. O erro retornado é:

> *"Param text cannot have new-line/tab characters or more than 4 consecutive spaces"*

> **Fontes:**
> - [n8n Community — WhatsApp API line break](https://community.n8n.io/t/whatsapp-api-line-break/200174)
> - [Manychat Community — Issue with Line Breaks in WhatsApp Templates](https://community.manychat.com/general-q-a-43/issue-with-line-breaks-in-custom-fields-via-whatsapp-templates-4780)
> - [Zapier Community — Line breaks not working](https://community.zapier.com/troubleshooting-99/line-breaks-not-working-in-whatsapp-notifications-causing-error-100-47030)

### 2.3 Workarounds para a restrição de `\n`

**Opção A — Múltiplas variáveis com quebras de linha no texto fixo do template:**

Crie o template com as quebras de linha JÁ NO TEXTO FIXO:

```
Olá {{1}}, seus lembretes para hoje:

{{2}}
{{3}}
{{4}}
{{5}}
{{6}}
{{7}}
{{8}}
{{9}}
{{10}}

Responda se precisar alterar algo.
```

Cada `{{N}}` recebe um item. Se o user tem 3 itens, preencha `{{4}}` a `{{10}}` com string vazia `""`.

**Limitação:** Número máximo de itens é fixo (definido no template). Se definiu 9 slots e o user tem 15, não cabe.

**Opção B — Separador inline (sem quebra de linha):**

Use um separador visual dentro de uma variável só:

```
Olá {{1}}, seus lembretes para hoje: {{2}}

Responda se precisar alterar algo.
```

No Code Node:
```javascript
const lembretes = $input.all();
const texto = lembretes
  .map((item, i) => `${i + 1}. ${item.json.descricao}`)
  .join(' | ');

return [{ json: { listaFormatada: texto } }];
```

Resultado: `1. Reunião às 9h | 2. Enviar relatório | 3. Revisar contrato`

**Limitação:** Menos legível, mas funciona sem restrições.

**Opção C — Emojis como separadores visuais:**

```javascript
const texto = lembretes
  .map((item, i) => `${i + 1}. ${item.json.descricao}`)
  .join(' ✦ ');
```

**Opção D (RECOMENDADA) — Template com variável + convite para resposta:**

Envie um template curto com resumo, e convide o user a responder. Quando ele responde, a **janela de serviço** abre e você pode mandar mensagens free-form **com quebra de linha e sem template**.

Template:
```
Olá {{1}}, você tem {{2}} lembrete(s) para hoje. Responda "ver" para receber a lista completa.
```

Quando o user responder "ver" → janela de serviço abre → envie mensagem free-form com a lista formatada com `\n` normalmente.

> **Esta é a melhor opção** porque combina economia (1 template utility barato) + experiência boa (lista formatada com quebras de linha na resposta grátis).

---

## 3. Falhas Conhecidas e Como Contornar

### 3.1 Estouro de caracteres (>1024)

**Problema:** Usuário com muitos itens ou itens com descrições longas estoura o limite.

**Solução — Truncar com aviso (para Opção B/C):**

```javascript
// Code Node no n8n
const MAX_CHARS = 900;
const lembretes = $input.all();
let texto = '';

for (let i = 0; i < lembretes.length; i++) {
  const item = `${i + 1}. ${lembretes[i].json.descricao}`;
  const separador = i > 0 ? ' | ' : '';

  if ((texto + separador + item).length > MAX_CHARS) {
    const restante = lembretes.length - i;
    texto += ` ... e mais ${restante} item(ns). Responda "ver tudo" para a lista completa.`;
    break;
  }
  texto += separador + item;
}

return [{ json: { listaFormatada: texto.trim() } }];
```

> **Dica:** Ao truncar, convide o usuário a responder. Isso abre a **janela de serviço (24h)** e você pode mandar o restante como mensagem free-form **gratuita** (desde Nov/2024, conversas de serviço são grátis e ilimitadas).

### 3.2 Template classificado como Marketing pela Meta

**Problema:** Você submete como utility mas a Meta reclassifica como marketing (~9x mais caro).

**Como evitar — Regras para aprovação como Utility:**

1. **O texto deve estar ligado a uma ação/transação existente do usuário**
   - Bom: "Seus lembretes para hoje" (o user cadastrou os lembretes)
   - Ruim: "Veja o que preparamos para você" (parece promoção)

2. **Não use linguagem promocional**
   - Evite: "aproveite", "oferta", "exclusivo", "imperdível", "desconto"
   - Use: "lembrete", "atualização", "confirmação", "agendamento"

3. **Não inclua CTA de venda**
   - Evite: "Compre agora", "Saiba mais sobre nossos planos"
   - Pode: "Responda se precisar de ajuda"

4. **Seja direto e informativo**
   - O template deve **informar**, não **persuadir**

> **Fonte:** [Meta — Template Categorization](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/template-categorization)
> Definição oficial de utility: *"Messages typically triggered by a user action [...] non-promotional [...] specific to or requested by a user [...] or critical to a user."*

**Exemplo de template que passa como Utility:**

```
Olá {{1}}, você tem {{2}} lembrete(s) para hoje. Responda "ver" para a lista completa.
```

**Exemplo que será reclassificado como Marketing:**

```
Olá {{1}}! 🎉 Confira as novidades que separamos para você: {{2}} Não perca!
```

### 3.3 Variável vazia / usuário sem itens

**Problema:** Se o user não tem lembretes, enviar o template com variável vazia fica estranho.

**Solução:** No n8n, usar um nó **IF** antes do envio:

```
IF → $json.listaFormatada is not empty
  → Sim: envia template
  → Não: não envia (ou envia template diferente)
```

### 3.4 Formatação no body do template

**O que funciona no texto fixo do template:**

| Formato | Sintaxe |
|---|---|
| **Negrito** | `*texto*` |
| *Itálico* | `_texto_` |
| ~~Tachado~~ | `~texto~` |
| `Monospace` | `` ```texto``` `` |

> **Fonte:** [WhatsApp Help Center — How to format your messages](https://faq.whatsapp.com/539178204879377/?cms_platform=web)

**Restrições importantes:**

- **Header e footer:** não permitem emojis, formatação markdown, nem quebras de linha.
  > **Fonte:** [Infobip — WhatsApp Template Compliance](https://www.infobip.com/docs/whatsapp/compliance/template-compliance)
  > *"Do not include emojis, markup, or newline characters in headers or footers"*

- **Emojis no body:** permitidos em utility e marketing (marketing: máx 10 emojis). Proibidos em authentication.
  > **Fonte:** [Wati.io — Template Guidelines](https://support.wati.io/en/articles/11463458-whatsapp-template-message-guidelines-naming-formatting-and-translations)

---

## 4. Como Criar o Template na Meta

### 4.1 Passo a passo

1. Acesse o **Meta Business Suite** → **WhatsApp Manager**
2. Vá em **Ferramentas da conta** → **Modelos de mensagem**
3. Clique em **Criar modelo**
4. Preencha:
   - **Categoria:** `Utility`
   - **Nome:** `lembrete_diario` (snake_case, sem espaços)
   - **Idioma:** `Português (BR)`
5. No **Body**, escreva (usando a Opção D recomendada):

```
Olá {{1}}, você tem {{2}} lembrete(s) para hoje. Responda "ver" para receber a lista completa.
```

6. Adicione **exemplos** para cada variável (obrigatório):
   - `{{1}}` → `João`
   - `{{2}}` → `3`
7. Submeta para aprovação

> **Tempo de aprovação:** A maioria é aprovada em **minutos a poucas horas** via revisão automatizada. Templates flagados para revisão manual podem levar até **24 horas**.
> **Fonte:** [Infobip — Template Compliance](https://www.infobip.com/docs/whatsapp/compliance/template-compliance)
> *"Approval is usually immediate if your business is verified, but may take up to 24 hours otherwise."*

### 4.2 Dicas para aprovação rápida

- Forneça exemplos realistas e claros
- Mantenha tom informativo, não promocional
- Não use CAPS LOCK no texto fixo
- **Para marketing:** inclua o botão oficial de opt-out da Meta (obrigatório). Para utility: não é necessário opt-out.

> **Fonte (opt-out):** [Meta — Get opt-in for WhatsApp](https://developers.facebook.com/documentation/business-messaging/whatsapp/getting-opt-in) e [WhatsApp Business Policy](https://business.whatsapp.com/policy)

---

## 5. Janela de Conversa e Custos

### 5.1 Regras da janela de serviço

| Regra | Detalhe |
|---|---|
| **Quem abre a janela de serviço** | Somente o **usuário** ao enviar uma mensagem |
| **Duração** | 24 horas (reseta a cada nova mensagem do user) |
| **Dentro da janela** | Mensagens free-form + templates liberados |
| **Fora da janela** | Somente templates aprovados |
| **Custo da conversa de serviço** | **Grátis e ilimitado** (desde Nov/2024) |

> **Fonte:** [Meta — Send Messages](https://developers.facebook.com/documentation/business-messaging/whatsapp/messages/send-messages)
> *"Whenever a WhatsApp user messages you or calls you, a 24-hour timer called a customer service window starts."*
> *"When a customer service window is open [...] you can send any type of message to the user."*
> *"If a window is not open [...] you can only send template messages."*

### 5.2 Utility template dentro da janela de serviço = GRÁTIS

> **Fonte:** [Meta — Updates to Pricing](https://developers.facebook.com/docs/whatsapp/pricing/updates-to-pricing/)
> *"Utility template messages sent in response to a user's message (within an open customer service window) became free."*

### 5.3 Conversas de serviço agora são ilimitadas e grátis

Antes de Nov/2024, havia um limite de 1.000 conversas de serviço grátis por mês. **Isso foi removido.**

> **Fonte:** [Meta — Updates to Pricing](https://developers.facebook.com/docs/whatsapp/pricing/updates-to-pricing/)
> Desde Nov/2024: *"Service conversations are free for all businesses"* — sem limite mensal.

### 5.4 Entry-point gratuito (72h)

Quando o user inicia conversa via **Click-to-WhatsApp Ad** ou **botão CTA da página do Facebook**, a janela gratuita é de **72 horas** (não 24).

> **Fonte:** [WhatsApp Business Platform Pricing](https://business.whatsapp.com/products/platform-pricing)
> *"When customers send you a message from an Ad that clicks to WhatsApp or Facebook Page call-to-action button, for the following 3 days (72 hours), all of your messages are not charged."*

---

## 6. Fluxo Completo no n8n (Opção D — Recomendada)

```
Schedule Trigger (ex: todo dia 8h)
    │
    ▼
Banco/API ── busca users ativos com lembretes
    │
    ▼
SplitInBatches ── 1 user por vez
    │
    ▼
Banco/API ── busca lembretes DO user (+ COUNT)
    │
    ▼
IF ── tem lembretes?
    │         │
    ▼         ▼
  Sim       Não → (ignora)
    │
    ▼
HTTP Request ── POST template: "Você tem X lembretes. Responda 'ver'."
```

**Fluxo secundário (webhook de resposta):**

```
Webhook ── user respondeu "ver"
    │
    ▼
Banco/API ── busca lembretes do user
    │
    ▼
Code Node ── monta lista com \n (FUNCIONA em free-form!)
    │
    ▼
HTTP Request ── POST mensagem free-form (text, não template)
```

### 6.1 HTTP Request — Envio do template (proativo)

**URL:**
```
https://graph.facebook.com/v25.0/SEU_PHONE_NUMBER_ID/messages
```

> **Nota:** A versão atual da Graph API é **v25.0** (Fev 2026). Versões anteriores como v21.0 estão desatualizadas.
> **Fonte:** [Graph API Changelog](https://developers.facebook.com/docs/graph-api/changelog)

**Body (JSON):**

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

> **Fonte (estrutura JSON):** [Meta — Send Message Templates](https://developers.facebook.com/docs/whatsapp/cloud-api/guides/send-message-templates/)
> Parâmetros são **posicionais**: primeiro objeto = `{{1}}`, segundo = `{{2}}`, etc.

### 6.2 HTTP Request — Mensagem free-form (após user responder)

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

> Aqui o `\n` **funciona normalmente** porque é uma mensagem de texto livre, não um parâmetro de template.

---

## 7. Resumo de Limites (Verificado)

| Item | Valor | Fonte |
|---|---|---|
| Body total renderizado | 1024 chars | Wati.io, 8x8, Webex Connect |
| Header (texto) | 60 chars | 8x8, indigitall |
| Footer | 60 chars (sem variáveis) | 8x8, indigitall |
| `\n` em variáveis de template | **NÃO FUNCIONA** | n8n Community, Manychat, Zapier |
| `\n` em mensagem free-form | Funciona normalmente | — |
| Formatação no body | `*negrito*` `_itálico_` `~tachado~` | WhatsApp Help Center |
| Custo utility (BR) | ~$0.0068/msg | Rate card via flowcall.co |
| Custo marketing (BR) | ~$0.0625/msg (~9x mais caro) | Rate card via flowcall.co |
| Conversa de serviço | **Grátis e ilimitada** (desde Nov/2024) | Meta — Updates to Pricing |
| Janela de serviço | 24h (só user abre) | Meta — Send Messages |
| Entry-point (ads) | 72h grátis | WhatsApp Business Pricing |
| Aprovação de template | Minutos a 24h | Infobip |
| API version atual | **v25.0** (Fev 2026) | Graph API Changelog |

---

## 8. Fontes Completas

| # | Fonte | URL |
|---|---|---|
| 1 | Meta — Pricing | https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing |
| 2 | Meta — Updates to Pricing | https://developers.facebook.com/docs/whatsapp/pricing/updates-to-pricing/ |
| 3 | Meta — Template Categorization | https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/template-categorization |
| 4 | Meta — Send Messages | https://developers.facebook.com/documentation/business-messaging/whatsapp/messages/send-messages |
| 5 | Meta — Send Message Templates | https://developers.facebook.com/docs/whatsapp/cloud-api/guides/send-message-templates/ |
| 6 | Meta — New Template Guidelines | https://developers.facebook.com/docs/whatsapp/updates-to-pricing/new-template-guidelines/ |
| 7 | Meta — Get Opt-in | https://developers.facebook.com/documentation/business-messaging/whatsapp/getting-opt-in |
| 8 | WhatsApp Business Pricing | https://business.whatsapp.com/products/platform-pricing |
| 9 | WhatsApp Business Policy | https://business.whatsapp.com/policy |
| 10 | WhatsApp Help Center — Formatting | https://faq.whatsapp.com/539178204879377/?cms_platform=web |
| 11 | Graph API Changelog | https://developers.facebook.com/docs/graph-api/changelog |
| 12 | 8x8 — Template Components Reference | https://developer.8x8.com/connect/docs/whatsapp/template-components-reference/ |
| 13 | Wati.io — Template Guidelines | https://support.wati.io/en/articles/11463458-whatsapp-template-message-guidelines-naming-formatting-and-translations |
| 14 | Infobip — Template Compliance | https://www.infobip.com/docs/whatsapp/compliance/template-compliance |
| 15 | n8n Community — WhatsApp line break | https://community.n8n.io/t/whatsapp-api-line-break/200174 |
| 16 | Manychat — Line break issue | https://community.manychat.com/general-q-a-43/issue-with-line-breaks-in-custom-fields-via-whatsapp-templates-4780 |
| 17 | Zapier — Line breaks not working | https://community.zapier.com/troubleshooting-99/line-breaks-not-working-in-whatsapp-notifications-causing-error-100-47030 |
| 18 | Rate Card BR (flowcall.co) | https://www.flowcall.co/blog/whatsapp-business-api-pricing-2026 |
| 19 | yCloud — Category Guidelines Update | https://www.ycloud.com/blog/whatsapp-api-message-template-category-guidelines-update/ |
