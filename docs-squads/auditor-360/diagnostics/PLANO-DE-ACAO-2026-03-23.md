# PLANO DE AÇÃO — CORREÇÃO DO WORKFLOW DE TESTES AUDITOR-360

**Data:** 2026-03-23
**Baseado em:** AUDITORIA-CRITICA-2026-03-23.md (34 issues)
**Foco principal:** Tornar os testes mastigados para a IA auditora (Lupa)
**Escopo:** Financeiro (despesas-receitas) + Agenda (6 tasks ativas)

---

## COMO ESTE PLANO FUNCIONA

Cada ação abaixo contém:
1. **O que mudar** — arquivo e seção exata
2. **Teste mastigado** — input, payload, verificação SQL, critério PASS/FAIL prontos para copiar
3. **Prioridade** — ordem de implementação
4. **Esforço** — estimativa de complexidade

A IA auditora (Lupa) lê os `.md` de tasks literalmente. Então cada novo teste precisa estar no formato exato que ela entende:

```
| ID | Operação | Input | O que valida | Verificação banco |
```

---

# FASE 1 — CRITICOS (implementar primeiro)

---

## AÇÃO 1.1 — Teste de duplicação por double-tap

**Issue:** CRIT-01
**Arquivo:** `tasks/financeiro/01-despesas-receitas.md`
**Onde inserir:** Seção 5 (Quick), após FIN-Q5

### Teste mastigado para adicionar:

```markdown
| ID | Operação | Input | Output esperado | Verificação banco |
| FIN-Q6 | Double-tap | "gastei 28 na padaria" (enviar 2x com 2s intervalo) | Apenas 1 registro OU IA diz "já registrei" | spent: COUNT_DEPOIS = COUNT_ANTES + 1 (NÃO +2) |
```

### Algoritmo de execução específico:

```
PASSO 1 — SNAPSHOT ANTES
  COUNT_ANTES: GET /spent?select=id_spent&fk_user=eq.{user_id}
  LAST_LOG_ID: GET /log_users_messages?order=id.desc&limit=1

PASSO 2 — ENVIAR MENSAGEM #1
  POST webhook dev: "gastei 28 na padaria"
  Aguardar 2 segundos exatos (NÃO pollar ainda)

PASSO 3 — ENVIAR MENSAGEM #2 (mesmo texto)
  POST webhook dev: "gastei 28 na padaria"

PASSO 4 — POLLAR RESPOSTA (max 45s)
  GET /log_users_messages?id=gt.{LAST_LOG_ID}&user_phone=eq.554391936205
  Capturar TODAS as respostas (pode ter 1 ou 2)

PASSO 5 — VERIFICAR BANCO (aguardar 10s após última resposta)
  COUNT_DEPOIS: GET /spent?select=id_spent&fk_user=eq.{user_id}
  BUSCAR: GET /spent?name_spent=ilike.*padaria*&fk_user=eq.{user_id}&order=created_at.desc

PASSO 6 — CLASSIFICAR
  PASS: COUNT_DEPOIS = COUNT_ANTES + 1 (só 1 registro criado)
  FAIL: COUNT_DEPOIS = COUNT_ANTES + 2 (duplicou)
  PARTIAL: 2 respostas da IA mas só 1 registro no banco (IA duplicou resposta mas banco ok)
```

### Critérios PASS/FAIL:

```markdown
| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | Contagem | COUNT_DEPOIS = COUNT_ANTES + 1 | COUNT_DEPOIS = COUNT_ANTES + 2 |
| 2 | Registros | Busca por "padaria" retorna exatamente 1 novo | Retorna 2 novos |
| 3 | Valor | value_spent = 28 (um só) | 2x value_spent = 28 |
```

### Payload (enviar 2x):

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "744582292082931",
    "changes": [{
      "value": {
        "messaging_product": "whatsapp",
        "metadata": {"display_phone_number": "554384983452", "phone_number_id": "744582292082931"},
        "contacts": [{"profile": {"name": "Luiz Felipe"}, "wa_id": "554391936205"}],
        "messages": [{
          "from": "554391936205",
          "id": "wamid.TEST_DOUBLETAP_1",
          "timestamp": "{{timestamp}}",
          "type": "text",
          "text": {"body": "gastei 28 na padaria"}
        }]
      },
      "field": "messages"
    }]
  }]
}
```

> **NOTA:** Na segunda mensagem, usar `"id": "wamid.TEST_DOUBLETAP_2"` — IDs diferentes simulam 2 envios reais.

### Cleanup:

```
DELETE /spent?name_spent=ilike.*padaria*&fk_user=eq.{user_id}
```

### Se FAIL — recomendação para o N8N:

Adicionar no workflow `Main - Total Assistente` (hLwhn94JSHonwHzl):
- Nó `Function` antes do roteamento que verifica se `message_id` (wamid) já foi processado nos últimos 60s
- Usar tabela `processed_messages` ou cache Redis com TTL 60s
- Se duplicado → responder "Já processei essa mensagem" e parar

---

## AÇÃO 1.2 — Teste de múltiplos itens financeiros

**Issue:** CRIT-02
**Arquivo:** `tasks/financeiro/01-despesas-receitas.md`
**Onde inserir:** Seção 6 (Broad), como FIN-B11

### Teste mastigado:

```markdown
| ID | Operação | Input | O que valida | Verificação banco |
| FIN-B11 | Multi-item | "gastei 50 no almoço e 30 no uber" | Comportamento com 2 itens na mesma mensagem | Documentar: criou 1 ou 2 registros? Valores corretos? |
```

### Algoritmo:

```
PASSO 1 — SNAPSHOT ANTES
  COUNT_ANTES: GET /spent?select=id_spent&fk_user=eq.{user_id}

