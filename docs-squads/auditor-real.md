# auditor-real

**Squad de Auditoria de Requisicoes REAIS de Usuarios do Total Assistente**

---

## Resumo

O auditor-real analisa conversas completas de usuarios reais em producao, rastreando cada requisicao do inicio ao fim: **WhatsApp -> N8N -> OpenAI -> Supabase -> Resposta ao usuario**. Ele diagnostica falhas, avalia qualidade, classifica problemas por severidade e sugere correcoes.

**Em uma frase:** Le dados reais de producao (sem modificar nada), avalia cada interacao, e gera relatorio com diagnostico.

---

## Identidade

| Campo | Valor |
|-------|-------|
| Nome | auditor-real |
| Versao | 2.0.0 |
| Criado em | 18/Mar/2026 |
| Modo | **STRICT READ-ONLY** (nunca modifica producao, nunca envia mensagens) |
| Agentes | 6 (Argus, Reaper, Judge, Ranker, Pathfinder, Sentinel) |
| Tasks | 8 |
| Workflows | 2 (audit-user + deep-investigation) |
| Metodologia | Seguranca da Aviacao (Swiss Cheese, HFACS, TEM, Just Culture, SMS, LOSA, Kaizen) |

---

## O que ele FAZ

1. **Resolve o usuario** por nome ou telefone
2. **Coleta TODAS as mensagens** de 6+ fontes (Supabase + N8N + logs)
3. **Cruza fontes** para garantir zero perda de requisicoes (detecta orfas)
4. **Avalia cada interacao** com score 0-100 em 6 dimensoes
5. **Classifica problemas** por tipo, severidade, componente e frequencia
6. **Investiga causas raiz** em falhas criticas (6 camadas, sob demanda)
7. **Valida a propria auditoria** via 7 gates de seguranca da aviacao
8. **Gera relatorio** completo com recomendacoes acionaveis

---

## O que ele NAO faz

- NAO modifica dados de producao (STRICT READ-ONLY)
- NAO envia mensagens para usuarios reais
- NAO executa workflows N8N
- NAO dispara testes (isso e o auditor-360 ou testador-n8n)
- NAO faz deploy (isso e o guardian-deploy)

---

## Agentes (6)

### Argus (commander)
- **Papel:** Orquestrador de auditoria
- **O que faz:** Recebe o nome/telefone, coordena todos os agentes, consolida relatorio final
- **Arquetipo:** Sentinel (Libra)

### Reaper (coletor)
- **Papel:** Coleta e cruzamento de dados
- **O que faz:** Coleta de 6+ fontes, cruza, garante zero perda, constroi timeline
- **Arquetipo:** Harvester (Virgo)
- **Fontes:** phones_whatsapp, log_users_messages, messages, message_log, n8n executions, profiles

### Judge (avaliador)
- **Papel:** Avaliador de qualidade
- **O que faz:** Avalia cada interacao em 6 dimensoes com score ponderado
- **Arquetipo:** Arbiter (Libra)
- **Dimensoes:** Compreensao (25%), Precisao (25%), Persistencia (20%), Completude (15%), Contexto (10%), Tom (5%)

### Ranker (classificador)
- **Papel:** Classificacao e priorizacao
- **O que faz:** Classifica por tipo/severidade/componente/frequencia, gera ranking priorizado
- **Arquetipo:** Strategist (Capricorn)
- **Framework:** HFACS 4 niveis + Just Culture + LOSA patterns

### Pathfinder (deep-agent)
- **Papel:** Analista de causa raiz
- **O que faz:** Investigacao profunda em 6 camadas (convocado sob demanda para falhas criticas)
- **Arquetipo:** Detective (Scorpio)
- **Trigger:** Falha critica, padrao recorrente (3+), ou solicitacao direta via `*deep`

### Sentinel (safety-officer)
- **Papel:** Oficial de seguranca e completude
- **O que faz:** Valida a auditoria inteira contra 7 gates de seguranca da aviacao
- **Arquetipo:** Guardian (Capricorn)
- **Veredictos:** APROVADO / APROVADO COM RESSALVAS / REPROVADO

---

## Workflow Principal (audit-user)

```
Fase 1: Resolver usuario (Commander)
  |
Fase 2: Coletar dados de 6+ fontes (Coletor) → Construir timeline
  |
Fase 3: [PARALELO] Avaliar interacoes (Judge) + Classificar problemas (Ranker)
  |
Fase 4: [CONDICIONAL] Deep Investigation (Pathfinder) — se falhas criticas
  |
Fase 5: Consolidar relatorio final (Commander)
  |
Fase 6: Validar auditoria (Safety Officer) — 7 gates
```

---

## Sistema de Avaliacao

### 6 Dimensoes (Score 0-100)

| Dimensao | Peso | O que avalia |
|----------|------|-------------|
| Compreensao | 25% | Sistema entendeu a intencao do usuario? |
| Precisao | 25% | Resposta/acao foi factualmente correta? |
| Persistencia | 20% | Dados foram salvos corretamente no banco? |
| Completude | 15% | Pedido foi atendido por completo? |
| Contexto | 10% | Sistema considerou historico da conversa? |
| Tom | 5% | Tom adequado ao contexto? |

### Multiplicador por Dominio

| Dominio | Multiplicador |
|---------|--------------|
| Financeiro | x1.5 |
| Onboarding | x1.3 |
| Calendario | x1.2 |
| Lembretes | x1.1 |
| Conversacional | x1.0 |

### Veredictos

