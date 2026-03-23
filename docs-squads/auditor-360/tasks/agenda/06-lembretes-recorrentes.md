# Metodologia de Auditoria — Lembretes Recorrentes

**Funcionalidade:** `agenda/06-lembretes-recorrentes`
**Versão:** 1.0.0

---

## 1. Mapa do sistema

### Criação de recorrente

```
WhatsApp → Fix Conflito v2
  → Escolher Branch → criar_evento_recorrente
    → prompt_lembrete1 → AI Agent → tools
      → Lembretes Total Assistente (webhook "Criar Lembrete Recorrente")
        → Extrair Campos (set)
        → Calcular end & Recurrence (code) — gera rrule
        → If Conectado ao Google?
          ├── SIM → criar_evento_google_recorrente → create_calendar_sup_google (rec)
          └── NÃO → create_calendar_sup_local (rec)
```

### Disparo automático

```
Lembretes Total Assistente:
  → Schedule Trigger (1 min) — roda a cada 1 minuto
    → Get Reminders (due now) — SELECT calendar WHERE due_at <= now AND remembered=false
    → Get Recurring Reminders — SELECT is_recurring=true WHERE next_fire_at <= now
    → IF Recorrente?
      ├── SIM → Avançar next_fire_at (code) → UPDATE calendar
      └── NÃO → Mark as Remembered (UPDATE remembered=true)
    → Envia lembrete via WhatsApp (template ou texto)
    → Create a row (log_total: lembrete_automatico)
```

### Workflow: Lembretes Total Assistente (`b3xKlSunpwvC4Vwh`) — 54 nós

### Campos relevantes de recorrentes no `calendar`

| Campo | Uso em recorrentes |
|-------|-------------------|
| `is_recurring` | true |
| `rrule` | Ex: `FREQ=WEEKLY;BYDAY=MO,WE,FR` ou `FREQ=MONTHLY;BYMONTHDAY=5` |
| `next_fire_at` | Próximo disparo (avançado após cada lembrete) |
| `last_fired_at` | Último disparo |
| `repeats_until` | Data limite de repetição (null = infinito) |
| `exdates` | Datas excluídas (quando cancela 1 ocorrência) |

### Bugs conhecidos

- **Duplicação**: criar recorrente pode gerar múltiplos registros
- **next_fire_at errado**: cálculo pode falhar em edge cases (fim do mês, fuso)
- **Exclusão de uma ocorrência**: inconsistente (A-200)

---

## 2. Algoritmo de execução

### Para CRIAÇÃO:

```
PASSO 1 — Contar eventos com is_recurring=true → COUNT_REC_ANTES
PASSO 2 — Enviar mensagem ("academia toda segunda 7h")
PASSO 3 — Pollar resposta
PASSO 4 — Verificar:
  4.1  COUNT_REC_DEPOIS = COUNT_REC_ANTES + 1 (exatamente 1, sem duplicata)
  4.2  is_recurring = true
  4.3  rrule contém FREQ e BYDAY/BYMONTHDAY corretos
  4.4  next_fire_at calculado corretamente
  4.5  Se Google: evento recorrente criado no Google
```

### Para DISPARO AUTOMÁTICO:

```
PASSO 1 — Criar recorrente com horário próximo (daqui 2 min)
PASSO 2 — Esperar 3 min
PASSO 3 — Verificar:
  3.1  log_total: entrada "lembrete_automatico" com o nome do evento
  3.2  calendar: next_fire_at avançou para próxima ocorrência
  3.3  last_fired_at atualizado
```

---

## 3. Critérios de PASS/FAIL

### Criação

| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | is_recurring | true | false |
| 2 | rrule | FREQ correto + BYDAY/BYMONTHDAY correto | Vazio ou errado |
| 3 | Sem duplicata | Exatamente +1 registro | +2 ou mais |
| 4 | next_fire_at | Data/hora da próxima ocorrência | null ou errado |
| 5 | Google sync | Evento recorrente no Google | Só local |

### Disparo

| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | Lembrete disparou | log_total: lembrete_automatico | Sem entrada |
| 2 | next_fire_at avançou | Nova data = próxima ocorrência | Não avançou |
| 3 | Mensagem enviada | User recebeu lembrete | Sem mensagem |

---

## 4. Protocolo de diagnóstico de erros

