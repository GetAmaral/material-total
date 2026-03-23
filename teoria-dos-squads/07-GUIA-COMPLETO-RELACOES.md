# Guia Completo — Como Tudo se Relaciona

## Visao Geral do Ecossistema

O Synkra AIOS e um **sistema de orquestracao de agentes IA** onde cada componente tem um papel claro e se conecta aos demais de forma precisa. Este guia mostra como todos os conceitos se relacionam.

---

## O Mapa Completo

```
                         +-------------------+
                         |     USUARIO       |
                         | (prompt/comando)  |
                         +--------+----------+
                                  |
                                  v
                    +-------------+-------------+
                    |        HOOKS              |
                    |  (PreToolUse, etc.)        |
                    |  Interceptam antes/depois  |
                    +-------------+-------------+
                                  |
                                  v
              +-------------------+-------------------+
              |              SYNAPSE ENGINE            |
              |                                        |
              |  L0: Constitution (sempre)             |
              |  L1: Global (sempre)                   |
              |  L2: Agent (se ativo)                  |
              |  L3: Workflow (se ativo)               |
              |  L4: Task (se ativa)                   |
              |  L5: Squad (se ativo)                  |
              |  L6: Keyword (se detectado)            |
              |  L7: Star-Command (se digitado)        |
              |                                        |
              |  Output: <synapse-rules> XML            |
              +-------------------+-------------------+
                                  |
                                  v
              +-------------------+-------------------+
              |              AGENT ATIVO              |
              |                                        |
              |  Persona + Expertise + Autoridade      |
              |  Equipado com SKILLS                   |
              |  Membro de um SQUAD                    |
              +-------------------+-------------------+
                                  |
                                  v
              +-------------------+-------------------+
              |              WORKFLOW                  |
              |                                        |
              |  Orquestra TASKS em fases              |
              |  Controla paralelismo e condicoes      |
              |  Aplica GATES de qualidade             |
              +-------------------+-------------------+
                                  |
                                  v
              +-------------------+-------------------+
              |               TASKS                   |
              |                                        |
              |  Procedimentos passo-a-passo           |
              |  Executados por AGENTS                 |
              |  Dentro de WORKFLOWS                   |
              +-------------------+-------------------+
                                  |
                                  v
                         +--------+----------+
                         |      OUTPUT       |
                         | (.md, .json, etc) |
                         +-------------------+
```

---

## Relacoes Entre Componentes

### 1. SYNAPSE <-> AGENT
**Relacao:** O Synapse alimenta o agente com contexto.

```
Synapse injeta regras do agente (L2)
    --> Agente recebe suas proprias regras de comportamento
    --> Agente sabe quem e, o que pode, o que nao pode
```

**Exemplo:** Quando `@dev` e ativado, o Synapse L2 injeta:
- Personalidade do Dex
- Comandos disponiveis
- Restricoes (nao pode fazer push)
- Expertise (implementacao de codigo)

### 2. SYNAPSE <-> SQUAD
**Relacao:** O Synapse carrega regras de dominio do squad (L5).

```
Squad registra dominio no .synapse/manifest
    --> Synapse L5 escaneia e carrega
    --> Regras de dominio injetadas no contexto
```

**Exemplo:** O squad `testador-n8n` registra keywords como "webhook" e "n8n":
- Quando voce menciona "testar webhook", o Synapse detecta (L6)
- Injeta automaticamente as regras do squad (L5)
- O agente recebe contexto especifico de N8N

### 3. SYNAPSE <-> WORKFLOW
**Relacao:** O Synapse injeta regras da fase atual do workflow (L3).

```
Workflow em execucao (fase "collect-data")
    --> Synapse L3 injeta regras da fase
    --> Agente sabe em qual fase esta
    --> Agente sabe o que ja foi feito e o que falta
```

### 4. SYNAPSE <-> HOOK
**Relacao:** Hooks podem alimentar o Synapse com informacoes.

```
Hook precompact dispara
    --> Captura digest da sessao
    --> Alimenta Memory (Pro)
    --> Synapse usa memoria em DEPLETED/CRITICAL brackets
```

### 5. AGENT <-> SKILL
**Relacao:** Agentes usam skills como ferramentas.

```
@dev precisa gerar testes
    --> Aciona skill "test-generator"
    --> Skill executa a geracao
    --> Resultado volta para @dev
```

**Um agente pode ter multiplas skills. Uma skill pode ser usada por multiplos agentes.**

### 6. AGENT <-> SQUAD
**Relacao:** Agentes pertencem a squads.

```
Squad "auditor-real" contem:
    --> @commander (orquestrador)
    --> @coletor (coletor)
    --> @avaliador (avaliador)
    --> @classificador (classificador)
    --> @deep-agent (investigador)
    --> @safety-officer (validador)
```

**Agentes de squad sao diferentes dos 11 agentes core.** Eles existem apenas dentro do contexto do squad.

### 7. AGENT <-> WORKFLOW
**Relacao:** Workflows designam agentes para cada fase.