PASSO 2 — ENVIAR: "gastei 50 no almoço e 30 no uber"

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — VERIFICAR BANCO
  COUNT_DEPOIS: GET /spent?select=id_spent&fk_user=eq.{user_id}
  BUSCAR_ALMOCO: GET /spent?name_spent=ilike.*almo*&fk_user=eq.{user_id}&order=created_at.desc&limit=1
  BUSCAR_UBER: GET /spent?name_spent=ilike.*uber*&fk_user=eq.{user_id}&order=created_at.desc&limit=1

PASSO 5 — CLASSIFICAR
  CENÁRIO A (ideal): COUNT_DEPOIS = COUNT_ANTES + 2, ambos encontrados com valores corretos
  CENÁRIO B (aceitável): COUNT_DEPOIS = COUNT_ANTES + 1, IA registrou 1 e pediu confirmação pro 2º
  CENÁRIO C (FAIL): COUNT_DEPOIS = COUNT_ANTES + 1, registrou com valor errado (ex: R$80 somando os dois)
  CENÁRIO D (FAIL): COUNT_DEPOIS = COUNT_ANTES + 0, não registrou nada
```

### Critérios:

```markdown
| # | Critério | PASS | PARTIAL | FAIL |
|---|----------|------|---------|------|
| 1 | 2 registros criados | Almoço=50 + Uber=30 | 1 registro + IA pediu confirmação | 0 registros ou valor somado |
| 2 | Nomes corretos | name_spent coerente com cada item | — | Nome misturado |
| 3 | Valores corretos | 50 e 30 separados | — | 80 num único registro |
```

### Cleanup:

```
DELETE /spent?name_spent=ilike.*almo*&fk_user=eq.{user_id}&created_at=gte.{hoje}
DELETE /spent?name_spent=ilike.*uber*&fk_user=eq.{user_id}&created_at=gte.{hoje}
```

---

## AÇÃO 1.3 — Teste de regressão: end_event ao mover start_event

**Issue:** CRIT-03
**Arquivo:** `tasks/agenda/03-modificacao-compromissos.md`
**Onde inserir:** Seção Quick, como MOD-Q4 (NOVO — promovido de known bug para teste obrigatório)

### Teste mastigado:

```markdown
| ID | Operação | Input | Output esperado | Verificação banco |
| MOD-Q4 | Mover data (regressão end_event) | SETUP: criar evento dia {D} 14h-15h. TESTE: "muda a reunião pro dia {D+2}" | "✅ Evento editado!" | start_event = D+2 14:00 E end_event = D+2 15:00 (AMBOS devem mover) |
```

### Algoritmo:

```
SETUP — CRIAR EVENTO BASE
  Enviar: "marca reunião dia {D} às 14h" (onde D = amanhã)
  Pollar resposta + verificar calendar: +1
  Salvar: EVENT_ID, start_event, end_event
  VALIDAR: end_event = start_event + 30min ou + 1h (salvar DURACAO_ORIGINAL)

PASSO 1 — SNAPSHOT ANTES
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}
  Salvar: START_ANTES, END_ANTES, DURACAO = END_ANTES - START_ANTES

PASSO 2 — ENVIAR: "muda a reunião pro dia {D+2}"

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — WAIT 20s (async)

PASSO 5 — VERIFICAR BANCO
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}
  Salvar: START_DEPOIS, END_DEPOIS

PASSO 6 — CLASSIFICAR
  CHECK 1: START_DEPOIS contém dia D+2? (start moveu)
  CHECK 2: END_DEPOIS contém dia D+2? (end moveu junto)
  CHECK 3: END_DEPOIS - START_DEPOIS = DURACAO? (duração preservada)
  CHECK 4: END_DEPOIS > START_DEPOIS? (end nunca antes de start)

  PASS: Todos os 4 checks TRUE
  FAIL_CRITICO: CHECK 4 = FALSE (end antes de start — dado corrompido)
  FAIL: CHECK 2 = FALSE (end não moveu)
  PARTIAL: CHECK 3 = FALSE (duração mudou mas end > start)
```

### Critérios:

```markdown
| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | start_event moveu | Contém dia D+2 | Ainda dia D |
| 2 | end_event moveu junto | Contém dia D+2 | Ainda dia D ← BUG CONHECIDO |
| 3 | Duração preservada | END - START = DURACAO original | Diferente |
| 4 | end > start | SEMPRE | end < start = DADO CORROMPIDO |
```

### Cleanup:

```
DELETE /calendar?id=eq.{EVENT_ID}
```

### Se FAIL — onde corrigir no N8N:

Workflow `Calendar WebHooks` (sSEBeOFFSOapRfu6), nó `editar_evento_google3` ou `Update a row1`:
- Quando `start_event` muda e `end_event` NÃO veio no request → calcular:
  `new_end_event = new_start_event + (old_end_event - old_start_event)`
- Adicionar nó `Function` antes do UPDATE que faz esse cálculo

---

## AÇÃO 1.4 — Teste de regressão: rename deleta do Google

**Issue:** CRIT-04
**Arquivo:** `tasks/agenda/03-modificacao-compromissos.md`
**Onde inserir:** Seção Quick, como MOD-Q5

### Teste mastigado:

```markdown
| ID | Operação | Input | Verificação |
| MOD-Q5 | Rename (regressão Google delete) | SETUP: criar evento "TestRename". TESTE: "muda o nome pra PlanningTest" | calendar: event_name = PlanningTest. session_event_id_google = MESMO ID de antes. Google: evento EXISTE. |
```

### Algoritmo:

```
SETUP — CRIAR EVENTO
  Enviar: "marca TestRename amanhã às 16h"
  Pollar + verificar calendar: +1
  Salvar: EVENT_ID, GOOGLE_ID = session_event_id_google