```
CAMADA 1 — CLASSIFICADOR: Foi pra criar_evento_recorrente?
CAMADA 2 — WORKFLOW LEMBRETES: Webhook "Criar Lembrete Recorrente" executou?
CAMADA 3 — RRULE: Calcular end & Recurrence gerou rrule correto?
CAMADA 4 — GOOGLE: Recorrente criado no Google?
CAMADA 5 — SCHEDULE: Schedule Trigger está rodando? (a cada 1 min)
CAMADA 6 — QUERY: Get Reminders (due now) retornou o evento?
CAMADA 7 — AVANÇO: Avançar next_fire_at calculou corretamente?
```

---

## 5. Testes

**🟢 Quick (4 testes):**

| ID | Input | Verificação |
|----|-------|-------------|
| REC-Q1 | "yoga toda terça e quinta 7h" | is_recurring=true, rrule=FREQ=WEEKLY;BYDAY=TU,TH |
| REC-Q2 | "me lembra todo dia 5 de pagar aluguel" | rrule=FREQ=MONTHLY;BYMONTHDAY=5 |
| REC-Q3 | "cancela a yoga" | Registro removido ou desativado. Se permanece ativo → FAIL. |
| REC-Q4 | "academia toda segunda e sexta 18h" | ⚠️ Anti-duplicação: calendar EXATAMENTE +1 registro com is_recurring=true. Se +2 ou mais → FAIL (bug de duplicação conhecido). |

### Algoritmo específico REC-Q4 (Anti-duplicação):

```
PASSO 1 — GET /calendar?user_id=eq.{user_id}&is_recurring=eq.true&event_name=ilike.*academia*
           Esperado: 0 registros
PASSO 2 — Enviar: "academia toda segunda e sexta às 18h"
PASSO 3 — Pollar resposta (max 45s)
PASSO 4 — Aguardar 10s (dar tempo pro workflow terminar)
PASSO 5 — GET /calendar?user_id=eq.{user_id}&is_recurring=eq.true&event_name=ilike.*academia*
           PASS: Exatamente 1 registro, rrule contém BYDAY=MO,FR
           FAIL: 2+ registros (duplicou)
           FAIL: 0 registros (não criou)
CLEANUP:   DELETE /calendar?event_name=ilike.*academia*&user_id=eq.{user_id}
```

**🟡 Broad (Quick + 8 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| REC-B1 | Criar mesmo recorrente 2x | NÃO duplica? Ou cria 2? |
| REC-B2 | "cancela a yoga de terça" | Exclui só 1 dia (exdates)? |
| REC-B3 | Verificar next_fire_at | Calculado corretamente pra próxima ocorrência |
| REC-B4 | "todo dia às 22h me lembra de tomar remédio" | FREQ=DAILY, next_fire_at = hoje 22h |
| REC-B5 | Google sync de recorrente | Evento recorrente aparece no Google Calendar |
| REC-B6 | "me lembra todo dia 31 de pagar X" | ⚠️ Edge case fim de mês: rrule=FREQ=MONTHLY;BYMONTHDAY=31. next_fire_at = próximo mês com dia 31 (pula fevereiro?). Documentar comportamento. |
| REC-B7 | "me lembra toda segunda 8h" (verificar timezone) | ⚠️ next_fire_at deve estar no timezone correto (America/Sao_Paulo), não UTC. Se 8h UTC (= 5h Brasil) → FAIL. |
| REC-B8 | SETUP: criar "remédio toda manhã 9h". TESTE: "muda o lembrete do remédio pra 10h" | Editar recorrente: start_event muda de 9h pra 10h. next_fire_at recalculado. rrule intacta. Se não mudou → FAIL. |

**🔴 Complete (Broad + 3 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| REC-C1 | Disparo automático | Criar com horário próximo, esperar, verificar log |
| REC-C2 | next_fire_at após disparo | Avançou para próxima ocorrência |
| REC-C3 | repeats_until | Se definido, para de disparar após a data |

---

## 6. Formato do log

```markdown
| ID | Input | is_recurring | rrule | next_fire_at | duplicou? | Google sync | Veredicto |
```

---

## 7. Melhorias sugeridas

| O que | Impacto |
|-------|---------|
| Verificação anti-duplicata antes de criar | Evita múltiplos registros |
| Logar rrule gerado no log_total | Verificar sem acessar calendar |
| Suporte a exdates pra excluir 1 ocorrência | Funcionalidade incompleta hoje |
