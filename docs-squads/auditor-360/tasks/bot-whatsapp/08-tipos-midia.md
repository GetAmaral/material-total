# Metodologia de Auditoria — Tipos de Mídia e Mensagens Especiais

**Funcionalidade:** `bot-whatsapp/08-tipos-midia`
**Versão:** 1.0.0

---

## 1. Mapa do sistema

### O que é

O WhatsApp permite enviar diversos tipos de mensagem além de texto. O roteador (Main workflow) tem um Switch que verifica o tipo de mídia (`texto`, `audio`, `image/document`). Mas existem tipos que NÃO estão no Switch:

| Tipo | Frequência | Coberto pelo Switch? |
|------|-----------|---------------------|
| `text` | Sempre | ✅ |
| `audio` | Frequente | ✅ |
| `image` | Frequente | ✅ |
| `document` (PDF) | Às vezes | ✅ |
| **`sticker`** | **Muito frequente** | ❓ Verificar |
| **`reaction`** | **Frequente** | ❓ Verificar |
| **`video`** | Às vezes | ❓ Verificar |
| **`location`** | Às vezes | ❓ Verificar |
| **`contacts`** | Raro | ❓ Verificar |
| **Forwarded msg** | Frequente | ❓ Verificar |
| **Reply/quoted** | Frequente | ❓ Verificar |

### Caminho esperado

```
WhatsApp → Main → Switch (tipo de mídia):
  ├── texto → fluxo normal
  ├── audio → Whisper → fluxo normal
  ├── image/document → OCR → fluxo normal
  └── OUTRO (sticker, video, location, etc) → ???
      → Idealmente: resposta educada "não consigo processar este tipo"
      → Pior caso: crash do workflow / sem resposta / loop
```

### Payload WhatsApp para cada tipo

**Sticker:**
```json
"messages": [{"type": "sticker", "sticker": {"mime_type": "image/webp", "sha256": "...", "id": "..."}}]
```

**Reaction:**
```json
"messages": [{"type": "reaction", "reaction": {"message_id": "wamid.XXX", "emoji": "👍"}}]
```

**Location:**
```json
"messages": [{"type": "location", "location": {"latitude": -25.42, "longitude": -49.27, "name": "Escritório"}}]
```

**Video:**
```json
"messages": [{"type": "video", "video": {"mime_type": "video/mp4", "id": "..."}}]
```

**Forwarded message:**
```json
"messages": [{"type": "text", "text": {"body": "gastei 50 no almoço"}, "context": {"forwarded": true}}]
```

**Reply/quoted:**
```json
"messages": [{"type": "text", "text": {"body": "sim"}, "context": {"quoted_message_id": "wamid.XXX"}}]
```

---

## 2. Algoritmo de execução

```
PASSO 1 — SNAPSHOT ANTES
  1.1  Contar spent → SPENT_ANTES
  1.2  Contar calendar → CAL_ANTES
  1.3  Último log_id → LAST_LOG_ID

PASSO 2 — ENVIAR PAYLOAD com tipo de mídia não-padrão

PASSO 3 — POLLAR RESPOSTA (max 45s)
  Se resposta → capturar ai_message
  Se timeout → marcar TIMEOUT (pode ser crash silencioso)

PASSO 4 — VERIFICAR
  4.1  Workflow Main executou? (GET /executions)
  4.2  Status = success? (se error → o tipo de mídia quebrou o workflow)
  4.3  Resposta existe no log_users_messages?
  4.4  spent e calendar inalterados (tipo de mídia não deveria criar registros)

PASSO 5 — REGISTRAR
```

---

## 3. Critérios de PASS/FAIL

| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | Workflow não crashou | Execução Main = success (ou sem execução se rejeitou cedo) | Execução = error |
| 2 | Resposta coerente | "Não consigo processar stickers" ou similar | Erro, crash, resposta de outra branch |
| 3 | Sem side effects | spent e calendar inalterados | Criou registro fantasma |
| 4 | Sem loop | Apenas 1 execução do Main (não entrou em loop) | Múltiplas execuções repetidas |
| 5 | Forwarded text processado | Texto encaminhado é tratado como texto normal | Ignorou ou crashou |

---

## 4. Protocolo de diagnóstico de erros

