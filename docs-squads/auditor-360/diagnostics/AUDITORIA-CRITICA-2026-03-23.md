# AUDITORIA CRITICA DO WORKFLOW DE TESTES — AUDITOR-360

**Data:** 2026-03-23
**Escopo:** Financeiro (despesas-receitas) + Agenda (7 tasks)
**Excluido:** VIP Calendar, Limites por Categoria, Metas Financeiras, Limite Mensal Global, distinção Premium/Standard
**Autor:** Claude Opus 4.6 — análise crítica do workflow de testes

---

## INDICE DE SEVERIDADE

| Severidade | Significado | Ação esperada |
|-----------|-------------|---------------|
| CRITICO | Teste não cobre cenário que pode corromper dados ou causar perda | Corrigir ANTES de rodar próxima auditoria |
| ALTO | Lacuna que deixa comportamento importante sem cobertura | Incluir no próximo ciclo de testes |
| MEDIO | Gap que reduz a confiança nos resultados mas não bloqueia | Planejar para inclusão |
| BAIXO | Melhoria desejável, sem risco imediato | Backlog |

---

# BUGS E LACUNAS — CRITICOS

---

## CRIT-01 — Sem teste de mensagem duplicada (double-tap)

**Categoria:** Financeiro — `01-despesas-receitas`
**Severidade:** CRITICO

**O problema:**
Nenhum teste envia a mesma mensagem duas vezes em sequência rápida. Em produção, isso acontece CONSTANTEMENTE — o user aperta "enviar" duas vezes no WhatsApp, a conexão oscila e reenvia, ou o user repete achando que não foi.

**O que pode acontecer:**
- "gastei 35 no almoço" enviado 2x → 2 registros de R$35 → user tem R$70 de gasto fantasma
- O system não tem deduplicação visível (nenhum nó no workflow verifica `message_id` do WhatsApp)

**Como deveria ser testado:**
```
TESTE: Enviar "gastei 50 no jantar" duas vezes com intervalo de 2 segundos
ESPERADO: Apenas 1 registro no banco OU IA responde "já registrei esse gasto"
VERIFICAÇÃO: COUNT_DEPOIS = COUNT_ANTES + 1 (não +2)
```

**Por que é CRITICO:**
Se não há deduplicação, TODOS os gastos registrados via WhatsApp estão sujeitos a duplicação. Isso corrompe todo o financeiro do user. E não é edge case — é cenário do dia-a-dia.

**Onde corrigir no workflow de testes:**
Adicionar como `FIN-Q6` (nível Quick) — se isso falha, o sistema tem problema estrutural.

**Onde corrigir no workflow N8N (se confirmar bug):**
Workflow `Main - Total Assistente` (hLwhn94JSHonwHzl) — adicionar nó que verifica `message_id` do payload WhatsApp antes de processar. Se `message_id` já foi processado nos últimos 60s, ignorar.

---

## CRIT-02 — Mensagem com múltiplos itens financeiros não testada

**Categoria:** Financeiro — `01-despesas-receitas`
**Severidade:** CRITICO

**O problema:**
Nenhum teste envia mensagem com mais de um gasto. User real faz isso frequentemente:
- "gastei 50 no almoço e 30 no uber"
- "paguei 200 de luz e 150 de internet"
- "comprei café 8 reais e pão 5 reais"

**O que pode acontecer:**
- Sistema registra apenas o primeiro item (perda de dado)
- Sistema registra apenas o último item (perda de dado)
- Sistema soma os dois num único registro (dado corrompido: "almoço R$80")
- Sistema cria 2 registros corretamente (ideal, mas não sabemos)
- Sistema pede confirmação item a item (possível mas não documentado)

**Como deveria ser testado:**
```
TESTE: "gastei 50 no almoço e 30 no uber"
ESPERADO (opção A): 2 registros — Almoço R$50 + Uber R$30
ESPERADO (opção B): IA pede pra registrar um de cada vez
FAIL: Apenas 1 registro com valor errado OU nenhum registro
VERIFICAÇÃO: COUNT_DEPOIS >= COUNT_ANTES + 2 (se opção A)
```

