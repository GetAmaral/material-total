# BLUEPRINT DE EXECUÇÃO — AUDITOR-360

**Data:** 2026-03-23
**Contexto:** Após análise crítica completa, criação de 63 novos testes, 3 novas funcionalidades, e correção de lacunas.

---

## O QUE TEMOS HOJE

```
27 funcionalidades no squad.yaml
├── 10 testáveis end-to-end (mandam webhook, verificam banco)
├── 6 rasos (só verificam tabela/schema)
├── 5 esqueletos (< 60 linhas, quase nada)
├── 3 problemáticos (user errado, feature removida, intestável)
└── 3 excluídos do squad (arquivos ainda no disco)
```

### As 10 testáveis end-to-end:

| # | Task | Testes | Pronto pra rodar? |
|---|------|--------|-------------------|
| 1 | `financeiro/01-despesas-receitas` | 38 (6Q+17B+15C) | ✅ SIM |
| 2 | `agenda/01-agendamento-proprio` | 19 (5Q+10B+4C) | ✅ SIM |
| 3 | `agenda/02-consulta-compromissos` | 14 (4Q+7B+3C) | ✅ SIM |
| 4 | `agenda/03-modificacao-compromissos` | 16 (5Q+6B+5C) | ✅ SIM |
| 5 | `agenda/04-exclusao-compromissos` | 15 (6Q+7B+2C) | ✅ SIM |
| 6 | `agenda/06-lembretes-recorrentes` | 15 (4Q+8B+3C) | ✅ SIM |
| 7 | `relatorios/01-relatorio-pdf-whatsapp` | 9 (2Q+4B+3C) | ✅ SIM |
| 8 | `bot-whatsapp/07-conversa-generica` | 12 (4Q+5B+3C) | ✅ SIM |
| 9 | `bot-whatsapp/08-tipos-midia` | 11 (3Q+5B+3C) | ✅ (text-based) |
| 10 | `bot-whatsapp/09-multi-intencao` | 8 (2Q+4B+2C) | ✅ SIM |

**Total: 157 testes prontos. 41 Quick, 73 Broad, 43 Complete.**

---

## BLUEPRINT — 5 FASES

---

### FASE 0 — PREPARAÇÃO (antes de rodar qualquer teste)

**Objetivo:** Garantir que o ambiente está limpo e funcional.

| # | Ação | Como | Critério de pronto |
|---|------|------|-------------------|
| 0.1 | Verificar N8N DEV online | `curl http://76.13.172.17:5678/healthz` | HTTP 200 |
| 0.2 | Verificar API Key funciona | `GET /api/v1/workflows?limit=1` com header `X-N8N-API-KEY` | Retorna JSON com workflows |
| 0.3 | Verificar Supabase Principal | `GET /profiles?id=eq.2eb4065b-280c-4a50-8b54-4f9329bda0ff` | Retorna perfil do Luiz Felipe |
| 0.4 | Verificar Supabase AI Messages | `GET /log_users_messages?order=id.desc&limit=1` | Retorna último log |
| 0.5 | Limpar dados de teste antigos | `DELETE /spent?fk_user=eq.{user_id}&name_spent=ilike.*teste*` | Sem registros de teste residuais |
| 0.6 | Snapshot baseline | Contar spent, calendar, log_users_messages do user de teste | Salvar como BASELINE para comparação pós-testes |
| 0.7 | Verificar webhook responde | `POST /webhook/dev-whatsapp` com payload "oi" | HTTP 200 + "Workflow was started" |

**Entregável:** Arquivo `output/logs/PREP-{data}.md` com status de cada check.

---

### FASE 1 — QUICK SMOKE (rodar os 41 testes Quick de todas as 10 funcionalidades)

**Objetivo:** Saber o que funciona e o que está quebrado. Rápido, 20-30min total.

**Ordem de execução:**