PASSO 1 — SNAPSHOT ANTES
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}
  Salvar: event_name, session_event_id_google, connect_google

PASSO 2 — ENVIAR: "muda o nome de TestRename pra PlanningTest"

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — WAIT 20s

PASSO 5 — VERIFICAR BANCO
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}
  CHECK 1: event_name = "PlanningTest" (ou variante capitalizada)
  CHECK 2: session_event_id_google = GOOGLE_ID (mesmo ID — não recriou)
  CHECK 3: connect_google = true (ainda conectado)
  CHECK 4: active = true (não desativou)

  BUSCAR DELETADO: GET /calendar?event_name=ilike.*TestRename*&fk_user=eq.{user_id}
  CHECK 5: Retorna vazio (nome antigo sumiu — esperado, foi renomeado)

PASSO 6 — CLASSIFICAR
  PASS: Checks 1-5 todos TRUE
  FAIL_CRITICO: CHECK 2 FALSE (Google ID mudou — evento foi deletado e recriado)
  FAIL_CRITICO: Registro inteiro sumiu do banco (DELETE em vez de UPDATE)
  FAIL: CHECK 1 FALSE (nome não mudou)
```

### Critérios:

```markdown
| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | event_name | = "PlanningTest" | Não mudou ou registro sumiu |
| 2 | Google ID | Mesmo session_event_id_google | Mudou (= deletou e recriou) |
| 3 | Evento existe | active = true, registro no banco | Sumiu ← BUG CONHECIDO |
| 4 | Outros campos | Intactos | Regressão |
```

### Cleanup:

```
DELETE /calendar?event_name=ilike.*PlanningTest*&fk_user=eq.{user_id}
DELETE /calendar?event_name=ilike.*TestRename*&fk_user=eq.{user_id}
```

---

## AÇÃO 1.5 — Teste de exclusão de evento recorrente

**Issue:** CRIT-05
**Arquivo:** `tasks/agenda/04-exclusao-compromissos.md`
**Onde inserir:** Seção Quick, como DEL-Q4 e DEL-Q5

### Teste mastigado — excluir recorrente inteiro:

```markdown
| ID | Operação | Input | Verificação |
| DEL-Q4 | Excluir recorrente (todas) | SETUP: criar "corrida toda terça 7h". TESTE: "cancela todas as corridas" | calendar: -1, registro com is_recurring=true SUMIU |
```

### Algoritmo DEL-Q4:

```
SETUP — CRIAR RECORRENTE
  Enviar: "me lembra toda terça às 7h de correr"
  Pollar + verificar calendar: +1, is_recurring=true
  Salvar: EVENT_ID

PASSO 1 — SNAPSHOT ANTES
  COUNT_ANTES: GET /calendar?select=id&user_id=eq.{user_id}
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}
  Confirmar: is_recurring = true, rrule contém BYDAY=TU

PASSO 2 — ENVIAR: "cancela todas as corridas"

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — WAIT 20s

PASSO 5 — VERIFICAR BANCO
  COUNT_DEPOIS: GET /calendar?select=id&user_id=eq.{user_id}
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}

  PASS: Busca retorna vazio (deletou) E COUNT_DEPOIS = COUNT_ANTES - 1
  FAIL: Registro ainda existe ← BUG CONHECIDO (A-200)
  PARTIAL: active = false mas registro existe (desativou sem deletar)
```

### Teste mastigado — excluir UMA ocorrência:

```markdown
| ID | Operação | Input | Verificação |
| DEL-Q5 | Excluir 1 ocorrência de recorrente | SETUP: criar "yoga toda segunda e quarta 8h". TESTE: "cancela a yoga desta segunda" | calendar: registro EXISTE, exdates CONTÉM data da segunda. Demais ocorrências intactas. |
```

### Algoritmo DEL-Q5:

```
SETUP — CRIAR RECORRENTE
  Enviar: "me lembra toda segunda e quarta às 8h de yoga"
  Pollar + verificar calendar: +1, is_recurring=true
  Salvar: EVENT_ID, rrule

PASSO 1 — SNAPSHOT ANTES
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}
  Salvar: exdates (provavelmente null ou vazio)

PASSO 2 — ENVIAR: "cancela a yoga desta segunda"
  (usar data absoluta da próxima segunda: {PROXIMA_SEGUNDA})

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — WAIT 20s

PASSO 5 — VERIFICAR BANCO
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}

  CHECK 1: Registro AINDA EXISTE (não deletou tudo)
  CHECK 2: is_recurring AINDA = true
  CHECK 3: exdates CONTÉM {PROXIMA_SEGUNDA} (ocorrência excluída)
  CHECK 4: rrule INALTERADA

  PASS: Checks 1-4 todos TRUE
  FAIL: Registro sumiu (deletou tudo quando deveria excluir 1)
  FAIL: exdates vazio (não implementou exclusão parcial) ← PROVÁVEL
  PARTIAL: IA disse que não consegue excluir 1 ocorrência (feature não implementada — documentar)
```

> **NOTA:** Se DEL-Q5 retornar PARTIAL (feature não implementada), registrar como melhoria necessária no N8N, não como bug do teste.

---

## AÇÃO 1.6 — Teste anti-duplicação de recorrentes

**Issue:** CRIT-06
**Arquivo:** `tasks/agenda/06-lembretes-recorrentes.md`
**Onde inserir:** Seção Quick, como REC-Q4

### Teste mastigado:

```markdown
| ID | Operação | Input | Verificação |
| REC-Q4 | Anti-duplicação | "academia toda segunda e sexta 18h" | calendar: EXATAMENTE 1 registro com is_recurring=true. NÃO 2 ou 3. |
```

### Algoritmo:

```
PASSO 1 — SNAPSHOT ANTES
  COUNT_REC_ANTES: GET /calendar?select=id&user_id=eq.{user_id}&is_recurring=eq.true&event_name=ilike.*academia*
  Esperado: 0 (não existe)