**Por que é CRITICO:**
Sem saber como o sistema se comporta com múltiplos itens, o auditor não pode confiar no COUNT de nenhum teste subsequente. Se o user mandou "gastei 50 e 30" e virou 1 registro de R$80, a reconciliação financeira inteira está errada.

**Onde corrigir no workflow de testes:**
Adicionar como `FIN-B11` (nível Broad) — documentar o comportamento atual primeiro, depois decidir se é bug.

---

## CRIT-03 — BUG CONHECIDO: end_event não atualiza ao mover start_event

**Categoria:** Agenda — `03-modificacao-compromissos`
**Severidade:** CRITICO

**O problema:**
Documentado como bug conhecido no Watson T6: quando o user move a data de início de um evento (ex: "muda a reunião do dia 25 pro dia 27"), o `start_event` atualiza para 27 mas o `end_event` permanece no dia 25.

**Resultado:** O evento termina ANTES de começar. O dado fica corrompido.

```
ANTES:  start_event = 2026-03-25 14:00  |  end_event = 2026-03-25 15:00
DEPOIS: start_event = 2026-03-27 14:00  |  end_event = 2026-03-25 15:00  ← CORROMPIDO
```

**Impacto real:**
- Consultas de "o que tenho hoje?" podem não retornar o evento (end < start quebra filtros)
- Google Calendar pode rejeitar o evento (invalid range)
- Frontend pode crashar ao renderizar evento com duração negativa
- Expansão de recorrentes pode gerar datas impossíveis

**Status atual:**
O documento `03-modificacao-compromissos.md` lista isso como "bug conhecido" mas:
- Não tem severity atribuída
- Não tem ticket/issue vinculado
- Não tem teste de regressão dedicado no nível Quick
- Os testes continuam rodando como se fosse "normal"

**Onde está o bug no N8N:**
Workflow `Calendar WebHooks - Total Assistente` (sSEBeOFFSOapRfu6), nó `editar_evento_google3` ou `Update a row1` — quando recebe apenas `start_event` novo, não recalcula `end_event = new_start + duração_original`.

**Como deveria ser testado:**
```
TESTE: Criar evento 14h-15h no dia 25. Depois: "muda a reunião pro dia 27"
VERIFICAÇÃO OBRIGATÓRIA:
  - start_event = 2026-03-27 14:00 ✓
  - end_event = 2026-03-27 15:00 ✓ (DEVE mover junto)
  - end_event > start_event ✓ (NUNCA pode ser anterior)
```

**Recomendação:** Promover a teste `AGE-Q5` (Quick). Se falhar, classificar como **BLOCKER** — dado corrompido não pode ser tolerado.

---

## CRIT-04 — BUG CONHECIDO: Rename de evento pode DELETAR do Google Calendar

**Categoria:** Agenda — `03-modificacao-compromissos`
**Severidade:** CRITICO

**O problema:**
Documentado na Bateria 04: ao renomear um evento (ex: "muda o nome da reunião pra planning"), a IA responde "editado com sucesso" mas o evento é DELETADO do Google Calendar.

**Sequência observada:**
1. User: "muda o nome da reunião pra planning"
2. IA: "✅ Evento editado!"
3. Banco (Supabase): event_name atualizado corretamente
4. Google Calendar: evento SUMIU

**Impacto real:**
- User confia que editou, mas o evento não existe mais no Google
- Lembretes do Google não vão disparar
- Se o user usa Google Calendar como fonte principal, perdeu o compromisso
- A IA disse que deu certo — confiança quebrada

**Causa provável:**
O nó `editar_evento_google3` no workflow Calendar WebHooks pode estar chamando DELETE + INSERT ao invés de UPDATE quando o campo alterado é o `event_name`. Ou o endpoint da Google Calendar API está recebendo payload malformado que resulta em remoção.

**Como deveria ser testado:**
```
TESTE: Criar evento "Reunião" → Renomear para "Planning"
VERIFICAÇÃO:
  1. Banco: event_name = "Planning" ✓
  2. Google Calendar: evento existe com nome "Planning" ✓
  3. session_event_id_google: mesmo ID de antes ✓ (não recriou)
```