```
BLOCO A — Infraestrutura (primeiro, porque se falhar aqui, nada funciona)
  1. bot-whatsapp/07-conversa-generica  → GEN-Q1 a Q4  (4 testes)
     "oi", "o que você faz?", "obrigado", "hahaha"
     Se Q1 FAIL ("oi" não funciona) → PARAR. Sistema inoperante.

BLOCO B — Financeiro (CRUD completo)
  2. financeiro/01-despesas-receitas    → FIN-Q1 a Q6  (6 testes)
     Criar, buscar, editar, excluir, double-tap
     Se Q1 FAIL → skip Q3-Q5 (BLOCKED)

BLOCO C — Agenda (CRUD + recorrentes)
  3. agenda/01-agendamento-proprio      → AGD-Q1 a Q5  (5 testes)
  4. agenda/02-consulta-compromissos    → CON-Q1 a Q4  (4 testes)
  5. agenda/03-modificacao-compromissos → MOD-Q1 a Q5  (5 testes)
  6. agenda/04-exclusao-compromissos    → DEL-Q1 a Q6  (6 testes)
  7. agenda/06-lembretes-recorrentes    → REC-Q1 a Q4  (4 testes)

BLOCO D — Relatórios
  8. relatorios/01-relatorio-pdf        → REL-Q1 a Q2  (2 testes)

BLOCO E — Mídias e multi-intenção
  9. bot-whatsapp/08-tipos-midia        → MID-Q1 a Q3  (3 testes)
  10. bot-whatsapp/09-multi-intencao    → MULTI-Q1 a Q2 (2 testes)
```

**Regras durante a Fase 1:**
- Se GEN-Q1 ("oi") FAIL → PARAR TUDO. Problema é no roteador/webhook.
- Se FIN-Q1 FAIL → skip FIN-Q3 a Q5 como BLOCKED.
- Cada teste Quick gera 1 linha no log.
- Cleanup ao final de cada bloco.

**Entregável:** `output/logs/QUICK-{data}.md` com tabela de resultados + lista de FAIL/BLOCKED.

---

### FASE 2 — DIAGNÓSTICO DOS FAILS (investigar cada Quick que falhou)

**Objetivo:** Para cada FAIL da Fase 1, rodar o protocolo de diagnóstico de 5 camadas e gerar documento de erro.

**Para cada teste FAIL:**

```
1. Identificar camada do erro (Classificador → AI Agent → Tool HTTP → Supabase → Resposta)
2. Coletar evidências:
   - GET /executions/{exec_id} → status, nós executados
   - GET /log_users_messages → ai_message completa
   - GET /spent ou /calendar → estado do banco
3. Classificar causa raiz:
   CLASSIFICACAO_ERRADA | TOOL_NAO_CHAMADA | TOOL_FALHOU |
   BANCO_REJEITOU | RESPOSTA_ERRADA | ASYNC_INCOMPLETO |
   COMPORTAMENTO_NAO_DOCUMENTADO
4. Gerar ERR-{NNN}.md em output/errors/
```

**Entregável:** Um `ERR-{NNN}.md` por teste FAIL, com causa raiz e evidências.

---

### FASE 3 — BROAD (rodar os 73 testes Broad nas funcionalidades que passaram no Quick)

**Objetivo:** Testar edge cases, cenários de stress, e comportamentos não-óbvios.

**Regra:** SÓ rodar Broad numa funcionalidade se o Quick dela teve ≥80% PASS.

**Ordem:**

```
BLOCO A — Financeiro Broad (17 testes)
  FIN-B1 a B17: gírias, multi-item, datas relativas, multi-turno,
  valor negativo, reembolso, exclusão em massa

BLOCO B — Agenda Broad (38 testes)
  AGD-B1 a B10: duração, Google sync, duplicata, passado, conflito, multi-turno
  CON-B1 a B7: multi-turno, range, recorrentes, consulta por nome
  MOD-B1 a B6: relativo, ambíguo, Google, editar recorrente
  DEL-B1 a B7: múltipla, recorrente, cascade, multi-turno
  REC-B1 a B8: duplicata, next_fire_at, timezone, editar recorrente

BLOCO C — Relatórios Broad (4 testes)
  REL-B1 a B4

BLOCO D — Bot WhatsApp Broad (14 testes)
  GEN-B1 a B5
  MID-B1 a B5
  MULTI-B1 a B4
```

**Entregável:** `output/logs/BROAD-{data}.md` + novos `ERR-{NNN}.md` para FAILs.

---

### FASE 4 — RELATÓRIO CONSOLIDADO + DECISÕES

**Objetivo:** Consolidar todos os resultados e apresentar ao PO/time para decisões.