```
Workflow "audit-user":
    Fase 1 --> @commander executa "resolve-user"
    Fase 2 --> @coletor executa "collect-messages"
    Fase 3 --> @avaliador + @classificador (paralelo)
    Fase 4 --> @deep-agent (condicional)
    Fase 5 --> @commander consolida
    Fase 6 --> @safety-officer valida
```

### 8. WORKFLOW <-> TASK
**Relacao:** Workflows orquestram tasks.

```
Workflow define a sequencia
    --> Task 1 executa (agente A)
    --> Task 2 executa (agente B)
    --> Task 3 e 4 executam em paralelo
    --> Task 5 executa se condicao atendida
```

**Tasks sao a unidade atomica de execucao. Workflows sao a composicao.**

### 9. SQUAD <-> WORKFLOW
**Relacao:** Squads contem workflows.

```
Squad "deploy-review" contem:
    --> workflow "full-review" (pipeline completo)
    --> Que orquestra 5 agentes em 6 fases
```

### 10. HOOK <-> AGENT
**Relacao:** Hooks podem interceptar acoes de agentes.

```
Hook "beforeAgent" dispara
    --> Verifica se agente tem permissao
    --> Se nao, bloqueia ativacao
    --> Se sim, permite e loga
```

---

## Fluxo Completo de Uma Operacao

Vamos seguir uma operacao real do inicio ao fim:

### Cenario: "Auditar o usuario Luiz Felipe"

```
1. USUARIO digita: "auditar o usuario Luiz Felipe"

2. HOOKS (Pre)
   - PreToolUse verifica permissoes
   - Loga a requisicao no audit trail

3. SYNAPSE processa:
   L0: Carrega Constitution (principios inviolaveis)
   L1: Carrega regras globais
   L2: Carrega regras do @commander (agente orquestrador)
   L5: Detecta squad "auditor-real" e carrega regras de dominio
   L6: Keyword "auditar" + "usuario" ativa dominio de auditoria
   --> Injeta <synapse-rules> com tudo necessario

4. AGENT (@commander) e ativado
   - Recebe contexto completo do Synapse
   - Sabe que e o orquestrador da auditoria
   - Conhece suas autoridades e restricoes

5. WORKFLOW "audit-user" inicia

   Fase 1: @commander
   - Executa TASK "resolve-user"
   - Busca "Luiz Felipe" no Supabase
   - Encontra: ID, telefone, email, plano

   Fase 2: @coletor
   - Synapse L3 atualiza: "Fase 2 - Coleta"
   - Executa TASK "collect-messages"
   - Busca em 3+ fontes (Supabase, N8N logs, OpenAI)
   - Monta timeline cronologica

   Fase 3: PARALELO
   - @avaliador executa TASK "evaluate-interactions"
     Score: 0-100 em 5 dimensoes
   - @classificador executa TASK "classify-problems"
     Ranking por tipo/severidade
   - (Ambos executam ao mesmo tempo)

   Fase 4: CONDICIONAL
   - Se Fase 3 encontrou falhas criticas:
     @deep-agent executa TASK "deep-investigate"
     Investigacao 6 camadas (Swiss Cheese)
   - Se nao: SKIP

   Fase 5: @commander
   - Executa TASK "generate-report"
   - Consolida todos os resultados
   - Gera relatorio .md completo

   Fase 6: @safety-officer
   - Executa TASK "validate-safety-gates"
   - Verifica 7 gates (CVR, HFACS, RCA, SMS, Swiss Cheese, LOSA, Kaizen)
   - Se todos passam: Auditoria COMPLETA
   - Se algum falha: Reprocessa

6. OUTPUT gerado:
   - 2024-03-23-audit-luiz-felipe.md (relatorio)
   - 2024-03-23-timeline-luiz-felipe.json (timeline)
   - 2024-03-23-ranking-luiz-felipe.md (ranking)
   - [opcional] 2024-03-23-deep-XXX.md (deep analysis)

7. HOOKS (Post)
   - PostToolUse loga conclusao
   - Notificacao: "Auditoria concluida!"
```

---

## Hierarquia de Responsabilidades

```
SQUAD (O Time)
  |-- define QUEM faz parte
  |-- define QUAL dominio
  |-- define QUAIS regras
  |
  |-- AGENT (O Especialista)
  |     |-- define QUEM e
  |     |-- define O QUE pode fazer
  |     |-- usa SKILLS como ferramentas
  |
  |-- WORKFLOW (A Orquestracao)
  |     |-- define COMO executar
  |     |-- define QUANDO cada fase
  |     |-- define CONDICOES e GATES
  |     |
  |     |-- TASK (A Acao)
  |           |-- define PASSOS especificos
  |           |-- executada por um AGENT
  |           |-- dentro de uma FASE do workflow
  |
  |-- SYNAPSE (O Cerebro)
  |     |-- alimenta TODOS com contexto
  |     |-- adapta ao bracket
  |     |-- injeta regras de dominio
  |
  |-- HOOK (O Gatilho)
        |-- intercepta EVENTOS
        |-- valida, loga, protege
        |-- conecta componentes
```