**Recomendação:** Teste de regressão obrigatório no Quick. Se confirmar bug, é **BLOCKER** — operação de edit causa delete silencioso.

---

## CRIT-05 — Exclusão de eventos recorrentes FALHA CONSISTENTEMENTE

**Categoria:** Agenda — `04-exclusao-compromissos`
**Severidade:** CRITICO

**O problema:**
Auditoria A-200 mostrou falhas consistentes: ao pedir pra excluir uma ocorrência de evento recorrente, a IA responde "deletado" mas o evento permanece no banco.

**Cenário:**
```
User: "apaga a corrida de terça"
IA: "✅ Evento excluído!"
Banco: evento recorrente ainda existe, is_recurring = true, mesma rrule
```

**Por que falha:**
Eventos recorrentes têm `is_recurring = true` e uma `rrule` (ex: `FREQ=WEEKLY;BYDAY=TU`). Excluir "a corrida de terça" deveria:
- **Opção A:** Deletar TODAS as ocorrências (excluir o registro inteiro)
- **Opção B:** Excluir apenas UMA ocorrência (adicionar data ao campo `exdates`)

O workflow atual não implementa nenhuma das duas corretamente:
- Não deleta o registro (talvez porque o filtro não encontra match exato)
- Não usa o campo `exdates` para exclusão de ocorrência única
- A IA responde sucesso baseada na intenção, não na execução real

**Como deveria ser testado:**
```
TESTE A (excluir todas): "cancela todas as corridas"
  VERIFICAÇÃO: is_recurring record deletado, COUNT -1

TESTE B (excluir uma): "cancela a corrida desta terça"
  VERIFICAÇÃO: exdates contém a data da terça, demais ocorrências intactas
```

**Onde está o bug no N8N:**
Workflow `Calendar WebHooks` (sSEBeOFFSOapRfu6), nós `excluir_evento` / `Excluir eventos - Webhook` — o filtro de busca provavelmente não matcha eventos recorrentes corretamente, ou o delete tenta por `event_name` exato e o nome expandido é diferente do nome base.

**Recomendação:** Separar em 2 testes Quick:
- `DEL-Q4`: Excluir evento recorrente inteiro
- `DEL-Q5`: Excluir uma ocorrência (quando `exdates` for implementado)

---

## CRIT-06 — BUG CONHECIDO: Criação de recorrente pode duplicar registros

**Categoria:** Agenda — `06-lembretes-recorrentes`
**Severidade:** CRITICO

**O problema:**
Documentado como bug conhecido: ao criar lembrete recorrente, o sistema pode gerar MÚLTIPLOS registros no banco ao invés de apenas 1.

**Cenário:**
```
User: "me lembra toda terça às 10h de tomar remédio"
ESPERADO: 1 registro com is_recurring=true, rrule=FREQ=WEEKLY;BYDAY=TU
OBSERVADO: 2 ou 3 registros com dados similares
```

**Impacto real:**
- Lembrete dispara 2-3x no mesmo horário (spam no WhatsApp do user)
- Ao excluir, user deleta 1 mas os outros continuam disparando
- Consulta de "meus lembretes" mostra duplicatas
- next_fire_at pode divergir entre os duplicados, causando lembretes em horários errados

**Causa provável:**
Workflow `Lembretes Total Assistente` (b3xKlSunpwvC4Vwh) — possível race condition entre o nó que cria o registro e o Schedule Trigger que roda a cada 1 minuto. Se a criação demora e o trigger dispara no meio, pode processar o registro parcial e criar outro.

**Como deveria ser testado:**
```
TESTE: "me lembra toda segunda às 9h de reunião"
VERIFICAÇÃO:
  1. COUNT de registros com event_name LIKE "reunião" AND is_recurring = true
  2. DEVE ser exatamente 1
  3. Se > 1: FAIL — duplicação confirmada
```

**Recomendação:** Adicionar verificação anti-duplicação como `LEM-Q1` (Quick). Rodar 3x consecutivas para provocar a race condition.