**Entregável:** `output/logs/RELATORIO-CONSOLIDADO-{data}.md` contendo:

```markdown
## 1. Cobertura

| Categoria | Funcionalidades | Testes rodados | Pass | Fail | % |
|-----------|----------------|----------------|------|------|---|

## 2. Bugs confirmados (com evidência)

| ID | Funcionalidade | Severidade | Causa raiz | Onde corrigir |
|----|---------------|-----------|------------|---------------|

## 3. Comportamentos documentados (não eram bug nem feature)

| Input | O que aconteceu | Decisão necessária |
|-------|----------------|-------------------|

## 4. Funcionalidades intestáveis (e por que)

| Funcionalidade | Motivo | O que precisaria pra testar |
|---------------|--------|-----------------------------|

## 5. Recomendações de correção priorizadas

| Prioridade | O que corrigir | Esforço | Impacto |
|-----------|---------------|---------|---------|
```

---

### FASE 5 — CORREÇÕES NO N8N (pós-aprovação)

**Objetivo:** Corrigir os bugs encontrados nos workflows do N8N DEV.

**Só executar após aprovação do PO/time na Fase 4.**

| Bug provável | Onde corrigir | O que fazer |
|-------------|--------------|-------------|
| Double-tap duplica registros | Main (`hLwhn94JSHonwHzl`) | Adicionar deduplicação por `message_id` (wamid) |
| end_event não move junto com start_event | Calendar WebHooks (`sSEBeOFFSOapRfu6`), nó `Update a row1` | Calcular `new_end = new_start + duração_original` |
| Rename deleta do Google | Calendar WebHooks, nó `editar_evento_google3` | Verificar se está chamando DELETE+INSERT em vez de PATCH |
| Exclusão de recorrente falha | Calendar WebHooks, nós `excluir_evento` | Verificar filtro de busca pra is_recurring=true |
| Duplicação ao criar recorrente | Lembretes (`b3xKlSunpwvC4Vwh`) | Race condition entre criação e Schedule Trigger |
| Switch sem default pra tipos de mídia | Main, nó `Switch` | Adicionar output default → resposta "tipo não suportado" |

**Após cada correção:** Re-rodar o teste Quick correspondente pra confirmar fix.

---

## MAPA VISUAL

```
FASE 0          FASE 1          FASE 2          FASE 3          FASE 4          FASE 5
Preparação  →   Quick Smoke  →  Diagnóstico  →  Broad Tests  →  Relatório   →  Correções
                                dos FAILs                       Consolidado     no N8N

7 checks        41 testes       1 ERR por       73 testes       1 documento     1 fix por
ambiente        10 funcs        cada FAIL       edge cases      final           bug

~10 min         ~30 min         ~15 min/FAIL    ~2h total       ~30 min         varia
```

---

## O QUE NÃO VAMOS FAZER (e por que)

| Excluído | Motivo |
|----------|--------|
| Testes Complete (43 testes) | Só rodar se Broad tiver >90% PASS. Senão é desperdício — tem bug demais pra corrigir antes. |
| Audio/Image/Sticker end-to-end | Precisa de MEDIA_ID real da Meta API. Não testável no DEV. |
| Funcionalidades frontend (OAuth, 2FA, gestão conta, checkout, export) | 100% frontend. Auditor não acessa browser. |
| Login OTP end-to-end | User de teste já ativado. Precisaria de user novo. |
| Fluxo Standard | User de teste é Premium. E se não existe mais distinção, a task perde sentido. |
| Agenda diária automática | Depende de Schedule Trigger. Não provocável. Só monitoramento passivo. |
| Correções no N8N sem aprovação | Fase 5 só após PO/time validar Fase 4. |

---

## CHECKLIST PRÉ-EXECUÇÃO

- [ ] N8N DEV online e respondendo
- [ ] API Keys configuradas (N8N_DEV_API_KEY, SUPABASE_PRINCIPAL_SERVICE_KEY, SUPABASE_AI_MESSAGES_SERVICE_KEY)
- [ ] User de teste (Luiz Felipe) com perfil ativo no Supabase
- [ ] Webhook `/dev-whatsapp` respondendo
- [ ] Nenhum dado de teste residual no banco
- [ ] Baseline snapshot salvo

---

*Blueprint gerado por Claude Opus 4.6 — 2026-03-23*
