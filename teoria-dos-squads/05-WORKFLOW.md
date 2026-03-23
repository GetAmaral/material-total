# WORKFLOW — A Orquestracao

## O Que E

Um **Workflow** e uma **sequencia orquestrada de etapas** que coordena multiplos agentes e tasks para atingir um objetivo complexo. Ele define a **ordem de execucao, paralelismo, condicoes, gates e tratamento de erros**.

**Analogia simples:** Se agentes sao jogadores e tasks sao jogadas, o workflow e a **tatica do time** — define quem joga quando, quais jogadas executar em sequencia, quais em paralelo, e quando parar.

---

## Diferenca Entre Workflow, Task e Skill

| Aspecto | Workflow | Task | Skill |
|---------|----------|------|-------|
| **O que e** | Orquestracao de multiplas etapas | Procedimento unico | Capacidade reutilizavel |
| **Escopo** | Amplo (processo completo) | Medio (uma acao) | Focado (uma capacidade) |
| **Envolve multiplos agentes?** | Sim | Geralmente nao | Nao |
| **Tem fases?** | Sim | Nao (e linear) | Nao |
| **Paralelismo?** | Sim (entre fases) | Nao | Nao |
| **Condicoes?** | Sim (gates, ifs) | Simples | Nao |
| **Exemplo** | "Auditoria completa de usuario" | "Coletar mensagens" | "Analisar seguranca" |

---

## Anatomia de um Workflow

Workflows sao definidos em arquivos YAML dentro da pasta `workflows/` do squad:

```yaml
name: full-audit
description: Auditoria completa de um usuario
version: 1.0.0

# Fases do workflow
phases:
  - name: resolve-user
    agent: commander
    task: resolve-user
    description: "Resolve o usuario por nome ou telefone"
    required: true

  - name: collect-data
    agent: coletor
    task: collect-messages
    description: "Coleta mensagens de 3+ fontes"
    required: true
    depends_on: resolve-user

  - name: analyze
    parallel: true               # <-- PARALELISMO
    steps:
      - agent: avaliador
        task: evaluate-interactions
      - agent: classificador
        task: classify-problems
    depends_on: collect-data

  - name: deep-investigation
    agent: deep-agent
    task: deep-investigate
    condition: "analyze.has_critical_failures"   # <-- CONDICIONAL
    depends_on: analyze

  - name: consolidate
    agent: commander
    task: generate-report
    depends_on: [analyze, deep-investigation]

  - name: validate
    agent: safety-officer
    task: validate-safety-gates
    required: true
    depends_on: consolidate
```

### Conceitos-Chave

#### Fases (Phases)
Blocos de execucao sequenciais. Cada fase pode conter:
- Um unico agente + task
- Multiplos agentes + tasks em paralelo

#### Dependencias (depends_on)
Define qual fase deve terminar antes de outra comecar:
```yaml
depends_on: collect-data          # Uma dependencia
depends_on: [analyze, deep-inv]   # Multiplas dependencias
```

#### Paralelismo (parallel: true)
Permite que multiplos agentes executem simultaneamente:
```yaml
- name: analyze
  parallel: true
  steps:
    - agent: avaliador        # Executa ao mesmo tempo
    - agent: classificador    # que este
```

#### Condicoes (condition)
Fases que so executam se uma condicao for verdadeira:
```yaml
condition: "analyze.has_critical_failures"
```
Fases condicionais sao **puladas** se a condicao nao for atendida.

#### Obrigatoriedade (required)
Fases marcadas como `required: true` **nao podem ser puladas** — se falharem, o workflow falha.

---

## Flow-State — Estado Dinamico

O **Flow-State** e o estado do workflow determinado em **runtime** (tempo de execucao), nao em design time. Ele permite que o workflow se adapte dinamicamente:

```
PENDING     -> Fase nao iniciada
IN_PROGRESS -> Fase em execucao
COMPLETED   -> Fase concluida com sucesso
FAILED      -> Fase falhou
SKIPPED     -> Fase pulada (condicao nao atendida)
BLOCKED     -> Fase bloqueada por dependencia
```

