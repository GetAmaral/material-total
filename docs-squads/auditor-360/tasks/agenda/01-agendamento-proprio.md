# Metodologia de Auditoria — Agendamento Próprio (Criar Eventos)

**Funcionalidade:** `agenda/01-agendamento-proprio`
**Versão:** 1.0.0

---

## 1. Mapa do sistema

### Caminho da mensagem

```
WhatsApp → Main → Fix Conflito v2
  → Escolher Branch → criar_evento_agenda | criar_lembrete_agenda | criar_evento_recorrente
    → prompt_criar1 / prompt_lembrete / prompt_lembrete1 (set prompt)
    → AI Agent processa
      → Tool HTTP: Calendar WebHooks (webhook Webhook-Calendar-Creator)
        → Verifica Google Calendar conectado
          ├── SIM → criar_evento_google (Google Calendar API) + create_calendar_sup_google (Supabase)
          └── NÃO → create_calendar_sup_google1 (só Supabase)
        → Responde JSON ao AI Agent
    → IA responde "✅ Evento agendado!"
    → Log em log_users_messages
```

### Workflows

| Workflow | ID | Papel |
|----------|----|-------|
| Fix Conflito v2 | `ImW2P52iyCS0bGbQ` | Classificador + AI Agent |
| Calendar WebHooks | `sSEBeOFFSOapRfu6` | CRUD real no `calendar` + Google sync |

### Branches do classificador

| Input | Branch | Prompt |
|-------|--------|--------|
| "reunião amanhã 10h" | `criar_evento_agenda` | `prompt_criar1` |
| "me lembra de comprar pão" | `criar_lembrete_agenda` | `prompt_lembrete` |
| "academia toda segunda 7h" | `criar_evento_recorrente` | `prompt_lembrete1` |

### Tabela: `calendar`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | UUID | PK |
| `event_name` | TEXT | Nome do evento |
| `start_event` | TIMESTAMP | Início |
| `end_event` | TIMESTAMP | Fim (padrão: start + 30min) |
| `is_recurring` | BOOLEAN | Recorrente? |
| `rrule` | TEXT | Regra de recorrência (RRULE) |
| `reminder` | BOOLEAN | É lembrete? |
| `remembered` | BOOLEAN | Já foi lembrado? |
| `user_id` | UUID | FK user |
| `connect_google` | BOOLEAN | Sincronizado com Google? |
| `session_event_id_google` | TEXT | ID do evento no Google Calendar |
| `compromisso_tipo` | TEXT | "compromisso" ou "lembrete" |
| `active` | BOOLEAN | Ativo? |
| `due_at` | TIMESTAMP | Quando disparar lembrete |
| `timezone` | TEXT | Fuso (America/Sao_Paulo) |
| `exdates` | TEXT | Datas excluídas (recorrentes) |
| `next_fire_at` | TIMESTAMP | Próximo disparo (recorrentes) |

### Comportamento

- Evento pontual: `is_recurring=false`, `rrule=null`
- Lembrete: `compromisso_tipo="lembrete"`, `reminder=true`
- Recorrente: `is_recurring=true`, `rrule` preenchido (ex: `FREQ=WEEKLY;BYDAY=MO,WE,FR`)
- Sem horário: sistema define horário padrão (9h)
- Google sync: se `connect_google=true`, cria no Google Calendar e salva `session_event_id_google`

---

## 2. Endpoints de verificação

| Verificação | Query |
|-------------|-------|
| Buscar por nome | `GET /calendar?event_name=ilike.*{nome}*&user_id=eq.{user_id}&order=created_at.desc&limit=1` |
| Contar total | `GET /calendar?select=id&user_id=eq.{user_id}` + `Prefer: count=exact` |
| Criados hoje | `GET /calendar?user_id=eq.{user_id}&created_at=gte.{hoje}T00:00:00&order=created_at.desc` |
| Recorrentes | `GET /calendar?user_id=eq.{user_id}&is_recurring=eq.true` |
| Execuções Calendar WH | `GET /executions?workflowId=sSEBeOFFSOapRfu6&limit=5` (N8N API) |

---

## 3. Algoritmo de execução

```
PASSO 1 — SNAPSHOT ANTES
  1.1  Contar calendar do user → COUNT_ANTES
  1.2  Último log_id → LAST_LOG_ID

PASSO 2 — ENVIAR MENSAGEM

PASSO 3 — POLLAR RESPOSTA (log_users_messages)

PASSO 4 — VERIFICAR BANCO
  4.1  Contar calendar → COUNT_DEPOIS = COUNT_ANTES + 1?
  4.2  Buscar evento por nome → REGISTRO
  4.3  Cruzar campos:
       - event_name match com input
       - start_event = data/hora correta
       - end_event = start + 30min (ou duração implícita)
       - compromisso_tipo = "compromisso" ou "lembrete" (correto?)
       - is_recurring = false (se pontual)
       - active = true
       - user_id = user de teste
       - connect_google = true (se user tem Google)
       - session_event_id_google não null (se Google conectado)
       - timezone = America/Sao_Paulo

PASSO 5 — REGISTRAR
```

---

## 4. Critérios de PASS/FAIL

| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | IA confirmou | "✅ Evento agendado!" | Outro texto |
| 2 | COUNT +1 | COUNT_DEPOIS = COUNT_ANTES + 1 | Diferente |
| 3 | event_name | Coerente com input | Diferente |
| 4 | start_event | Data/hora corretas | Erradas |
| 5 | end_event (sem duração) | end_event = start_event + 30min | end_event NULL, = start_event, ou > start + 24h |
| 6 | end_event (com duração) | end_event = start_event + duração informada | Diferente do esperado |
| 7 | end_event > start_event | SEMPRE verdade | end_event <= start_event = DADO CORROMPIDO |
| 8 | compromisso_tipo | Correto pro contexto | Invertido |
| 9 | Não duplicou | Exatamente 1 novo evento com esse nome | 2+ |
| 10 | Google sync | session_event_id_google presente (se conectado) | null quando deveria ter |
| 11 | timezone | "America/Sao_Paulo" (ou equivalente) | null, UTC puro, ou timezone errado |