| Score | Veredicto | Simbolo |
|-------|-----------|---------|
| >= 80 | OK | ✅ |
| >= 50 | Parcial | ⚠️ |
| < 50 | Falha | ❌ |
| N/A | Sem Resposta | 🔇 |
| Especial | Erro de Contexto | 🔄 |

---

## Classificacao de Problemas

### Tipos (TEM)

| Codigo | Tipo | Exemplo |
|--------|------|---------|
| INT | Erro de Intent | Usuario quis registrar gasto, sistema entendeu consulta |
| DAT | Dado Incorreto | Gasto R$50 registrado como R$500 |
| TMO | Timeout | Sem resposta apos 60s |
| SIL | Falha Silenciosa | Mensagem sem execucao N8N correspondente |
| GEN | Resposta Generica | Bot responde "Como posso ajudar?" para pedido especifico |
| CTX | Perda de Contexto | Sistema pede info que usuario ja forneceu |
| ACT | Acao Errada | Pediu criar evento, sistema registrou gasto |
| PRT | Execucao Parcial | Gasto registrado sem categoria |

### Severidade

| Codigo | Nivel | Acao |
|--------|-------|------|
| S1 | Critico | Correcao imediata |
| S2 | Alto | Correcao 24-48h |
| S3 | Medio | Proxima sprint |
| S4 | Baixo | Backlog |

### Componentes Rastreados

N8N-MAIN, N8N-PREM, N8N-STD, N8N-FIN, N8N-CAL, OAI (OpenAI), SB (Supabase), RDS (Redis), WA (WhatsApp API)

---

## Investigacao Profunda (6 Camadas)

Ativada pelo Pathfinder quando ha falhas criticas:

| Camada | Nome | Pergunta central |
|--------|------|-----------------|
| 1 | Webhook | Mensagem chegou ao N8N? |
| 2 | Routing | State machine roteou corretamente? |
| 3 | Extraction | Texto/audio extraido e normalizado ok? |
| 4 | LLM | Prompt adequado? Tool correta? Alucinacao? |
| 5 | Action | Query/insert executou? RLS? Constraint? |
| 6 | Response | Resposta formatada e enviada via WhatsApp? |

---

## Validacao da Auditoria (7 Gates)

O Safety Officer (Sentinel) valida a auditoria inteira:

| Gate | Nome | Metodologia | Severidade |
|------|------|-------------|-----------|
| 1 | CVR | Resposta da IA capturada em todos os testes? | BLOQUEIO |
| 2 | HFACS | Cada erro classificado com S1-S4? | BLOQUEIO |
| 3 | RCA | Cada FAIL tem analise 5-Why? | BLOQUEIO |
| 4 | SMS | Medidas corretivas e preventivas? | AVISO |
| 5 | Swiss Cheese | Trace de 6 camadas preenchido? | BLOQUEIO/AVISO |
| 6 | LOSA | Padroes entre erros identificados? | AVISO |
| 7 | Kaizen | Recomendacoes e aprendizados? | AVISO |

---

## Acessos

| Recurso | Acesso | Endereco |
|---------|--------|----------|
| Supabase (producao) | SELECT apenas (service_role) | `ldbdtakddxznfridsarn.supabase.co` |
| N8N (producao) | read-only via SSH + docker | `188.245.190.178` |
| Docker logs | read-only | `docker logs` (sem stop/restart) |
| GitHub | read-only | `LuizFelipeDosSantos/total-assistente` |
| Qualquer escrita | **BLOQUEADO** | — |

---

## Comandos Principais

```
*auditar "Luiz Felipe"                     # Auditoria completa por nome
*auditar "554391936205"                    # Auditoria por telefone
*auditar "Luiz" --ultimas 10              # Limitar ultimas N interacoes
*auditar "Luiz" --periodo "2026-03-01:2026-03-18"  # Filtrar por periodo
*deep "evt_12"                             # Investigacao profunda em interacao
*resumo "Luiz"                             # Relatorio resumido
*ranking "Luiz"                            # Ranking de problemas
*timeline "Luiz"                           # Timeline cronologica
*buscar-usuario "Luiz"                     # Buscar sem auditar
```

---

## Output

```
squads/auditor-real/output/
├── {date}-audit-{user}.md          # Relatorio completo
├── {date}-timeline-{user}.json     # Timeline em JSON
├── {date}-ranking-{user}.md        # Ranking de problemas
├── {date}-deep-{event_id}.md       # Deep analysis (quando aplicavel)
└── {date}-safety-validation-{user}.md  # Validacao dos 7 gates
```

---

## Diferenca entre auditor-360 e auditor-real

| Aspecto | auditor-360 | auditor-real |
|---------|-------------|-------------|
| **Ambiente** | DEV | PRODUCAO |
| **Modo** | Read-Write (DEV) | Strict Read-Only |
| **O que analisa** | Features (28 funcionalidades) | Usuarios reais (conversas completas) |
| **Como testa** | Dispara requisicoes via webhook | Le dados historicos |
| **Foco** | "A feature funciona?" | "O usuario foi bem atendido?" |
| **Modifica dados?** | Sim (dados de teste no DEV) | NUNCA |
| **Envia mensagens?** | Sim (webhook DEV) | NUNCA |
| **Quando usar** | Antes do deploy, validar features | Apos interacao real, auditar qualidade |

---

*Squad auditor-real v2.0.0 — STRICT READ-ONLY | 6 agentes | Metodologia de Seguranca da Aviacao*