O Synapse L3 (Workflow-Scoped) usa o Flow-State para injetar regras especificas da fase atual do workflow.

---

## Confidence Gate — Decisao por Confianca

Alguns workflows usam **Confidence Gates** — pontos de decisao baseados em um score de confianca:

```yaml
- name: quality-check
  agent: qa
  gate:
    type: confidence
    threshold: 0.8          # Minimo 80% de confianca
    on_below: retry          # Se abaixo: retry, escalate, ou fail
    max_retries: 3
```

**Opcoes quando abaixo do threshold:**
- `retry` — Reexecuta a fase (ate max_retries)
- `escalate` — Passa para agente mais senior
- `fail` — Falha o workflow
- `warn` — Continua mas registra alerta

---

## Execution Profile — Nivel de Autonomia

Workflows podem operar em diferentes niveis de autonomia:

| Perfil | Descricao | Quando Usar |
|--------|-----------|-------------|
| `safe` | Confirma cada etapa com usuario | Primeiras execucoes, ambientes sensiveis |
| `balanced` | Confirma apenas decisoes criticas | Uso diario normal |
| `aggressive` | Executa tudo automaticamente | Ambientes de teste, CI/CD |

```yaml
execution_profile: balanced
```

---

## Workflows dos Seus Squads

### auditor-real: `audit-user`
```
Fase 1: Commander resolve usuario
    |
    v
Fase 2: Coletor busca mensagens (3+ fontes)
    |
    v
Fase 3: [PARALELO] Avaliador + Classificador
    |
    v
Fase 4: [CONDICIONAL] Deep-Agent (se ha falhas criticas)
    |
    v
Fase 5: Commander consolida relatorio
    |
    v
Fase 6: Safety-Officer valida 7 gates
```
**Destaque:** Paralelismo na Fase 3, condicionalidade na Fase 4, safety gates na Fase 6.

### deploy-review: `full-review`
```
Fase 1: Coletar paths (DEV/PROD)
    |
    v
Fase 2: [PARALELO x4] diff-site + diff-n8n + diff-database + diff-docker
    |
    v
Fase 3: [PARALELO x2] Error hunting + Quality testing
    |
    v
Fase 4: Gerar changelog
    |
    v
Fase 5: Apresentar veredito (GO/NO-GO/CONDITIONAL)
```
**Destaque:** Paralelismo massivo nas Fases 2 e 3.

### analisador-n8n: `full-investigation`
```
Fase 1: Investigar erro (SSH + logs)
    |
    v
Fase 2: Mapear fluxos N8N
    |
    v
Fase 3: Mapear edge functions
    |
    v
Fase 4: Mapear banco de dados
    |
    v
Fase 5: Gerar diagnostico
    |
    v
Fase 6: Scan de boas praticas
```
**Destaque:** Pipeline linear completo de investigacao.

### testador-n8n: `full-test-cycle`
```
Fase 1: Preparar ambiente de teste
    |
    v
Fase 2: Simular mensagens WhatsApp
    |
    v
Fase 3: Verificar resultados no Supabase
    |
    v
Fase 4: Comparar DEV vs PROD
    |
    v
Fase 5: Gerar relatorio de testes
```

### squad-pixel: `full-pixel-setup`
```
Fase 1: Analisar site (paginas, jornadas, eventos)
    |
    v
Fase 2: Auditoria LGPD
    |
    v
Fase 3: Arquitetura do Pixel (data layer)
    |
    v
Fase 4: Validacao (testes + consent)
    |
    v
Fase 5: Export (codigo + guia)
```

---

## Casos de Uso

### 1. Pipeline de CI/CD Inteligente
```yaml
name: smart-ci-cd
phases:
  - name: lint-and-type
    parallel: true
    steps:
      - task: run-eslint
      - task: run-typecheck
  - name: test
    task: run-tests
    gate:
      type: confidence
      threshold: 0.95
  - name: security-scan
    task: scan-vulnerabilities
    condition: "test.passed"
  - name: build
    task: build-production
    depends_on: [test, security-scan]
  - name: deploy
    agent: devops
    task: deploy-staging
    execution_profile: safe    # Sempre confirma
```