---

## CRIT-07 — Agenda diária automática é fundamentalmente intestável

**Categoria:** Agenda — `07-agenda-diaria-automatica`
**Severidade:** CRITICO

**O problema:**
O workflow de agenda diária depende de um Schedule Trigger que roda a cada 1 minuto. O squad auditor-360 NÃO tem como:
- Acionar o trigger manualmente
- Controlar quando ele dispara
- Garantir que o teste captura o disparo correto

**O que os testes fazem hoje:**
Verificam logs passados (`log_total`) e execuções recentes do workflow. Isso é **auditoria passiva** — confirma que "algo rodou", mas não testa se o conteúdo estava correto, se o timing estava certo, ou se todos os users receberam.

**O que falta:**
```
TESTE ATIVO (não existe hoje):
  1. Criar evento para daqui 25 minutos
  2. Esperar o Schedule Trigger capturar (janela de 30min antes do evento)
  3. Verificar se WhatsApp foi enviado com conteúdo correto
  4. Verificar se log_total registrou "lembrete_automatico"

PROBLEMA: Esperar 25+ minutos numa auditoria não é viável.
```

**Alternativas que deveriam ser documentadas:**
1. **API call direta ao workflow** — se existe webhook de teste no Lembretes, chamar direto pra simular trigger
2. **Mock do Schedule Trigger** — ativar o workflow manualmente via N8N API (`POST /workflows/{id}/activate`) e verificar execução
3. **Aceitar como auditoria passiva** — mas então REMOVER dos critérios PASS/FAIL ativos e classificar como "monitoramento"

**Recomendação:** Ou criar mecanismo de trigger manual, ou reclassificar como monitoramento passivo. Não pode contar como "testado" algo que não foi provocado.

---

# BUGS E LACUNAS — ALTOS

---

## ALTO-01 — Datas relativas não cobertas adequadamente

**Categoria:** Financeiro — `01-despesas-receitas`
**Severidade:** ALTO

**O problema:**
Apenas 1 teste (FIN-C14, nível Complete) usa data relativa: "gastos da semana passada". Faltam:

| Frase do user | O que deveria acontecer | Testado? |
|---------------|------------------------|----------|
| "gastei 50 ontem" | date_spent = ontem | NÃO |
| "gasto de segunda-feira" | date_spent = última segunda | NÃO |
| "recebi salário dia 5" | date_spent = dia 5 do mês atual | NÃO |
| "gastei 30 anteontem" | date_spent = 2 dias atrás | NÃO |
| "comprei na black friday" | date_spent = ??? | NÃO |

**Por que é ALTO:**
O campo `date_spent` é usado em filtros de período. Se "gastei 50 ontem" grava com data de HOJE, todas as buscas por período ficam erradas. O user pergunta "quanto gastei ontem?" e o gasto aparece em "hoje".

**Recomendação:**
Adicionar ao nível Broad:
- `FIN-B11`: "gastei 50 ontem no mercado" → verificar date_spent = D-1
- `FIN-B12`: "paguei 200 na segunda" → verificar date_spent = última segunda

---

## ALTO-02 — Multi-turno (FIN-C2) está no nível errado

**Categoria:** Financeiro — `01-despesas-receitas`
**Severidade:** ALTO

**O problema:**
O teste FIN-C2 ("gastei no uber" → "foram 25") está no nível Complete (30-60 min). Mas multi-turno é cenário REAL e frequente — user manda mensagem incompleta e complementa na próxima.

**Cenários comuns não cobertos no Quick/Broad:**
- "gastei no almoço" → "45 reais" (valor na segunda mensagem)
- "registra um gasto" → "pizza 35" (tudo na segunda)
- "sim" / "ok" como confirmação (REGRA DE CONTINUAÇÃO do prompt)

**Risco:**
Se o Redis (contexto de sessão) falhar, multi-turno quebra silenciosamente. O user manda "foram 25" e o sistema não sabe do que se trata → ou ignora, ou cria gasto genérico de R$25.