> **INSTRUÇÃO OBRIGATÓRIA:** Em TODOS os testes de criação de evento (AGD-Q1 a Q5, B1 a B8), verificar end_event e timezone. Não pular estes campos.

---

## 5. Protocolo de diagnóstico de erros

Mesmo protocolo base de `financeiro/02-limites-categoria.md`, com camadas:

```
CAMADA 1 — CLASSIFICADOR: Foi pra branch certa? (criar_evento vs criar_lembrete vs recorrente)
CAMADA 2 — AI AGENT: Chamou tool HTTP correta?
CAMADA 3 — CALENDAR WEBHOOKS: Webhook-Calendar-Creator executou?
CAMADA 4 — GOOGLE: Se connect_google=true, Google API retornou sucesso?
CAMADA 5 — SUPABASE: INSERT no calendar executou?
CAMADA 6 — RESPOSTA: IA formatou resposta corretamente?
```

---

## 6. Testes

**🟢 Quick (5 testes):**

| ID | Input | Verificação |
|----|-------|-------------|
| AGD-Q1 | "reunião amanhã às 14h" | calendar: +1, start=amanhã 14:00, tipo=compromisso. end_event = 14:30 (default 30min). |
| AGD-Q2 | "me lembra de tomar remédio 20h" | calendar: +1, tipo=lembrete, reminder=true. end_event > start_event. |
| AGD-Q3 | "dentista dia 25 às 9h" | calendar: +1, start=25/mês 09:00. end_event = 09:30. |
| AGD-Q4 | "me lembra de comprar pão" | calendar: +1, sem horário → default 9h? Documentar. end_event preenchido. |
| AGD-Q5 | "reunião amanhã às 15h" | ⚠️ Timezone: start_event contém 15:00 no horário local (America/Sao_Paulo). timezone = "America/Sao_Paulo". Se 15:00 UTC (= 12h Brasil) → FAIL. |

### Algoritmo específico AGD-Q5 (Timezone):

```
PASSO 1 — Enviar: "reunião amanhã às 15h"
PASSO 2 — Pollar resposta
PASSO 3 — GET /calendar?event_name=ilike.*reuni*&user_id=eq.{user_id}&order=created_at.desc&limit=1
           CHECK 1: timezone = "America/Sao_Paulo"
           CHECK 2: start_event equivale a 15h BRT (pode ser 15:00-03:00 ou 18:00Z)
           CHECK 3: start_event NÃO é 15:00 UTC puro (senão = 12h no Brasil)

           PASS: 15h no horário brasileiro
           FAIL: timezone null ou errado
           FAIL: 15:00 UTC (= 12h Brasil)
CLEANUP:   DELETE /calendar?event_name=ilike.*reuni*&user_id=eq.{user_id}&created_at=gte.{TS_ANTES}
```

**🟡 Broad (Quick + 8 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| AGD-B1 | "consulta médica segunda às 10h30" | Dia da semana + horário com minutos. end_event = 11:00. |
| AGD-B2 | "reunião de 2 horas amanhã 14h" | end_event = start + 2h (não 30min padrão). Verificar: end_event = 16:00. |
| AGD-B3 | "almoço hoje 12h" | Evento no mesmo dia. end_event = 12:30. |
| AGD-B4 | "compromisso" (sem detalhes) | IA pede mais informações? |
| AGD-B5 | Verificar Google sync | session_event_id_google preenchido |
| AGD-B6 | Verificar duplicata | Criar mesmo evento 2x → 2 registros separados ou conflito? |
| AGD-B7 | "marca reunião ontem às 10h" | Evento no passado: aceita ou recusa? Se criou → date_spent no passado é intencional? Documentar comportamento. |
| AGD-B8 | SETUP: criar evento amanhã 14h. TESTE: "marca dentista amanhã 14h" | Conflito de horário: IA avisa sobre conflito? Cria em cima? Sugere outro horário? Documentar comportamento. |
| AGD-B9 | MSG1: "marca reunião" → pollar → MSG2: "amanhã às 14h" | Multi-turno agenda: IA pede detalhes na MSG1, registra na MSG2. calendar: +1, start=amanhã 14:00. Se não criou → contexto perdido. |
| AGD-B10 | MSG1: "me lembra de algo" → pollar → MSG2: "tomar remédio às 9h" | Multi-turno lembrete: IA pede detalhes, registra na segunda msg. calendar: +1, tipo=lembrete. |

**🔴 Complete (Broad + 4 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| AGD-C1 | "evento o dia todo sábado" | Evento all-day? start/end corretos? |
| AGD-C2 | "reunião em 15 minutos" | Data relativa curta |
| AGD-C3 | Nome com emoji "🎂 aniversário" | event_name sem emoji? |
| AGD-C4 | Verificar todos os campos | Todos 15+ campos do calendar coerentes |

---

## 7. Formato do log

```markdown
| ID | Input | IA disse | event_name | start_event | end_event | tipo | Google sync | Veredicto |
```

---

## 8. Melhorias sugeridas

| O que | Impacto |
|-------|---------|
| Logar `id` do evento criado no `log_total` | Rastreio direto |
| Padronizar comportamento "sem horário" | Documentar default (9h?) |
| Validar end_event = start + duração | Evitar bug de end_event desatualizado |
