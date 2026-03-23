# SYNAPSE — O Cerebro do AIOS

## O Que E

O **Synapse** e o **motor de contexto inteligente** do Synkra AIOS. Ele e responsavel por decidir **quais informacoes, regras e instrucoes** cada agente recebe em cada momento da conversa.

Pense nele como o **sistema nervoso central** de todo o framework. Sem o Synapse, os agentes seriam genericos — com ele, cada agente recebe exatamente o contexto que precisa para atuar com precisao.

---

## Como Funciona — O Pipeline de 8 Camadas

Toda vez que voce envia um prompt, o Synapse executa um pipeline sequencial de 8 camadas (layers) que **injeta regras contextuais** no prompt antes de ele chegar ao agente:

```
Seu prompt
    |
    v
+--------------------------------------------------+
| L0: Constitution (Principios Inviolaveis)        |  <- SEMPRE ativo
| L1: Global/Context (Regras Universais)           |  <- SEMPRE ativo
| L2: Agent-Scoped (Regras do Agente Ativo)        |  <- Se ha agente ativo
| L3: Workflow-Scoped (Regras do Workflow Ativo)    |  <- Se ha workflow ativo
| L4: Task-Scoped (Contexto da Tarefa Atual)        |  <- Se ha tarefa ativa
| L5: Squad-Scoped (Regras do Squad Instalado)      |  <- Se ha squad ativo
| L6: Keyword-Trigger (Palavras-chave no Prompt)    |  <- Se detectar keywords
| L7: Star-Command (Comandos *modo)                |  <- Se usuario digitar *cmd
+--------------------------------------------------+
    |
    v
<synapse-rules> XML injetado no prompt do agente
```

### Detalhamento de Cada Camada

#### L0 — Constitution (Sempre Ativa)
Carrega os **6 principios inviolaveis** do AIOS:
- CLI First
- Agent Authority
- Story-Driven Development
- No Invention
- Quality First
- Absolute Imports

**Nunca e removida**, mesmo quando o contexto esta quase esgotado.

#### L1 — Global + Context
Regras universais aplicaveis a **qualquer agente** em qualquer situacao. Inclui regras de formatacao, padrao de resposta e comportamento base. Tambem injeta regras especificas do **bracket de contexto** atual.

#### L2 — Agent-Scoped
Quando um agente esta ativo (ex: `@dev`, `@qa`), esta camada carrega as **regras especificas daquele agente** — sua personalidade, comandos, restricoes e expertise.

#### L3 — Workflow-Scoped
Se um workflow esta em execucao (ex: `full-test-cycle`), injeta regras da **fase atual do workflow** — o que ja foi feito, qual a proxima etapa, quais restricoes se aplicam.

#### L4 — Task-Scoped
Quando uma tarefa especifica esta ativa, injeta o **contexto da tarefa** — ID, story associada, tipo de executor, criterios de aceitacao.

#### L5 — Squad-Scoped
Escaneia a pasta `squads/` procurando arquivos `.synapse/manifest`. Cada squad registra suas **regras de dominio** que sao injetadas quando o squad esta ativo. Usa cache de 60 segundos para performance.

#### L6 — Keyword-Trigger (RECALL)
Analisa o texto do seu prompt buscando **palavras-chave de dominio** registradas nos manifests dos squads. Se encontrar, injeta automaticamente as regras daquele dominio — sem voce precisar ativar o squad manualmente.

**Exemplo:** Se o manifest do squad `testador-n8n` registra a keyword "webhook" e voce digita "quero testar o webhook", o Synapse automaticamente injeta as regras do squad.

#### L7 — Star-Command
Detecta comandos com prefixo `*` (ex: `*brief`, `*dev`, `*debug`) e injeta modos especiais de operacao — como modo verboso, modo conciso, modo debug.

---

## Context Brackets — Gestao Inteligente de Contexto

O Synapse monitora **quanto do contexto da conversa ja foi consumido** e ajusta a injecao:

| Bracket | Contexto Restante | Camadas Ativas | Budget de Tokens |
|---------|-------------------|----------------|------------------|
| **FRESH** | 60-100% | L0, L1, L2, L7 | 800 tokens |
| **MODERATE** | 40-60% | L0 a L7 (todas) | 1500 tokens |
| **DEPLETED** | 25-40% | Todas + memory hints | 2000 tokens |
| **CRITICAL** | 0-25% | Todas + memory + alerta de handoff | 2500 tokens |