**Recomendação:**
Mover FIN-C2 para `FIN-B11` (Broad). Adicionar `FIN-B12`: "registra um gasto" → "mercado 80 reais".

---

## ALTO-03 — Exclusão em massa sem teste de confirmação

**Categoria:** Financeiro — `01-despesas-receitas`
**Severidade:** ALTO

**O problema:**
FIN-B7 testa "remove todos de uber" — espera que todos com name LIKE "uber" sejam removidos. Mas NÃO verifica:
1. O sistema pede confirmação antes? ("Encontrei 5 gastos de Uber. Deseja excluir todos?")
2. Se não pede, e o user tem 50 registros de Uber de 6 meses, todos somem sem aviso?

**Risco:**
Exclusão irreversível de dados financeiros sem confirmação = perda de histórico. Se o user diz "remove os uber" pensando nos de hoje, e o sistema apaga todos de sempre, não tem rollback.

**Recomendação:**
- Adicionar teste que verifica se há mensagem de confirmação antes do delete em massa
- Adicionar teste de "escopo": "remove os uber de hoje" vs "remove todos os uber" — o sistema diferencia?

---

## ALTO-04 — Sem teste de resultado vazio na consulta de agenda

**Categoria:** Agenda — `02-consulta-compromissos`
**Severidade:** ALTO

**O problema:**
Nenhum teste pergunta "o que tenho amanhã?" quando NÃO há nada agendado.

**O que pode acontecer:**
- IA responde "Nada agendado para amanhã" (correto)
- IA inventa evento ("Você tem uma reunião às 14h") — alucinação
- IA retorna erro ou timeout (bug)
- IA retorna eventos de OUTRO dia (filtro errado)

**Por que é ALTO:**
Resultado vazio é provavelmente o cenário MAIS COMUM da consulta de agenda. Se a IA alucina eventos que não existem, o user vai pra reunião fantasma.

**Recomendação:**
Adicionar como `CONS-Q4` (Quick):
```
SETUP: Verificar que não há eventos para data X
TESTE: "o que tenho dia X?"
VERIFICAÇÃO: IA responde vazio E não criou nada no banco
```

---

## ALTO-05 — Expansão de recorrentes na consulta só no Complete

**Categoria:** Agenda — `02-consulta-compromissos`
**Severidade:** ALTO

**O problema:**
Se "corrida toda terça" é um evento recorrente, a consulta "o que tenho essa semana?" precisa expandir a rrule e mostrar a terça. Esse teste só existe no nível Complete.

**Risco:**
Se a expansão falha, o user pergunta "o que tenho amanhã?" (terça) e a IA não mostra a corrida — porque ela só existe como rrule, não como registro individual.

**O nó responsável:** `Expandir Recorrentes` (code node) no workflow Calendar WebHooks. Se esse nó tem bug, TODAS as consultas de agenda com recorrentes falham.

**Recomendação:** Mover para Broad. Adicionar:
```
TESTE: Criar evento recorrente (toda terça). Consultar "o que tenho terça?"
VERIFICAÇÃO: IA lista o evento recorrente na resposta
```

---

## ALTO-06 — Sem teste de edição de evento recorrente

**Categoria:** Agenda — `03-modificacao-compromissos`
**Severidade:** ALTO

**O problema:**
Todos os testes de edição usam eventos simples (únicos). Nenhum teste edita evento recorrente:
- "muda a corrida de terça pra quarta" — muda TODAS as ocorrências? Só a próxima?
- "muda o horário da reunião semanal pra 15h" — altera a rrule?

**Risco:**
Se editar recorrente altera apenas o registro base mas não recalcula next_fire_at ou rrule, o lembrete continua disparando no horário antigo.

**Recomendação:**
Adicionar ao Broad:
- `MOD-B6`: "muda a corrida de terça pra quarta" → verificar rrule alterada (BYDAY=TU → BYDAY=WE)
- `MOD-B7`: "muda o lembrete das 10h pra 11h" → verificar next_fire_at recalculado

---

## ALTO-07 — Sem teste de confirmação antes de excluir evento

