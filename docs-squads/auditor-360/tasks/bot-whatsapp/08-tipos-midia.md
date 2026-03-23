# Metodologia de Auditoria — Tipos de Mídia e Mensagens Especiais

**Funcionalidade:** `bot-whatsapp/08-tipos-midia`
**Versão:** 1.1.0

---

## 1. Mapa do sistema

### O que é

O WhatsApp permite enviar diversos tipos de mensagem além de texto. O roteador (Main workflow) tem um Switch que verifica o tipo de mídia (`texto`, `audio`, `image/document`). Existem tipos que NÃO estão no Switch e cujo comportamento é desconhecido.

### Testabilidade por tipo

| Tipo | Testável via webhook DEV? | Por que |
|------|--------------------------|---------|
| **Forwarded text** | ✅ SIM | É `type: text` com `context.forwarded: true`. Webhook processa normalmente. |
| **Reply/quoted text** | ✅ SIM | É `type: text` com `context.quoted_message_id`. Webhook processa normalmente. |
| **Sticker** | ⚠️ PARCIAL | Payload chega com `type: sticker`. O Switch avalia o type ANTES de tentar download. Se Switch não tem branch pra sticker → testa se crasha ou tem fallback. Media ID fake não importa — o teste é sobre o Switch, não o download. |
| **Reaction** | ⚠️ PARCIAL | Payload tem estrutura diferente (sem campo `text`). Testa se o webhook trigger aceita ou rejeita. |
| **Location** | ⚠️ PARCIAL | `type: location` com lat/long. Testa se Switch tem fallback. Não precisa de dados reais. |
| **Video** | ⚠️ PARCIAL | Mesmo caso do sticker — testa o Switch, não o conteúdo. |
| **Contacts** | ⚠️ PARCIAL | `type: contacts` com dados de contato. Testa o Switch. |

### O que realmente testamos

**Para tipos não-texto (sticker, reaction, video, location, contacts):**
O teste NÃO verifica processamento do conteúdo. O teste verifica se o **Switch do Main workflow tem fallback** para tipos desconhecidos. Se não tem → workflow crasha (execução = error). Se tem → resposta educada ou silêncio controlado.

**Para forwarded/reply:**
O teste verifica se mensagens encaminhadas e replies são **processadas como texto normal**, já que o `type` é `text`.

### Caminho no Switch

```
Main → Switch (tipo de mídia):
  ├── texto → fluxo normal
  ├── audio → Download → Whisper → fluxo normal
  ├── image/document → Download → OCR → fluxo normal
  └── DEFAULT / NENHUM → ???
      → Se existe nó default: resposta educada ("tipo não suportado")
      → Se NÃO existe default: workflow ERROR (crash silencioso)
```

---

## 2. Algoritmo de execução

### Para tipos não-texto (sticker, reaction, video, etc):

```
PASSO 1 — SNAPSHOT ANTES
  1.1  Contar spent → SPENT_ANTES
  1.2  Contar calendar → CAL_ANTES
  1.3  Último log_id → LAST_LOG_ID
  1.4  Contar execuções Main → EXEC_COUNT_ANTES

PASSO 2 — ENVIAR PAYLOAD com type não-padrão (media ID fake é OK — o Switch avalia antes do download)

PASSO 3 — VERIFICAR EXECUÇÃO (não pollar resposta — pode não ter)
  3.1  Aguardar 10s
  3.2  GET /executions?workflowId=hLwhn94JSHonwHzl&limit=3
  3.3  Se nova execução existe:
       → status = success? → Switch tem fallback → PASS
       → status = error? → Switch crashou com tipo desconhecido → FAIL (documentar erro)
  3.4  Se nenhuma execução nova:
       → Webhook rejeitou o payload antes do workflow → PASS (rejeição válida)

PASSO 4 — VERIFICAR SIDE EFFECTS
  4.1  spent = SPENT_ANTES (nada criado)
  4.2  calendar = CAL_ANTES (nada criado)
  4.3  Verificar log_users_messages: algo logado?

PASSO 5 — REGISTRAR comportamento observado
```