PASSO 2 — ENVIAR: "academia toda segunda e sexta às 18h"

PASSO 3 — POLLAR RESPOSTA (max 45s)

PASSO 4 — WAIT 10s (dar tempo pro workflow terminar)

PASSO 5 — VERIFICAR BANCO
  BUSCAR: GET /calendar?user_id=eq.{user_id}&is_recurring=eq.true&event_name=ilike.*academia*&order=created_at.desc
  COUNT_REC = número de registros retornados

  PASS: COUNT_REC = 1 exatamente
  FAIL: COUNT_REC > 1 ← BUG DE DUPLICAÇÃO
  FAIL: COUNT_REC = 0 (não criou)

PASSO 5.1 — SE COUNT_REC = 1, verificar campos:
  is_recurring = true
  rrule contém FREQ=WEEKLY
  rrule contém BYDAY=MO,FR (ou MO e FR)
  event_name coerente com "academia"
```

### Critérios:

```markdown
| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | Quantidade | Exatamente 1 registro | 2+ registros (duplicou) |
| 2 | is_recurring | true | false |
| 3 | rrule | FREQ=WEEKLY com BYDAY contendo MO e FR | Incompleta ou errada |
```

### Cleanup:

```
DELETE /calendar?event_name=ilike.*academia*&user_id=eq.{user_id}
```

### Teste de stress (opcional, Broad):

```markdown
| ID | Operação | Input | Verificação |
| REC-B6 | Anti-duplicação stress | Enviar "corrida toda terça 7h" 3x consecutivas (5s intervalo) | calendar: EXATAMENTE 1 registro. NÃO 3. |
```

---

## AÇÃO 1.7 — Resolver intestabilidade da agenda diária

**Issue:** CRIT-07
**Arquivo:** `tasks/agenda/07-agenda-diaria-automatica.md`
**O que fazer:** Reclassificar como monitoramento passivo + criar teste semi-ativo

### Mudança na task:

Adicionar seção no início:

```markdown
## NOTA DE LIMITAÇÃO

Esta funcionalidade NÃO pode ser testada ativamente pelo auditor porque depende
de Schedule Trigger. Os testes abaixo são de MONITORAMENTO PASSIVO — verificam
se o sistema está funcionando baseado em evidências, não por provocação direta.

Para teste ativo, seria necessário:
1. Endpoint de trigger manual no workflow Lembretes (não existe hoje)
2. OU acesso à API do N8N para ativar workflow manualmente
```

### Teste semi-ativo mastigado (o melhor possível):

```markdown
| ID | Operação | Input | Verificação |
| DIA-Q1 | Verificar execuções recentes | (nenhum input — consulta passiva) | GET /executions?workflowId=b3xKlSunpwvC4Vwh&limit=10 → pelo menos 1 execução nas últimas 2h com status=success |
| DIA-Q2 | Verificar logs de envio | (nenhum input — consulta passiva) | GET /log_total?tipo=eq.lembrete_automatico&order=created_at.desc&limit=5 → pelo menos 1 nas últimas 24h |
```

### Recomendação para tornar testável no futuro:

Adicionar ao workflow `Lembretes Total Assistente` (b3xKlSunpwvC4Vwh):
- Webhook alternativo `/test-trigger-lembrete` que executa a mesma lógica do Schedule Trigger sob demanda
- Guardrail: só aceita se header `X-Test-Mode: true`
- Com isso, o auditor pode provocar o envio e verificar resultado

---

# FASE 2 — ALTOS (implementar após CRITICOS)

---

## AÇÃO 2.1 — Testes de datas relativas

**Issue:** ALTO-01
**Arquivo:** `tasks/financeiro/01-despesas-receitas.md`
**Onde inserir:** Seção 6 (Broad), como FIN-B12 e FIN-B13

### Testes mastigados:

```markdown
| ID | Operação | Input | O que valida | Verificação banco |
| FIN-B12 | Data relativa: ontem | "gastei 50 no mercado ontem" | date_spent = data de ontem (D-1) | spent: +1, date_spent = {ONTEM}T00:00:00 (±24h da data do teste) |
| FIN-B13 | Data relativa: dia da semana | "paguei 200 na segunda" | date_spent = última segunda-feira | spent: +1, date_spent cai em segunda-feira (day_of_week = 1) |
```

### Verificação SQL para FIN-B12:

```
BUSCAR: GET /spent?name_spent=ilike.*mercado*&fk_user=eq.{user_id}&order=created_at.desc&limit=1
EXTRAIR: date_spent do registro
COMPARAR: date_spent deve ser >= {ONTEM}T00:00:00 E < {HOJE}T00:00:00

PASS: date_spent está no dia de ontem
FAIL: date_spent = hoje (ignorou "ontem")
PARTIAL: date_spent = ontem mas horário estranho (ex: 00:00:00 exato)
```

### Cleanup:

```
DELETE /spent?name_spent=ilike.*mercado*&fk_user=eq.{user_id}&created_at=gte.{ONTEM}
```

---

## AÇÃO 2.2 — Promover multi-turno para Broad

**Issue:** ALTO-02
**Arquivo:** `tasks/financeiro/01-despesas-receitas.md`
**O que fazer:** Mover FIN-C2 para seção 6 (Broad) como FIN-B14. Adicionar variação.

### Testes mastigados:

```markdown
| ID | Operação | Input | O que valida | Verificação banco |
| FIN-B14 | Multi-turno (valor) | MSG1: "gastei no uber" → POLLAR → MSG2: "foram 25" | IA recupera contexto e registra gasto Uber R$25 | spent: +1, name=Uber, value=25 |
| FIN-B15 | Multi-turno (tudo) | MSG1: "registra um gasto" → POLLAR → MSG2: "mercado 80 reais" | IA registra gasto Mercado R$80 | spent: +1, name=Mercado, value=80 |
```

### Algoritmo FIN-B14:

```
PASSO 1 — SNAPSHOT ANTES
  COUNT_ANTES, LAST_LOG_ID