**Categoria:** Agenda — `04-exclusao-compromissos`
**Severidade:** ALTO

**O problema:**
"apaga todos os meus compromissos" — o sistema executa direto ou pede confirmação? Nenhum teste verifica.

**Risco:**
Se o user manda isso num momento de frustração e o sistema apaga tudo sem confirmar, perde semanas/meses de agenda. Inclusive eventos sincronizados com Google Calendar.

**Recomendação:**
Adicionar ao Quick:
```
TESTE: "apaga todos os meus compromissos"
ESPERADO: IA pede confirmação ("Tem certeza? Você tem X compromissos.")
FAIL: IA deleta tudo sem perguntar
```

---

## ALTO-08 — Sem teste de cancelar lembrete recorrente

**Categoria:** Agenda — `06-lembretes-recorrentes`
**Severidade:** ALTO

**O problema:**
Existem testes para CRIAR lembrete recorrente, mas NENHUM para cancelar.

**Cenários não cobertos:**
- "cancela o lembrete de tomar remédio"
- "para de me lembrar da corrida"
- "remove o lembrete de toda terça"

**Risco:**
Se o user não consegue cancelar, o lembrete dispara PARA SEMPRE. A cada semana, recebe mensagem indesejada. A única saída seria deletar direto no banco — que o user não tem acesso.

**Recomendação:**
Adicionar ao Quick:
```
SETUP: Criar lembrete recorrente
TESTE: "cancela o lembrete de [nome]"
VERIFICAÇÃO: registro deletado OU active = false
```

---

## ALTO-09 — next_fire_at edge cases só no Complete

**Categoria:** Agenda — `06-lembretes-recorrentes`
**Severidade:** ALTO

**O problema:**
Bugs CONHECIDOS de cálculo de next_fire_at em edge cases:
- Final de mês (31 jan → 28 fev?)
- Mudança de horário de verão (timezone shift)
- repeats_until expirado (deveria parar mas continua?)

Esses testes estão todos no nível Complete. Mas são bugs DOCUMENTADOS — já se sabe que falham.

**Recomendação:**
Mover para Broad. Se já sabemos que são bugs, devem ser testes de regressão que rodam sempre.

---

## ALTO-10 — Sem teste de timezone na criação de eventos

**Categoria:** Agenda — `01-agendamento-proprio`
**Severidade:** ALTO

**O problema:**
O campo `timezone` existe na tabela `calendar` mas NENHUM teste verifica:
- "reunião às 15h" grava com qual timezone?
- Se o user está em GMT-3 (Brasília), o start_event é 15:00-03:00 ou 15:00 UTC?
- Se grava em UTC, a consulta converte de volta pra horário local?

**Risco:**
Se timezone está errado, TODOS os eventos ficam com 3h de diferença. "Reunião às 15h" vira 18h ou 12h dependendo da conversão. Lembretes disparam no horário errado.

**Recomendação:**
Adicionar ao Quick:
```
TESTE: "marca reunião amanhã às 15h"
VERIFICAÇÃO: start_event contém 15:00 no timezone do user (America/Sao_Paulo)
```

---

## ALTO-11 — end_event default de 30min não verificado

**Categoria:** Agenda — `01-agendamento-proprio`
**Severidade:** ALTO

**O problema:**
O checklist diz que end_event tem default de 30 minutos após start_event. Mas NENHUM teste Quick ou Broad verifica se esse default é aplicado corretamente.

**Cenário:**
```
User: "marca dentista amanhã às 10h" (não especifica duração)
ESPERADO: start_event = 10:00, end_event = 10:30
MAS E SE: end_event = NULL? Ou end_event = 10:00? Ou end_event = 23:59?
```

**Risco:**
Se end_event é NULL ou igual a start_event, a duração é zero. Google Calendar pode não exibir o evento. Consultas de "estou ocupado das X às Y" falham.

**Recomendação:**
Verificar end_event em TODOS os testes de criação de evento. Adicionar critério PASS/FAIL: `end_event = start_event + 30min (quando duração não especificada)`.

---

# BUGS E LACUNAS — MEDIOS