**Por que isso importa?**
- Em FRESH, o Synapse e enxuto — pouco contexto ja esta consumido, entao injeta apenas o essencial.
- Em MODERATE, injeta o pacote completo.
- Em DEPLETED/CRITICAL, **reforcea** regras criticas e adiciona dicas de memoria para nao perder informacoes importantes antes da compactacao.

---

## O Manifest — Como Registrar Regras de Dominio

Cada squad pode ter um arquivo `.synapse/manifest` que registra suas regras de dominio:

```
# .synapse/manifest
DOMAIN=n8n-testing
KEYWORDS=webhook,workflow,n8n,automacao
RULES_FILE=synapse-rules.md
MERGE_MODE=extend
```

**Modos de merge:**
- `extend` — Adiciona regras ao contexto existente (padrao)
- `override` — Substitui regras do contexto base
- `none` — Opera de forma independente

---

## Comandos do Synapse

| Comando | O que faz |
|---------|-----------|
| `*synapse status` | Mostra bracket atual, camadas ativas, tokens consumidos |
| `*synapse domains` | Lista todos os dominios registrados por squads |
| `*synapse debug` | Mostra exatamente o que esta sendo injetado |
| `*synapse create` | Cria novo manifest de dominio |
| `*synapse add` | Adiciona regra a um dominio existente |
| `*synapse edit` | Edita regras existentes |
| `*synapse toggle` | Ativa/desativa um dominio |

---

## Casos de Uso

### 1. Contexto Automatico por Dominio
Crie um manifest para seu dominio (ex: "fintech") com keywords como "pagamento", "transacao", "PIX". Toda vez que voce mencionar essas palavras, o Synapse injeta automaticamente regras de compliance, padroes de seguranca e boas praticas financeiras.

### 2. Guardrails de Seguranca
Use a L0 (Constitution) para definir regras que NUNCA podem ser violadas — como "nunca acessar producao" ou "nunca deletar dados". Essas regras persistem mesmo quando o contexto esta quase esgotado.

### 3. Troca de Contexto entre Squads
Quando voce troca de um squad para outro, o Synapse automaticamente remove as regras do squad anterior e injeta as do novo. Zero configuracao manual.

### 4. Debug de Comportamento
Se um agente esta se comportando de forma inesperada, use `*synapse debug` para ver exatamente quais regras estao sendo injetadas — pode haver um conflito entre camadas.

### 5. Otimizacao de Conversas Longas
Em conversas longas (bracket DEPLETED/CRITICAL), o Synapse automaticamente prioriza regras criticas e adiciona dicas de memoria, evitando que o agente "esqueca" informacoes importantes.

---

## Area de Criatividade

### Ideias para Explorar

1. **Synapse Multi-Projeto:** Crie manifests diferentes para cada projeto e alterne entre eles sem perder contexto.

2. **Synapse como Filtro de Qualidade:** Registre keywords de anti-patterns (ex: "any", "TODO", "hack") para que o Synapse injete automaticamente regras de refatoracao quando detectar esses termos.

3. **Synapse Educacional:** Crie um manifest com keywords de conceitos tecnicos que injeta explicacoes simplificadas automaticamente — util para onboarding de novos membros.

4. **Synapse de Compliance:** Registre termos regulatorios (LGPD, GDPR, PCI-DSS) que ativam automaticamente checklists de conformidade.

5. **Synapse Contextual por Branch:** Diferentes regras para branches de feature vs hotfix vs release — o L3 (Workflow-Scoped) ja suporta isso nativamente.

6. **Synapse para Documentacao Viva:** Keywords que detectam quando voce esta discutindo arquitetura e injetam automaticamente os ADRs (Architecture Decision Records) relevantes.

---

## Performance

| Metrica | Target |
|---------|--------|
| Pipeline completo | < 70ms |
| Camada individual | < 15ms |
| Cache de squad (TTL) | 60 segundos |
| Cache de agente (TTL) | 5 minutos |

O Synapse e projetado para ser **imperceptivel** — voce nunca sente delay, mas ele esta sempre trabalhando.

---

## Resumo

> O Synapse e o que transforma agentes genericos em especialistas contextuais. Ele garante que cada agente receba **apenas o que precisa, quando precisa, na quantidade certa**. E a diferenca entre um chatbot e um sistema inteligente de orquestracao.