PASSO 2A — ENVIAR MSG1: "gastei no uber"

PASSO 3A — POLLAR RESPOSTA MSG1 (max 45s)
  Capturar ai_message → deve pedir valor ("Quanto foi?")
  Salvar: LOG_ID_MSG1

PASSO 2B — ENVIAR MSG2: "foram 25"

PASSO 3B — POLLAR RESPOSTA MSG2 (max 45s, id > LOG_ID_MSG1)
  Capturar ai_message → deve confirmar registro

PASSO 4 — VERIFICAR BANCO
  COUNT_DEPOIS: +1?
  BUSCAR: name_spent ILIKE uber, value_spent = 25

PASS: Registro criado com name=Uber, value=25
FAIL: Não criou (contexto perdido)
FAIL: Criou com valor diferente de 25
PARTIAL: Criou com name genérico (sem "uber") mas value=25
```

---

## AÇÃO 2.3 — Teste de exclusão em massa com confirmação

**Issue:** ALTO-03
**Arquivo:** `tasks/financeiro/01-despesas-receitas.md`
**Onde inserir:** Seção 6 (Broad), substituir FIN-B7

### Teste mastigado revisado:

```markdown
| ID | Operação | Input | O que valida | Verificação banco |
| FIN-B7 | Excluir múltiplos (com confirmação) | SETUP: criar 3 gastos "uber" (R$15, R$20, R$25). TESTE: "remove todos os uber" | IA pede confirmação OU exclui todos. Documentar qual. | spent: se excluiu → -3. Se pediu confirmação → count inalterado. |
```

### Algoritmo:

```
SETUP — CRIAR 3 GASTOS
  Enviar: "gastei 15 de uber" → pollar → verificar +1
  Enviar: "gastei 20 de uber" → pollar → verificar +1
  Enviar: "gastei 25 de uber" → pollar → verificar +1
  Salvar: IDs dos 3 registros

PASSO 1 — SNAPSHOT ANTES
  COUNT_UBER: GET /spent?name_spent=ilike.*uber*&fk_user=eq.{user_id}
  Deve ser >= 3

PASSO 2 — ENVIAR: "remove todos os uber"

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — WAIT 20s

PASSO 5 — VERIFICAR
  CENÁRIO A (excluiu): COUNT_UBER_DEPOIS = COUNT_UBER - 3 → registrar: "exclusão sem confirmação"
  CENÁRIO B (confirmou): COUNT_UBER_DEPOIS = COUNT_UBER → registrar: "pede confirmação antes"
  CENÁRIO C (parcial): COUNT_UBER_DEPOIS = COUNT_UBER - 1 → registrar: "excluiu só 1"

  TODOS os cenários são informativos. O objetivo é DOCUMENTAR o comportamento.
```

---

## AÇÃO 2.4 — Teste de resultado vazio na consulta de agenda

**Issue:** ALTO-04
**Arquivo:** `tasks/agenda/02-consulta-compromissos.md`
**Onde inserir:** Seção Quick, como CONS-Q4

### Teste mastigado:

```markdown
| ID | Input | Verificação |
| CONS-Q4 | "o que tenho dia 31 de dezembro?" | IA responde "nada agendado" ou similar. NÃO inventa evento. calendar: nenhum registro criado. |
```

### Algoritmo:

```
PASSO 1 — VERIFICAR QUE NÃO HÁ EVENTOS
  GET /calendar?user_id=eq.{user_id}&start_event=gte.{ANO}-12-31T00:00:00&start_event=lt.{ANO+1}-01-01T00:00:00
  Se retornar algo → escolher outra data vazia

PASSO 2 — ENVIAR: "o que tenho dia 31 de dezembro?"

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — VERIFICAR
  CHECK 1: ai_message NÃO contém nome de evento inventado
  CHECK 2: ai_message contém "nada" ou "sem compromissos" ou "livre" ou "vazio"
  CHECK 3: COUNT calendar = inalterado (não criou nada)

  PASS: Checks 1-3 TRUE
  FAIL: IA inventou evento (alucinação)
  FAIL: IA criou registro no banco
  PARTIAL: IA disse "não sei" sem ser específica
```

---

## AÇÃO 2.5 — Teste de expansão de recorrentes na consulta

**Issue:** ALTO-05
**Arquivo:** `tasks/agenda/02-consulta-compromissos.md`
**Onde inserir:** Promover para Broad como CONS-B6

### Teste mastigado:

```markdown
| ID | Input | Verificação |
| CONS-B6 | SETUP: criar "corrida toda terça 7h". TESTE: "o que tenho terça?" | IA lista "corrida" na resposta. Mesmo que só exista 1 registro com rrule, a expansão mostra a ocorrência da terça. |
```

### Algoritmo:

```
SETUP — CRIAR RECORRENTE
  Enviar: "me lembra toda terça às 7h de correr"
  Pollar + verificar: is_recurring=true, rrule contém BYDAY=TU

PASSO 1 — IDENTIFICAR PRÓXIMA TERÇA
  Calcular: PROXIMA_TERCA = próxima terça-feira a partir de hoje