```
CAMADA 1 — WEBHOOK: Payload foi aceito? HTTP 200?
CAMADA 2 — SWITCH MÍDIA: O tipo caiu em qual branch? (texto/audio/image/nenhum)
  Se nenhum → tipo não tratado → CRASH provável
CAMADA 3 — FALLBACK: Existe nó default no Switch pra tipos desconhecidos?
CAMADA 4 — RESPOSTA: IA respondeu algo? Ou ficou muda?
CAMADA 5 — SIDE EFFECTS: Algo foi criado indevidamente no banco?
```

---

## 5. Testes

**🟢 Quick (3 testes):**

| ID | Input | Verificação |
|----|-------|-------------|
| MID-Q1 | Payload tipo `sticker` | Workflow não crashou. Resposta educada OU silêncio controlado. Nada criado no banco. |
| MID-Q2 | Payload tipo `reaction` (emoji 👍) | Workflow não crashou. Nenhuma resposta necessária (reaction não espera reply). Nada criado. |
| MID-Q3 | Texto `forwarded: true` "gastei 50 no almoço" | Mensagem encaminhada com gasto: sistema processa como texto normal? spent: +1? Ou ignora por ser forwarded? Documentar. |

### Algoritmo específico MID-Q1 (Sticker):

```
PASSO 1 — SNAPSHOT: SPENT_ANTES, CAL_ANTES, LAST_LOG_ID, EXEC_COUNT
PASSO 2 — POST webhook com payload sticker:
  {
    "object": "whatsapp_business_account",
    "entry": [{"id": "744582292082931", "changes": [{"value": {
      "messaging_product": "whatsapp",
      "metadata": {"display_phone_number": "554384983452", "phone_number_id": "744582292082931"},
      "contacts": [{"profile": {"name": "Luiz Felipe"}, "wa_id": "554391936205"}],
      "messages": [{"from": "554391936205", "id": "wamid.TEST_STICKER_1", "timestamp": "{{timestamp}}", "type": "sticker", "sticker": {"mime_type": "image/webp", "sha256": "abc123", "id": "STICKER_MEDIA_ID"}}]
    }, "field": "messages"}]}]
  }
PASSO 3 — Pollar log (max 30s). Pode não ter resposta (aceitável).
PASSO 4 — GET /executions?workflowId=hLwhn94JSHonwHzl&limit=3
           Se última execução = error → FAIL (sticker crashou o workflow)
           Se última execução = success → PASS (tratou sem crashar)
           Se nenhuma execução nova → PASS (rejeitou antes do workflow — aceitável)
PASSO 5 — Verificar spent e calendar inalterados
```

**🟡 Broad (Quick + 5 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| MID-B1 | Payload tipo `location` (lat/long) | Workflow não crashou. Documentar: ignora? Responde? |
| MID-B2 | Payload tipo `video` | Workflow não crashou. Não tentou transcrever como áudio. |
| MID-B3 | Payload tipo `contacts` | Workflow não crashou. |
| MID-B4 | Texto com `context.forwarded: true` "o que tenho hoje?" | Encaminhada com consulta de agenda: processa normalmente? |
| MID-B5 | Texto com `context.quoted_message_id` (reply) | Reply a mensagem anterior: IA usa contexto do quote? Ou ignora? |

**🔴 Complete (Broad + 3 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| MID-C1 | Sticker seguido de texto "gastei 30" | Sticker não travou e segundo msg é processada normalmente |
| MID-C2 | Reaction em loop (5 reactions seguidas) | Não causa loop no bot_guard, não bloqueia user |
| MID-C3 | Payload com type desconhecido ("type": "poll") | Tipo inventado: workflow não crashou |

---

## 6. Formato do log

```markdown
| ID | Tipo mídia | Workflow crashou? | Resposta? | Side effects? | Veredicto |
```

---

## 7. Melhorias sugeridas

| O que | Impacto |
|-------|---------|
| Default no Switch de mídia | Tipos desconhecidos não crasham — respondem "tipo não suportado" |
| Tratar reaction como no-op | Reactions não devem gerar execução do workflow inteiro |
| Logar tipo de mídia no log_total | Saber distribuição de texto/audio/image/sticker |