### Para forwarded/reply (texto com context):

```
PASSO 1 — SNAPSHOT ANTES (igual a teste de texto normal)

PASSO 2 — ENVIAR PAYLOAD tipo text com context.forwarded ou context.quoted_message_id

PASSO 3 — POLLAR RESPOSTA (max 45s — é texto, deve processar normalmente)

PASSO 4 — VERIFICAR
  4.1  Resposta existe? ai_message presente?
  4.2  Se input era "gastei 50 no almoço" (forwarded): spent +1? Ou ignorou por ser forwarded?
  4.3  Dados corretos se criou registro

PASSO 5 — REGISTRAR
```

---

## 3. Critérios de PASS/FAIL

### Tipos não-texto (sticker, reaction, video, location, contacts)

| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | Workflow não crashou | Execução = success OU sem execução (rejeitou cedo) | Execução = error |
| 2 | Sem side effects | spent e calendar inalterados | Criou registro fantasma |
| 3 | Sem loop | No máximo 1 execução nova | Múltiplas execuções (loop) |

> **NOTA:** Não esperar resposta da IA para sticker/reaction. Silêncio controlado é PASS.

### Forwarded / Reply (texto com context)

| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | Resposta existe | ai_message presente | Timeout (ignorou forwarded) |
| 2 | Processou como texto | Se "gastei 50": spent +1 (tratou normalmente) | Ignorou conteúdo |
| 3 | Dados corretos | value=50, name coerente | Dados errados |

---

## 4. Protocolo de diagnóstico de erros

```
CAMADA 1 — WEBHOOK: Payload foi aceito? HTTP 200?
  Se HTTP != 200 → webhook rejeitou o formato
CAMADA 2 — SWITCH MÍDIA: O type do payload caiu em qual branch?
  Verificar: GET /executions/{exec_id} → nó Switch → qual output ativou
  Se nenhum output → Switch não tem branch pro type → provável causa do crash
CAMADA 3 — FALLBACK: Existe nó default no Switch?
  Verificar: GET /workflows/hLwhn94JSHonwHzl → analisar nó Switch → outputs
CAMADA 4 — RESPOSTA: Se fallback existe, respondeu algo ao user?
CAMADA 5 — SIDE EFFECTS: Algo foi criado indevidamente no banco?
```

---

## 5. Testes

**🟢 Quick (3 testes):**

| ID | Input | Verificação |
|----|-------|-------------|
| MID-Q1 | Payload `"type": "sticker"` (media ID fake) | Workflow não crashou (execution != error). Nada criado no banco. Testa: Switch tem fallback? |
| MID-Q2 | Payload `"type": "reaction"` (emoji 👍, message_id fake) | Workflow não crashou. Nada criado. Reaction não deveria gerar resposta. |
| MID-Q3 | Texto `"forwarded": true` com body "gastei 50 no almoço" | Mensagem encaminhada processada como texto normal? spent: +1? Ou ignorou por ser forwarded? Documentar. |

### Payload MID-Q1 (Sticker):

```json
{
  "object": "whatsapp_business_account",
  "entry": [{"id": "744582292082931", "changes": [{"value": {
    "messaging_product": "whatsapp",
    "metadata": {"display_phone_number": "554384983452", "phone_number_id": "744582292082931"},
    "contacts": [{"profile": {"name": "Luiz Felipe"}, "wa_id": "554391936205"}],
    "messages": [{
      "from": "554391936205",
      "id": "wamid.TEST_STICKER_{{timestamp}}",
      "timestamp": "{{timestamp}}",
      "type": "sticker",
      "sticker": {"mime_type": "image/webp", "sha256": "fake", "id": "FAKE_MEDIA_ID"}
    }]
  }, "field": "messages"}]}]
}
```