PASSO 2 — ENVIAR: "o que tenho na terça?" (ou "o que tenho dia {PROXIMA_TERCA}?")

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — VERIFICAR
  CHECK 1: ai_message CONTÉM "corr" ou "corrida" (evento recorrente apareceu)
  CHECK 2: ai_message CONTÉM horário 7h ou 07:00

  PASS: Checks 1-2 TRUE (expansão funcionou)
  FAIL: IA não listou o evento recorrente (expansão falhou)
```

---

## AÇÃO 2.6 — Teste de edição de evento recorrente

**Issue:** ALTO-06
**Arquivo:** `tasks/agenda/03-modificacao-compromissos.md`
**Onde inserir:** Seção Broad, como MOD-B6

### Teste mastigado:

```markdown
| ID | Operação | Input | Verificação |
| MOD-B6 | Editar recorrente (dia) | SETUP: criar "yoga toda terça 8h". TESTE: "muda a yoga de terça pra quinta" | rrule: BYDAY mudou de TU para TH. is_recurring intacto. next_fire_at recalculado. |
```

### Algoritmo:

```
SETUP — CRIAR RECORRENTE
  Enviar: "yoga toda terça às 8h"
  Pollar + verificar: is_recurring=true, rrule contém BYDAY=TU
  Salvar: EVENT_ID, rrule, next_fire_at

PASSO 1 — SNAPSHOT ANTES
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}
  Salvar todos os campos

PASSO 2 — ENVIAR: "muda a yoga de terça pra quinta"

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — WAIT 20s

PASSO 5 — VERIFICAR
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}

  CHECK 1: rrule contém BYDAY=TH (não mais TU)
  CHECK 2: is_recurring = true (intacto)
  CHECK 3: next_fire_at = próxima quinta-feira (recalculado)
  CHECK 4: event_name intacto
  CHECK 5: Nenhum registro duplicado

  PASS: Checks 1-5 TRUE
  FAIL: rrule não mudou
  FAIL: next_fire_at não recalculou (vai disparar na terça)
  FAIL_CRITICO: Registro duplicou (criou novo + manteve antigo)
```

---

## AÇÃO 2.7 — Teste de confirmação antes de exclusão total

**Issue:** ALTO-07
**Arquivo:** `tasks/agenda/04-exclusao-compromissos.md`
**Onde inserir:** Seção Quick, como DEL-Q6

### Teste mastigado:

```markdown
| ID | Operação | Input | Verificação |
| DEL-Q6 | Exclusão total com confirmação | SETUP: criar 3 eventos. TESTE: "apaga todos os meus compromissos" | IA DEVE pedir confirmação. calendar: COUNT inalterado até user confirmar. |
```

### Algoritmo:

```
SETUP — CRIAR 3 EVENTOS
  Enviar 3 mensagens de criação (reunião, dentista, almoço)
  Verificar: calendar: +3

PASSO 1 — COUNT_ANTES

PASSO 2 — ENVIAR: "apaga todos os meus compromissos"

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — WAIT 20s

PASSO 5 — VERIFICAR
  COUNT_DEPOIS: GET /calendar?select=id&user_id=eq.{user_id}&active=eq.true

  CENÁRIO A (pede confirmação): ai_message contém "certeza" ou "confirma" → COUNT_DEPOIS = COUNT_ANTES → PASS
  CENÁRIO B (deleta tudo): COUNT_DEPOIS < COUNT_ANTES → registrar como RISCO ALTO
  CENÁRIO C (deleta parcial): COUNT_DEPOIS < COUNT_ANTES mas > 0 → registrar comportamento
```

---

## AÇÃO 2.8 — Teste de cancelar lembrete recorrente

**Issue:** ALTO-08
**Arquivo:** `tasks/agenda/06-lembretes-recorrentes.md`
**Onde inserir:** Seção Quick, como REC-Q5

### Teste mastigado:

```markdown
| ID | Operação | Input | Verificação |
| REC-Q5 | Cancelar recorrente | SETUP: criar "remédio toda manhã 9h". TESTE: "cancela o lembrete do remédio" | calendar: registro DELETADO ou active=false. Lembrete NÃO dispara mais. |
```

### Algoritmo:

```
SETUP — CRIAR RECORRENTE
  Enviar: "me lembra todo dia às 9h de tomar remédio"
  Pollar + verificar: is_recurring=true
  Salvar: EVENT_ID

PASSO 1 — SNAPSHOT ANTES
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}
  Confirmar: existe, is_recurring=true, active=true

PASSO 2 — ENVIAR: "cancela o lembrete do remédio"

PASSO 3 — POLLAR RESPOSTA

PASSO 4 — WAIT 20s

PASSO 5 — VERIFICAR
  BUSCAR: GET /calendar?id=eq.{EVENT_ID}

  CENÁRIO A (deletou): Retorna vazio → PASS
  CENÁRIO B (desativou): active = false → PASS
  CENÁRIO C (não fez nada): active = true, registro intacto → FAIL
```

---

## AÇÃO 2.9 — Promover edge cases de next_fire_at para Broad

**Issue:** ALTO-09
**Arquivo:** `tasks/agenda/06-lembretes-recorrentes.md`
**Onde inserir:** Mover testes REC-C3 e REC-C4 do Complete para Broad

### Testes a promover:

```markdown
| ID | Operação | Input | Verificação |
| REC-B7 | Final de mês | "me lembra todo dia 31 de pagar X" | rrule=FREQ=MONTHLY;BYMONTHDAY=31. next_fire_at = próximo mês com dia 31 (pula fevereiro?) |
| REC-B8 | Timezone | "me lembra toda segunda 8h" (verificar se 8h = America/Sao_Paulo) | next_fire_at no timezone correto, não UTC |
```

---

## AÇÃO 2.10 — Teste de timezone na criação de eventos

**Issue:** ALTO-10
**Arquivo:** `tasks/agenda/01-agendamento-proprio.md`
**Onde inserir:** Seção Quick, como AGD-Q5

### Teste mastigado:

```markdown
| ID | Input | Verificação |
| AGD-Q5 | "reunião amanhã às 15h" | calendar: start_event contém 15:00 no horário local (America/Sao_Paulo). timezone = "America/Sao_Paulo". Se gravado em UTC: start_event = 18:00Z (15h + 3h). |
```

### Algoritmo:

```
PASSO 1 — ENVIAR: "reunião amanhã às 15h"