### 2. Onboarding Automatizado
```yaml
name: client-onboarding
phases:
  - name: welcome
    agent: greeter
    task: send-welcome-kit
  - name: collect-info
    agent: greeter
    task: collect-client-info
  - name: setup
    parallel: true
    steps:
      - agent: trainer
        task: create-training-plan
      - agent: admin
        task: provision-accounts
  - name: training
    agent: trainer
    task: execute-training
    depends_on: setup
  - name: validate
    agent: validator
    task: readiness-check
    gate:
      type: confidence
      threshold: 0.9
      on_below: retry
```

### 3. Resposta a Incidentes
```yaml
name: incident-response
execution_profile: aggressive    # Velocidade e prioridade
phases:
  - name: detect
    agent: monitor
    task: identify-incident
  - name: triage
    parallel: true
    steps:
      - agent: monitor
        task: collect-logs
      - agent: alerter
        task: notify-team
  - name: investigate
    agent: responder
    task: root-cause-analysis
    depends_on: triage
  - name: fix
    agent: responder
    task: apply-fix
    condition: "investigate.fix_identified"
  - name: validate
    agent: monitor
    task: verify-fix
  - name: postmortem
    agent: responder
    task: write-postmortem
    depends_on: validate
```

### 4. Code Review Completo
```yaml
name: full-code-review
phases:
  - name: static-analysis
    parallel: true
    steps:
      - task: lint-check
      - task: type-check
      - task: security-scan
      - task: complexity-analysis
  - name: functional-review
    agent: qa
    task: review-logic
    depends_on: static-analysis
  - name: architecture-review
    agent: architect
    task: review-architecture
    depends_on: static-analysis
  - name: consolidate
    agent: qa
    task: generate-review-report
    depends_on: [functional-review, architecture-review]
```

---

## Area de Criatividade

### Padroes Avancados de Workflow

1. **Workflow com Loops**
   Repete uma fase ate condicao ser atendida:
   ```yaml
   - name: fix-loop
     agent: dev
     task: fix-issue
     loop:
       until: "qa.verdict == PASS"
       max_iterations: 5
   ```

2. **Workflow com Fallback**
   Se um agente falha, tenta outro:
   ```yaml
   - name: analyze
     agent: primary-analyst
     fallback:
       agent: secondary-analyst
       on: failure
   ```

3. **Workflow com Checkpoint**
   Salva estado para retomar depois:
   ```yaml
   - name: heavy-analysis
     task: long-running-analysis
     checkpoint: true    # Salva progresso
     resume_on: restart  # Retoma do checkpoint
   ```

4. **Workflow com Aprovacao Humana**
   Pausa e espera aprovacao do usuario:
   ```yaml
   - name: deploy-prod
     agent: devops
     task: deploy-production
     approval:
       required: true
       approvers: [tech-lead, product-owner]
       timeout: 24h
   ```

5. **Workflow Composto**
   Um workflow que chama outros workflows:
   ```yaml
   - name: full-pipeline
     sub_workflow: code-review     # Executa outro workflow
   - name: deploy
     sub_workflow: deploy-staging
     depends_on: full-pipeline
   ```

---

## Boas Praticas

1. **Nomeie fases com verbos.** `collect-data` e melhor que `data` ou `fase-2`.
2. **Paralelismo onde possivel.** Se duas fases sao independentes, execute em paralelo.
3. **Condicoes claras.** Fases condicionais devem ter condicoes obvias.
4. **Gates em pontos criticos.** Antes de deploy, antes de merge, antes de notificacao.
5. **Tratamento de falhas.** Defina o que acontece quando uma fase falha.
6. **Execution profile adequado.** `safe` para producao, `aggressive` para testes.
7. **Timeouts.** Defina timeout por fase para evitar workflows travados.

---

## Resumo

> Workflows sao a cola que une agentes e tasks em processos completos. Eles definem **quem faz o que, quando, e sob quais condicoes**. A forca de um workflow esta no **paralelismo** (fazer mais em menos tempo), **condicionalidade** (adaptar ao contexto) e **gates** (garantir qualidade antes de avancar).