---

## MED-01 — Valor negativo não testado

**Categoria:** Financeiro — `01-despesas-receitas`

**Cenário:** "gastei -50 no mercado"
**Risco:** value_spent = -50 pode quebrar SUM em consultas. "Quanto gastei esse mês?" retorna valor menor que o real.
**Recomendação:** Adicionar ao Broad. Esperado: IA recusa ou converte para positivo.

---

## MED-02 — Reembolso/devolução não testado

**Categoria:** Financeiro — `01-despesas-receitas`

**Cenário:** "me devolveram 50 reais do mercado", "recebi estorno de 30"
**Risco:** Classificador pode interpretar como receita genérica (transaction_type=entrada), quando deveria ser reembolso vinculado ao gasto original. Ou pior: ignorar.
**Recomendação:** Adicionar ao Broad. Documentar comportamento atual.

---

## MED-03 — Dependência cascata nos testes Quick (FIN)

**Categoria:** Financeiro — `01-despesas-receitas`

**O problema:** FIN-Q1 → Q3 → Q4 → Q5 têm dependência sequencial. Se Q1 falha, Q3/Q4/Q5 vão falhar também — mas por falta de dado, não por bug próprio.
**Risco:** Relatório mostra 4 FAIL quando o bug real é 1.
**Recomendação:** Adicionar instrução: "Se FIN-Q1 FAIL → skip Q3, Q4, Q5 e registrar como BLOCKED (não FAIL)".

---

## MED-04 — Cleanup não sistemático

**Categoria:** Financeiro — `01-despesas-receitas`

**O problema:** Os testes criam gastos reais (Almoço R$35, Freelance R$500, etc.) mas não há CLEANUP documentado. Rodar 10x polui o banco.
**Risco:** Buscas futuras retornam gastos de teste como se fossem reais. "Quanto gastei esse mês?" inclui R$35 do almoço de teste.
**Recomendação:** Adicionar seção CLEANUP obrigatória com DELETE dos registros criados (por id_spent).

---

## MED-05 — Async wait fixo de 20s sem retry

**Categoria:** Agenda — `03-modificacao-compromissos`, `04-exclusao-compromissos`

**O problema:** O documento diz "delay de 5-20s", mas o wait é fixo 20s. Se o sistema demora 25s, o teste marca FAIL por timing.
**Risco:** Falsos negativos. Teste reporta FAIL quando o sistema apenas estava lento.
**Recomendação:** Implementar polling com retry: verificar a cada 5s, máximo 45s. Se após 45s não atualizou, aí sim é FAIL.

---

## MED-06 — Sem teste de consulta de agenda por nome

**Categoria:** Agenda — `02-consulta-compromissos`

**Cenário:** "quando é a reunião com João?"
**O problema:** Todos os testes de consulta usam filtro por data. Nenhum busca por nome do evento.
**Recomendação:** Adicionar ao Broad: "quando é a reunião de planning?" → verificar se retorna evento correto.

---

## MED-07 — Sem teste de evento em data passada

**Categoria:** Agenda — `01-agendamento-proprio`

**Cenário:** "marca reunião ontem às 10h"
**Risco:** Sistema aceita e cria evento no passado? Ou recusa? Comportamento não documentado.
**Recomendação:** Adicionar ao Broad. Documentar comportamento.

---

## MED-08 — Sem teste de editar lembrete recorrente

**Categoria:** Agenda — `06-lembretes-recorrentes`

**Cenário:** "muda o lembrete de terça pra quinta"
**Risco:** Se não funciona, user precisa deletar e recriar. Se deletar também falha (CRIT-05), fica preso.
**Recomendação:** Adicionar ao Broad.

---

## MED-09 — Cascade na exclusão de eventos com lembretes

**Categoria:** Agenda — `04-exclusao-compromissos`

**Cenário:** Evento com lembrete associado é deletado. O lembrete também é deletado?
**Risco:** Lembrete órfão continua disparando para evento que não existe mais. User recebe "Lembrete: Reunião" mas a reunião foi cancelada.
**Recomendação:** Adicionar ao Broad: deletar evento → verificar que lembretes associados foram removidos.