PASSO 2 — POLLAR RESPOSTA

PASSO 3 — VERIFICAR
  BUSCAR: GET /calendar?event_name=ilike.*reuni*&user_id=eq.{user_id}&order=created_at.desc&limit=1

  EXTRAIR: start_event, timezone

  CHECK 1: timezone = "America/Sao_Paulo" (ou equivalente)
  CHECK 2: start_event contém horário equivalente a 15h BRT
    - Se armazena com timezone: 15:00:00-03:00
    - Se armazena em UTC: 18:00:00Z
    - Se armazena sem timezone: 15:00:00 (e timezone field indica BRT)
  CHECK 3: start_event NÃO é 15:00 UTC (senão são 12h no Brasil)

  PASS: 15h no horário brasileiro
  FAIL: 15h UTC (= 12h Brasil)
  FAIL: timezone null ou errado
```

---

## AÇÃO 2.11 — Verificar end_event default em todos os testes de criação

**Issue:** ALTO-11
**Arquivo:** `tasks/agenda/01-agendamento-proprio.md`
**O que fazer:** Adicionar verificação de end_event nos critérios PASS/FAIL existentes

### Mudança nos critérios (seção 4):

Adicionar à tabela de critérios de CRIAR evento:

```markdown
| # | Critério | PASS | FAIL |
| 9 | end_event (sem duração especificada) | end_event = start_event + 30min | end_event NULL ou = start_event ou > start + 24h |
| 10 | end_event (com duração) | end_event = start_event + duração informada | Diferente |
| 11 | end_event > start_event | SEMPRE | end_event <= start_event = DADO CORROMPIDO |
```

> **INSTRUÇÃO PARA LUPA:** Em TODOS os testes de criação de evento (AGD-Q1 a AGD-Q4, AGD-B1 a AGD-B3), verificar OBRIGATORIAMENTE end_event. Não pular este campo.

---

# FASE 3 — MEDIOS (implementar após ALTOS)

---

## AÇÃO 3.1 — Valor negativo

**Arquivo:** `tasks/financeiro/01-despesas-receitas.md`, Broad
```markdown
| FIN-B16 | Valor negativo | "gastei -50 no mercado" | IA recusa OU converte para 50. NÃO aceita negativo. | spent: value_spent > 0 sempre |
```

## AÇÃO 3.2 — Reembolso/devolução

**Arquivo:** `tasks/financeiro/01-despesas-receitas.md`, Broad
```markdown
| FIN-B17 | Reembolso | "me devolveram 50 reais do mercado" | Documentar: cria como entrada? Ignora? | spent: se criou, transaction_type=? |
```

## AÇÃO 3.3 — Cascata de testes (instrução para Lupa)

**Arquivo:** `tasks/financeiro/01-despesas-receitas.md`, Seção 5
Adicionar nota:
```markdown
> **REGRA DE CASCATA:** Se FIN-Q1 retornar FAIL → pular FIN-Q3, FIN-Q4, FIN-Q5.
> Registrar como BLOCKED (não FAIL). Motivo: dependem de dados criados por Q1.
```

## AÇÃO 3.4 — Cleanup obrigatório

**Arquivo:** `tasks/financeiro/01-despesas-receitas.md`
Adicionar seção 11:
```markdown
## 11. CLEANUP OBRIGATÓRIO (após todos os testes)

Executar ao final de CADA bateria:

DELETE /spent?fk_user=eq.{user_id}&created_at=gte.{TIMESTAMP_INICIO_BATERIA}

Verificar: COUNT volta ao valor do SNAPSHOT inicial.
```

## AÇÃO 3.5 — Async com retry em vez de sleep fixo

**Arquivo:** `tasks/agenda/03-modificacao-compromissos.md` e `04-exclusao-compromissos.md`
Substituir:
```
PASSO 4 — ESPERAR ASYNC
  4.1  sleep 20s
```
Por:
```
PASSO 4 — VERIFICAR COM RETRY
  4.1  Loop (max 45s, intervalo 5s):
         Verificar banco
         Se estado mudou → prosseguir
         Se não → sleep 5s, retry
  4.2  Se após 45s não mudou → FAIL (não TIMEOUT)