> **POR QUE MEDIA ID FAKE FUNCIONA:** O Switch do Main avalia `messages[0].type` ANTES de chamar Download/Get Media Info. Se `type = sticker` e o Switch não tem branch pra isso, o workflow vai pra default (se existir) ou crasha. O download nunca é tentado. O teste é sobre o **roteamento**, não sobre o conteúdo do sticker.

### Payload MID-Q3 (Forwarded text):

```json
{
  "object": "whatsapp_business_account",
  "entry": [{"id": "744582292082931", "changes": [{"value": {
    "messaging_product": "whatsapp",
    "metadata": {"display_phone_number": "554384983452", "phone_number_id": "744582292082931"},
    "contacts": [{"profile": {"name": "Luiz Felipe"}, "wa_id": "554391936205"}],
    "messages": [{
      "from": "554391936205",
      "id": "wamid.TEST_FWD_{{timestamp}}",
      "timestamp": "{{timestamp}}",
      "type": "text",
      "text": {"body": "gastei 50 no almoço"},
      "context": {"forwarded": true}
    }]
  }, "field": "messages"}]}]
}
```

> **POR QUE ESTE É 100% TESTÁVEL:** Forwarded é `type: text`. O webhook processa normalmente. O campo `context.forwarded` pode ou não ser lido pelo workflow. O teste verifica se o CONTEÚDO é processado independente de ser encaminhado.

**🟡 Broad (Quick + 5 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| MID-B1 | Payload `"type": "location"` (lat/long fake) | Switch tem fallback pra location? Não crashou? |
| MID-B2 | Payload `"type": "video"` (media ID fake) | Switch tem fallback pra video? |
| MID-B3 | Payload `"type": "contacts"` (dados fake) | Switch tem fallback pra contacts? |
| MID-B4 | Texto com `"context": {"forwarded": true}` body "o que tenho hoje?" | Forwarded com consulta de agenda: processa? calendar intacto. |
| MID-B5 | Texto com `"context": {"quoted_message_id": "wamid.FAKE"}` body "sim" | Reply/quoted: IA usa contexto do quote? Ou trata como "sim" isolado? |

**🔴 Complete (Broad + 3 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| MID-C1 | Sticker seguido de texto "gastei 30 no café" (2 msgs, 5s intervalo) | Sticker não travou o state do user. Segunda msg processa normalmente. |
| MID-C2 | 5 reactions seguidas (mesmo message_id) | Não causa loop no bot_guard. User não é bloqueado. |
| MID-C3 | Payload com `"type": "poll"` (tipo inventado) | Tipo que não existe na API oficial: workflow não crashou? |

---

## 6. Formato do log

```markdown
| ID | Tipo mídia | HTTP aceito? | Exec status | Resposta? | Side effects? | Veredicto |
```

---

## 7. Problemas conhecidos

| Problema | Impacto | Como lidar |
|----------|---------|------------|
| Media ID fake vai falhar se Switch tentar download | Download node retorna erro | Teste verifica o Switch, não o download — aceitável |
| Reaction tem estrutura sem `text` field | Pode crashar nós que esperam `text.body` | Se crashou → bug no workflow, não no teste |
| `context.forwarded` pode não ser lido | Sistema pode ignorar o flag e processar normalmente | Comportamento aceitável — documentar |

---

## 8. Melhorias sugeridas

| O que | Impacto |
|-------|---------|
| Default no Switch de mídia pra tipos desconhecidos | Tipos não cobertos respondem "formato não suportado" em vez de crashar |
| Tratar reaction como no-op (não processar) | Reactions não devem gerar execução completa do workflow |
| Ignorar `context.forwarded` (processar conteúdo normalmente) | Texto encaminhado tem mesmo valor que texto digitado |
| Logar tipo de mídia no log_total | Saber distribuição de uso e identificar tipos não tratados |
