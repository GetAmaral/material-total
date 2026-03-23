# Metodologia de Auditoria — Mensagens com Múltiplas Intenções

**Funcionalidade:** `bot-whatsapp/09-multi-intencao`
**Versão:** 1.0.0

---

## 1. Mapa do sistema

### O que é

Usuários frequentemente enviam mensagens com MAIS DE UMA intenção na mesma frase:

- "gastei 50 no almoço e marca reunião amanhã 14h"
- "mostra meus gastos do mês e gera o relatório"
- "me lembra de tomar remédio 9h e anota que gastei 15 no café"

O classificador (Escolher Branch) no Fix Conflito v2 escolhe UMA branch por mensagem. Quando há múltiplas intenções, o sistema precisa:
- Processar a primeira e ignorar a segunda? (parcial)
- Processar a mais provável e pedir confirmação da outra? (ideal)
- Tentar processar as duas? (perigoso — pode confundir)
- Falhar/travar? (bug)

### Caminho

```
WhatsApp → Main → Fix Conflito v2
  → Escolher Branch (classificador LLM)
    → Escolhe UMA branch (a "dominante")
    → Processa APENAS essa intenção
    → Segunda intenção: perdida? Ou respondida em texto?
```

### O problema

O classificador é single-output. Ele não foi projetado pra multi-intenção. O comportamento é DESCONHECIDO e NÃO DOCUMENTADO. Este teste serve pra **documentar o que acontece**.

---

## 2. Algoritmo de execução

```
PASSO 1 — SNAPSHOT ANTES
  1.1  Contar spent → SPENT_ANTES
  1.2  Contar calendar → CAL_ANTES
  1.3  Último log_id → LAST_LOG_ID

PASSO 2 — ENVIAR MENSAGEM com múltiplas intenções

PASSO 3 — POLLAR RESPOSTA (max 60s — pode demorar mais que o normal)

PASSO 4 — VERIFICAR O QUE ACONTECEU
  4.1  Contar spent → SPENT_DEPOIS
  4.2  Contar calendar → CAL_DEPOIS
  4.3  Analisar ai_message:
       → Respondeu sobre qual intenção? (financeiro? agenda? ambos?)
       → Mencionou a segunda intenção?
       → Pediu pra mandar separado?

PASSO 5 — CLASSIFICAR COMPORTAMENTO
  CENÁRIO A (ideal): Processou 1ª e pediu pra enviar 2ª separado
  CENÁRIO B (parcial): Processou 1ª, ignorou 2ª sem avisar
  CENÁRIO C (ambicioso): Tentou processar ambas (verificar se ambas corretas)
  CENÁRIO D (confuso): Misturou dados das duas (ex: gastou 50 na reunião)
  CENÁRIO E (falha): Não processou nenhuma

PASSO 6 — REGISTRAR comportamento observado
```

---

## 3. Critérios de PASS/FAIL

| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | Não travou | Resposta existe (qualquer que seja) | Timeout ou crash |
| 2 | Pelo menos 1 intenção processada | Gasto criado OU evento criado | Nenhum dos dois |
| 3 | Dados não corrompidos | Se criou gasto: value/name corretos. Se criou evento: start/name corretos. | Valor do gasto no nome do evento ou vice-versa |
| 4 | User informado | IA menciona que só processou 1 intenção OU processou ambas | Silêncio sobre a 2ª intenção |

> **NOTA:** Este teste é primariamente de DOCUMENTAÇÃO. O objetivo é saber o que acontece, não necessariamente que passe. Todos os cenários (A-E) são informativos.

---

## 4. Protocolo de diagnóstico de erros

```
CAMADA 1 — CLASSIFICADOR: Qual branch foi escolhida? (financeiro? agenda?)
CAMADA 2 — PROMPT: O prompt do branch lida com intenções múltiplas?
CAMADA 3 — AI AGENT: O agent tentou processar a 2ª intenção como tool call?
CAMADA 4 — BANCO: Registros criados correspondem a qual intenção?
CAMADA 5 — RESPOSTA: IA mencionou ambas as intenções?
```

---

## 5. Testes

**🟢 Quick (2 testes):**

| ID | Input | Verificação |
|----|-------|-------------|
| MULTI-Q1 | "gastei 50 no almoço e marca reunião amanhã 14h" | Qual intenção processou? spent +1? calendar +1? Ambos? Nenhum? Documentar. |
| MULTI-Q2 | "mostra meus gastos e gera o relatório" | Duas intenções da mesma categoria (busca + relatório). Qual executou? |

**🟡 Broad (Quick + 4 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| MULTI-B1 | "me lembra amanhã 9h de tomar remédio e anota que gastei 15 no café" | Agenda + financeiro na mesma msg. Qual processou? |
| MULTI-B2 | "cancela a reunião e marca dentista quinta 10h" | Exclusão + criação na mesma msg. Processou as duas? |
| MULTI-B3 | "gastei 30 no uber, 20 no café e 15 no estacionamento" | Três gastos na mesma msg (variação de FIN-B11 multi-item). |
| MULTI-B4 | "o que tenho hoje e quanto gastei esse mês?" | Consulta agenda + consulta financeiro. |

**🔴 Complete (Broad + 2 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| MULTI-C1 | Mensagem do payload `text_long` (5+ intenções) | Stress: msg com muitas intenções. O sistema prioriza alguma? |
| MULTI-C2 | "oi, gastei 50 no almoço" | Saudação + gasto. A saudação atrapalha o registro? Ou ignora e registra? |

---

## 6. Formato do log

```markdown
| ID | Input | Branch escolhida | Intenção 1 processada? | Intenção 2 processada? | Dados corretos? | Cenário (A-E) |
```

---

## 7. Melhorias sugeridas

| O que | Impacto |
|-------|---------|
| Detector de multi-intenção no classificador | Avisar user: "Detectei 2 pedidos. Vou processar o gasto. Manda a reunião separado." |
| Queue de intenções | Processar 1ª, guardar 2ª no Redis, perguntar depois |
| Suporte nativo a multi-intent no AI Agent | Agent processa ambas as intenções com tools separadas |