```

## AÇÃO 3.6 — Consulta por nome de evento

**Arquivo:** `tasks/agenda/02-consulta-compromissos.md`, Broad
```markdown
| CONS-B7 | "quando é a reunião de planning?" | IA retorna evento com nome "planning" (ou similar). Data/hora corretos. |
```

## AÇÃO 3.7 — Evento em data passada

**Arquivo:** `tasks/agenda/01-agendamento-proprio.md`, Broad
```markdown
| AGD-B4 | "marca reunião ontem às 10h" | Documentar: cria no passado? Recusa? Sugere amanhã? |
```

## AÇÃO 3.8 — Editar lembrete recorrente

**Arquivo:** `tasks/agenda/06-lembretes-recorrentes.md`, Broad
```markdown
| REC-B9 | "muda o lembrete de terça pra quinta" | rrule: BYDAY mudou. next_fire_at recalculado. |
```

## AÇÃO 3.9 — Cascade ao excluir evento com lembrete

**Arquivo:** `tasks/agenda/04-exclusao-compromissos.md`, Broad
```markdown
| DEL-B6 | SETUP: criar evento com lembrete. TESTE: "apaga a reunião" | Evento deletado E lembrete associado deletado. Verificar: não sobra registro órfão com due_at/reminder. |
```

## AÇÃO 3.10 — Conflito de horário

**Arquivo:** `tasks/agenda/01-agendamento-proprio.md`, Broad
```markdown
| AGD-B5 | SETUP: criar evento 14h. TESTE: "marca dentista amanhã 14h" | Documentar: avisa conflito? Cria em cima? Sugere outro horário? |
```

---

# FASE 4 — BAIXOS (backlog)

```markdown
| ID | Ação | Arquivo | Nível |
|----|------|---------|-------|
| 4.1 | Moeda estrangeira | financeiro/01, Complete | FIN-C16: "gastei 20 dólares" |
| 4.2 | Concorrência/stress | financeiro/01, Complete | FIN-C17: 2 msgs <1s |
| 4.3 | Evento multi-dia | agenda/01, Complete | AGD-C6: "viagem de sexta a domingo" |
| 4.4 | Volume de resultados | agenda/02, Complete | CONS-C5: dia com 20+ eventos |
```

---

# FASE 5 — TRANSVERSAIS

## AÇÃO 5.1 — Mitigar não-determinismo do LLM

**Arquivo:** `checklists/auditoria-360-checklist.md`
Adicionar seção:

```markdown
## REGRA DE DETERMINISMO

Critérios PASS/FAIL devem usar CONTAINS (não EQUALS) para texto da IA:
- PASS: ai_message CONTÉM "registrado" (não exige emoji exato)
- FAIL: ai_message NÃO contém nenhuma palavra de confirmação

Lista de palavras de confirmação aceitas:
  CREATE: "registrado", "registrei", "anotado", "anotei", "criado", "criei", "adicionado", "adicionei"
  EDIT: "editado", "editei", "alterado", "alterei", "atualizado", "atualizei", "modificado"
  DELETE: "excluído", "excluí", "apagado", "apaguei", "removido", "removi", "deletado", "deletei"
  SEARCH: "encontrei", "achei", "resultado", "total", "soma"
  REFUSE: "não consigo", "não posso", "não é possível", "infelizmente", "desculpe"

Para features críticas, se o resultado alternar entre PASS e FAIL em execuções
consecutivas com mesmo input, registrar como FLAKY e reportar separadamente.
```

## AÇÃO 5.2 — Guia de leitura de execuções N8N

**Arquivo:** Criar `data/n8n-execution-guide.yaml`

```yaml
name: "Guia de leitura de execuções N8N"
version: "1.0.0"

como_buscar_execucao:
  endpoint: "GET /api/v1/executions/{exec_id}"
  header: "X-N8N-API-KEY: {KEY}"

como_encontrar_execucao_do_teste:
  metodo: "Buscar por timestamp"
  query: "GET /executions?limit=5"
  filtro: "startedAt mais próximo do timestamp do POST webhook (±10s)"

campos_importantes:
  status: "success | error | waiting"
  startedAt: "Timestamp de início"
  stoppedAt: "Timestamp de fim"
  data:
    resultData:
      runData:
        "{nome_do_no}":
          - data:
              main:
                - - json: "← AQUI estão os dados de output do nó"

como_ver_branch_escolhida:
  no: "Switch - Branches1"
  path: "data.resultData.runData['Switch - Branches1'][0].data.main"
  interpretar: "O array que NÃO está vazio é a branch que foi ativada"
  exemplo: "main[0] = vazio, main[1] = [{json}] → branch 1 ativada"

como_ver_tools_do_ai_agent:
  no: "AI Agent"
  path: "data.resultData.runData['AI Agent'][0].data.main[0][0].json"
  campo: "output ou response"
  interpretar: "O texto do output contém as tools chamadas em formato de log"

como_ver_erro:
  path: "data.resultData.runData['{no_que_falhou}'][0].error"
  campos: "message, description, stack"
```

---

# RESUMO DE IMPLEMENTAÇÃO

| Fase | Ações | Arquivos afetados | Prioridade |
|------|-------|-------------------|------------|
| **1 — CRITICOS** | 7 ações (1.1 a 1.7) | 01-despesas-receitas, 03-modificacao, 04-exclusao, 06-recorrentes, 07-diaria | IMEDIATA |
| **2 — ALTOS** | 11 ações (2.1 a 2.11) | 01-despesas-receitas, 01-agendamento, 02-consulta, 03-modificacao, 04-exclusao, 06-recorrentes | Após Fase 1 |
| **3 — MEDIOS** | 10 ações (3.1 a 3.10) | Todos os arquivos de tasks + checklist | Após Fase 2 |
| **4 — BAIXOS** | 4 ações | financeiro/01, agenda/01, agenda/02 | Backlog |
| **5 — TRANSVERSAIS** | 2 ações | checklist + novo data file | Paralelo com Fase 1 |

### Novos testes adicionados por esta ação:

| Nível | Quantidade | IDs |
|-------|-----------|-----|
| Quick | +10 | FIN-Q6, MOD-Q4, MOD-Q5, DEL-Q4, DEL-Q5, DEL-Q6, REC-Q4, REC-Q5, CONS-Q4, AGD-Q5 |
| Broad | +15 | FIN-B11 a B17, CONS-B6 a B7, MOD-B6, REC-B6 a B9, AGD-B4, AGD-B5, DEL-B6 |
| Complete | +4 | FIN-C16, FIN-C17, AGD-C6, CONS-C5 |
| **Total** | **+29 testes** | |

---

*Plano gerado por Claude Opus 4.6 — 2026-03-23*
*Próximo passo: implementar Fase 1 (CRITICOS) nos arquivos de tasks*