---

## Analogias do Mundo Real

### Hospital

| AIOS | Hospital |
|------|----------|
| AIOS Framework | O hospital inteiro |
| Squad | Um departamento (cardiologia, neurologia) |
| Agent | Um medico especialista |
| Skill | Uma tecnica medica (sutura, intubacao) |
| Workflow | Um protocolo medico (atendimento de emergencia) |
| Task | Uma etapa do protocolo (triagem, exame, medicacao) |
| Synapse | O prontuario do paciente (contexto) |
| Hook | O alarme do monitor cardiaco (reacao automatica) |

### Empresa de Software

| AIOS | Empresa |
|------|---------|
| AIOS Framework | A empresa inteira |
| Squad | Um time/squad (backend, mobile, infra) |
| Agent | Um membro do time (dev senior, QA, DevOps) |
| Skill | Uma certificacao/especialidade (AWS, Kubernetes) |
| Workflow | Um processo (sprint, deploy pipeline, incident response) |
| Task | Uma tarefa do Jira/Linear |
| Synapse | O contexto do projeto (docs, ADRs, runbooks) |
| Hook | Automacoes de CI/CD (pre-commit, post-deploy) |

### Orquestra Musical

| AIOS | Orquestra |
|------|-----------|
| AIOS Framework | A orquestra inteira |
| Squad | Uma secao (cordas, metais, percussao) |
| Agent | Um musico (violinista, trompetista) |
| Skill | Uma tecnica (vibrato, staccato) |
| Workflow | A partitura (sequencia de movimentos) |
| Task | Um compasso (trecho musical) |
| Synapse | O maestro (coordena todos) |
| Hook | O metronomo (tempo, sincronia) |

---

## Principios de Arquitetura Escalavel

### 1. Separacao de Responsabilidades
Cada componente faz UMA coisa bem:
- Synapse: contexto
- Agent: execucao
- Skill: capacidade
- Squad: agrupamento
- Workflow: orquestracao
- Hook: automacao

### 2. Composabilidade
Componentes se compoem livremente:
- Um squad pode ter N agentes
- Um agente pode ter N skills
- Um workflow pode ter N tasks
- Um hook pode interceptar N eventos

### 3. Layered Security (Defesa em Profundidade)
Seguranca em multiplas camadas:
- Agente (persona READ-ONLY)
- Config (squad.yaml)
- API (chaves limitadas)
- SSH (whitelist)
- Output (diretorio travado)
- Workflow (gates)
- Runtime (auto-verificacao)

### 4. Context-Aware (Consciencia de Contexto)
O Synapse garante que cada componente receba apenas o contexto relevante:
- Bracket-aware: adapta ao consumo de contexto
- Agent-scoped: regras por agente
- Squad-scoped: regras por dominio
- Workflow-scoped: regras por fase

### 5. CLI-First
Tudo funciona via CLI antes de qualquer UI:
- Reprodutivel
- Automatizavel
- Testavel
- Integravel com CI/CD

---

## Anti-Patterns — O Que NÃO Fazer

| Anti-Pattern | Problema | Solucao |
|-------------|----------|---------|
| Squad monolitico | Dificil de manter, lento | Micro-squads composiveis |
| Agente "faz-tudo" | Perde foco, conflitos de autoridade | Um agente, uma responsabilidade |
| Workflow sem gates | Erros propagam sem verificacao | Gate em cada ponto critico |
| Hook lento | Atrasa toda a execucao | Hooks < 1 segundo |
| Skill duplicada | Manutencao dobrada | Skill compartilhada entre agentes |
| Synapse sem manifest | Squad nao integra com contexto | Sempre criar .synapse/manifest |
| Guardrail unico | Falha catastrofica se romper | Defesa em profundidade (7 camadas) |
| Workflow linear | Lento, nao escala | Paralelismo onde possivel |

---

## Tabela de Decisao — Quando Usar O Que

| Voce precisa de... | Use |
|--------------------|----|
| Contexto inteligente para agentes | **Synapse** |
| Um especialista para executar trabalho | **Agent** |
| Uma capacidade reutilizavel | **Skill** |
| Um time completo para um dominio | **Squad** |
| Orquestrar multiplas etapas | **Workflow** |
| Automatizar reacao a eventos | **Hook** |
| Definir passos de uma acao | **Task** |
| Proteger contra erros | **Gate** (dentro do Workflow) |

---

## Resumo Final

O AIOS e como uma **empresa de consultoria automatizada**:

- **Squads** sao os departamentos
- **Agents** sao os consultores especialistas
- **Skills** sao as certificacoes e ferramentas
- **Workflows** sao os processos e metodologias
- **Tasks** sao as entregas especificas
- **Synapse** e a inteligencia organizacional (sabe tudo sobre cada projeto)
- **Hooks** sao as automacoes de escritorio (alarmes, notificacoes, verificacoes)

Cada peca e **simples sozinha**, mas juntas criam um sistema de **complexidade orquestrada** — onde cada componente amplifica os demais.