---

## MED-10 — Conflito de horário na agenda não testado

**Categoria:** Agenda — `01-agendamento-proprio`

**Cenário:** Já existe evento às 14h. User: "marca reunião às 14h"
**Risco:** Cria por cima sem avisar → user perde o evento original ou fica confuso.
**Recomendação:** Adicionar ao Broad. Esperado: IA avisa sobre conflito.

---

# BUGS E LACUNAS — BAIXOS

---

## BAIXO-01 — Moeda estrangeira não testada

**Categoria:** Financeiro — `01-despesas-receitas`

**Cenário:** "gastei 20 dólares no jantar"
**Risco baixo:** Provavelmente registra como R$20. Mas se ignora "dólares" e registra, o user pode não perceber.
**Recomendação:** Adicionar ao Complete.

---

## BAIXO-02 — Concorrência não testada

**Categoria:** Financeiro — `01-despesas-receitas`

**Cenário:** Duas mensagens enviadas com <1s de diferença.
**Risco:** Race condition teórica no INSERT. Baixa probabilidade em uso normal.
**Recomendação:** Adicionar ao Complete como stress test.

---

## BAIXO-03 — Evento multi-dia não testado

**Categoria:** Agenda — `01-agendamento-proprio`

**Cenário:** "viagem de sexta a domingo"
**Risco:** Pode criar 1 evento (correto) ou 3 eventos separados (confuso).
**Recomendação:** Adicionar ao Complete. Documentar comportamento.

---

## BAIXO-04 — Volume de resultados na consulta de agenda

**Categoria:** Agenda — `02-consulta-compromissos`

**Cenário:** 20+ eventos no mesmo dia.
**Risco:** IA trunca, omite, ou formata mal. Baixa probabilidade de 20+ eventos/dia.
**Recomendação:** Adicionar ao Complete.

---

# PROBLEMAS TRANSVERSAIS

---

## TRANS-01 — Determinismo do AI Agent

**Afeta:** Toda auditoria que depende de resposta da IA

O AI Agent (GPT-4.1-mini) é não-determinístico. O mesmo input pode gerar respostas diferentes a cada execução. Isso significa:
- Testes que verificam texto exato ("✅ Gasto registrado!") podem falhar por variação de formatação
- Alertas de limite podem ou não aparecer dependendo do "humor" do modelo
- Classificação de branch pode variar em edge cases

**Mitigação recomendada:**
1. Critérios PASS/FAIL devem usar CONTAINS em vez de EQUALS
2. Para features críticas (alertas), lógica deveria ser determinística no workflow (IF/Switch), não delegada ao LLM
3. Rodar cada teste 3x e considerar PASS se 2/3 passam (para eliminar variação do LLM)

---

## TRANS-02 — Protocolo de diagnóstico não define como parsear execuções do N8N

**Afeta:** Todas as categorias

O diagnóstico diz "ver quais tools o AI Agent chamou" e "ver output do Switch", mas NÃO documenta:
- Qual campo do JSON de execução contém essa info
- Como navegar a estrutura aninhada (nó → item → json → data)
- Quais nós específicos procurar por nome

**Recomendação:** Adicionar seção "Como ler execuções N8N" com exemplos de jq/paths para cada verificação.

---

## CONTAGEM FINAL

| Severidade | Quantidade | IDs |
|-----------|-----------|-----|
| CRITICO | 7 | CRIT-01 a CRIT-07 |
| ALTO | 11 | ALTO-01 a ALTO-11 |
| MEDIO | 10 | MED-01 a MED-10 |
| BAIXO | 4 | BAIXO-01 a BAIXO-04 |
| TRANSVERSAL | 2 | TRANS-01, TRANS-02 |
| **TOTAL** | **34** | |

---

*Documento gerado por Claude Opus 4.6 — Auditoria crítica do workflow de testes do squad auditor-360*
*Próximas categorias pendentes: Bot WhatsApp, Autenticação, Pagamentos, Investimentos, Relatórios*
