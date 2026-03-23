# SQUAD — O Time Modular

## O Que E

Um **Squad** e um **pacote modular completo de agentes, tasks, workflows, dados e configuracoes** focado em um dominio especifico. E a unidade de extensibilidade do AIOS — o que permite usar o framework para **qualquer area**, nao apenas desenvolvimento de software.

**Analogia simples:** Se o AIOS e um hospital, um squad e um **departamento** — com seus proprios medicos (agentes), procedimentos (tasks), protocolos (workflows) e equipamentos (tools/data).

---

## Por Que Squads Existem

O AIOS core vem com 11 agentes focados em desenvolvimento de software. Mas e se voce quer:
- Auditar sistemas em producao?
- Testar workflows de N8N?
- Analisar compliance com LGPD?
- Gerenciar conteudo de marketing?
- Fazer onboarding de clientes?

**Squads permitem criar times especializados para qualquer dominio, mantendo toda a infraestrutura do AIOS** (Synapse, config, hooks, permissoes).

---

## Estrutura de um Squad

```
meu-squad/
|
|-- squad.yaml              # MANIFESTO (obrigatorio)
|                             Nome, versao, descricao, agentes, config
|
|-- .synapse/               # INTEGRACAO COM SYNAPSE
|   |-- manifest              Regras de dominio (KEY=VALUE)
|   |-- synapse-rules.md      Regras injetaveis
|
|-- agents/                 # AGENTES DO SQUAD
|   |-- agente-1.md           Definicao completa do agente
|   |-- agente-2.md           Outro agente
|
|-- tasks/                  # TAREFAS EXECUTAVEIS
|   |-- task-1.md             Procedimento passo-a-passo
|   |-- task-2.md             Outra tarefa
|
|-- workflows/              # ORQUESTRACOES
|   |-- workflow-1.yaml       Sequencia de tasks com agentes
|
|-- data/                   # BASE DE CONHECIMENTO
|   |-- knowledge-base.md     Dados especificos do dominio
|   |-- config.yaml           Configuracoes do dominio
|
|-- templates/              # TEMPLATES DE OUTPUT
|   |-- relatorio.md          Modelo de relatorio
|
|-- checklists/             # LISTAS DE VERIFICACAO
|   |-- checklist-1.md        Verificacoes obrigatorias
|
|-- output/                 # RESULTADOS GERADOS
|   |-- (arquivos gerados pelo squad)
|
|-- README.md               # DOCUMENTACAO DO SQUAD
```

---

## O Manifesto — squad.yaml

O `squad.yaml` e o coracao do squad. Ele define:

```yaml
name: meu-squad
version: 1.0.0
description: O que este squad faz
author: Seu Nome

# Modo de operacao
mode: dev-write-prod-block    # ou strict-read-only, read-test, full-access

# Prefixo de comandos
slash_prefix: meusquad        # Ativa comandos com /meusquad:*

# Agentes do squad
agents:
  - agente-1
  - agente-2

# Tasks disponiveis
tasks:
  - task-1
  - task-2

# Workflows
workflows:
  - workflow-1

# Dependencias (outros squads)
dependencies:
  - outro-squad

# Configuracoes de acesso
config:
  environments:
    dev:
      url: http://dev.exemplo.com
      access: read-write
    prod:
      url: http://prod.exemplo.com
      access: read-only       # NUNCA write em prod

# Guardrails
guardrails:
  - nivel: agent
    regra: "READ-ONLY supremacy"
  - nivel: api
    regra: "SELECT only em producao"
```

---

## Modos de Operacao

| Modo | Descricao | Quando Usar |
|------|-----------|-------------|
| `strict-read-only` | Apenas leitura, nenhuma escrita | Auditoria de producao |
| `read-test` | Le producao, testa em DEV | Testes que validam prod |
| `dev-write-prod-block` | Escreve em DEV, bloqueia prod | Testes e desenvolvimento |
| `full-access` | Leitura e escrita em tudo | Ambientes controlados |

---

## Seus 7 Squads — Analise Detalhada

### 1. analisador-n8n
| Aspecto | Detalhe |
|---------|---------|
| **Proposito** | Diagnostico READ-ONLY de producao |
| **Modo** | strict-read-only |
| **Agentes** | 1 (Sherlock — investigador) |
| **Tasks** | 7 (investigate-error, map-flows, scan-best-practices...) |
| **Workflows** | 2 (full-investigation, system-mapping) |
| **Acesso** | SSH read-only para VPS de producao |
| **Destaque** | Nunca modifica nada — apenas observa e documenta |

### 2. testador-n8n
| Aspecto | Detalhe |
|---------|---------|
| **Proposito** | Testar workflows N8N no DEV |
| **Modo** | dev-write-prod-block |
| **Agentes** | 1 (Watson — experimentador) |
| **Tasks** | Simulacao de WhatsApp, testes de webhook, comparacao DEV vs PROD |
| **Workflows** | 2 (full-test-cycle, regression-test) |
| **Acesso** | DEV N8N (write), PROD Supabase (read-only) |
| **Destaque** | Depende do analisador-n8n para knowledge base |

### 3. auditor-360
| Aspecto | Detalhe |
|---------|---------|
| **Proposito** | Auditoria end-to-end de 28 features |
| **Modo** | dev-write-prod-block |
| **Agentes** | 2 (auditor, investigador) |
| **Categorias** | Financeiro(4), Agenda(7), Bot(6), Auth(5), Pagamentos(3), Invest(1), Relatorios(2) |
| **Acesso** | DEV N8N + Supabase (read-write para dados de teste) |
| **Destaque** | Cobertura mais ampla de features |

### 4. auditor-real
| Aspecto | Detalhe |
|---------|---------|
| **Proposito** | Auditoria de requisicoes REAIS de usuarios |
| **Modo** | strict-read-only |
| **Agentes** | 6 (Commander, Coletor, Avaliador, Classificador, Deep-Agent, Safety-Officer) |
| **Metodologia** | Baseada em seguranca de aviacao (Swiss Cheese, HFACS, TEM, LOSA, Kaizen) |
| **Avaliacao** | 5 dimensoes com pesos (Compreensao 25%, Acuracia 25%, Completude 15%, Persistencia 20%, Contexto 10%, Tom 5%) |
| **Safety Gates** | 7 gates (CVR, HFACS, RCA, SMS, Swiss Cheese, LOSA, Kaizen) |
| **Destaque** | O mais sofisticado dos squads — pipeline multi-agente com paralelismo |

### 5. deploy-review
| Aspecto | Detalhe |
|---------|---------|
| **Proposito** | Analise pre-deploy comparando DEV vs PROD |
| **Modo** | strict-read-only |
| **Agentes** | 5 (commander, diff-analyst, error-hunter, quality-tester, changelog-writer) |
| **Workflow** | Diff paralelo (site, N8N, DB, Docker) -> Analise -> Changelog -> Veredito |
| **Vereditos** | GO / NO-GO / CONDITIONAL |
| **Destaque** | Paralelismo real na fase de diff |

### 6. qa-ux-inspector
| Aspecto | Detalhe |
|---------|---------|
| **Proposito** | Testar 29 features via webhook DEV |
| **Modo** | read-test |
| **Agentes** | 1 (inspector) |
| **Regra critica** | NUNCA usar /webhook/ (prod) — SEMPRE /webhook-test/ (dev) |
| **Verificacao** | Webhook -> Supabase -> Google Calendar |
| **Destaque** | Payloads com prefixo TEST_ para dados testaveis |

### 7. squad-pixel
| Aspecto | Detalhe |
|---------|---------|
| **Proposito** | Facebook Pixel com compliance LGPD |
| **Modo** | strict-read-only (site/DB), write (squad-pixel/) |
| **Agentes** | 5 (pixel-lead, pixel-analyst, pixel-architect, pixel-compliance, pixel-qa) |
| **Workflow** | Analise site -> Auditoria LGPD -> Arquitetura pixel -> Validacao -> Export |
| **Output** | Dual-file: codigo + passo-a-passo |
| **Destaque** | Compliance como first-class citizen |

---

## Integracao com o Synapse (L5)

Cada squad pode registrar regras de dominio via `.synapse/manifest`:

```
DOMAIN=n8n-testing
KEYWORDS=webhook,workflow,n8n
RULES_FILE=synapse-rules.md
MERGE_MODE=extend
```

Quando o squad esta ativo, o Synapse L5:
1. Escaneia `squads/` buscando manifests
2. Carrega as regras do squad ativo
3. Aplica namespace: `{SQUAD_NAME}_{DOMAIN_KEY}`
4. Cacheia por 60 segundos
5. Aplica merge mode (extend/override/none)

---

## Niveis de Distribuicao

| Nivel | Local | Descricao |
|-------|-------|-----------|
| **Level 1: LOCAL** | `./squads/` | Privado, dentro do projeto |
| **Level 2: AIOS-SQUADS** | GitHub SynkraAI | Publico, compartilhado |
| **Level 3: SYNKRA API** | api.synkra.dev | Marketplace |

---

## Comandos de Gerenciamento

| Comando | O que faz |
|---------|-----------|
| `*create-squad` | Cria novo squad interativamente |
| `*validate-squad` | Valida estrutura e manifesto |
| `*list-squads` | Lista squads instalados |
| `*download-squad` | Baixa squad do marketplace |
| `*design-squad` | Planeja estrutura de novo squad |
| `*publish-squad` | Publica no marketplace |

---

## Casos de Uso

### 1. Squad de Monitoramento de Infraestrutura
```
squad-infra-monitor/
  agents/
    monitor.md          — Observa metricas e logs
    alerter.md          — Gera alertas e notificacoes
    responder.md        — Sugere acoes de resposta
  tasks/
    check-health.md     — Verifica saude dos servicos
    analyze-logs.md     — Analisa logs em busca de anomalias
    generate-report.md  — Gera relatorio de status
  workflows/
    daily-check.yaml    — Verificacao diaria automatica
    incident-response.yaml — Resposta a incidentes
```

### 2. Squad de Conteudo/Marketing
```
squad-content/
  agents/
    strategist.md       — Define estrategia de conteudo
    writer.md           — Escreve conteudo
    seo-analyst.md      — Analisa e otimiza SEO
    social-manager.md   — Gerencia redes sociais
  tasks/
    create-post.md      — Cria post para blog/rede social
    seo-audit.md        — Auditoria de SEO
    calendar.md         — Planejamento de calendario editorial
  workflows/
    content-pipeline.yaml — Pipeline completo de criacao
```

### 3. Squad de Onboarding de Clientes
```
squad-onboarding/
  agents/
    greeter.md          — Primeiro contato, boas-vindas
    trainer.md          — Treinamento e tutoriais
    validator.md        — Valida que cliente esta pronto
  tasks/
    welcome-kit.md      — Gera kit de boas-vindas
    training-plan.md    — Cria plano de treinamento personalizado
    readiness-check.md  — Verifica se onboarding foi completo
  workflows/
    full-onboarding.yaml — Ciclo completo de onboarding
```

### 4. Squad de Analise Financeira
```
squad-finance/
  agents/
    analyst.md          — Analisa dados financeiros
    forecaster.md       — Projeta cenarios futuros
    compliance.md       — Verifica conformidade regulatoria
  tasks/
    monthly-report.md   — Relatorio mensal
    forecast.md         — Previsao trimestral
    audit-trail.md      — Trilha de auditoria
  workflows/
    monthly-cycle.yaml  — Ciclo mensal completo
```

### 5. Squad de QA para APIs
```
squad-api-qa/
  agents/
    tester.md           — Testa endpoints
    load-tester.md      — Testes de carga
    contract-validator.md — Valida contratos de API
  tasks/
    test-endpoint.md    — Testa um endpoint especifico
    run-collection.md   — Executa colecao de testes
    compare-versions.md — Compara versoes de API
  workflows/
    full-api-audit.yaml — Auditoria completa de API
```

---

## Area de Criatividade

### Padroes de Squad Avancados

1. **Squad com Heranca (Dependencies)**
   Um squad que herda knowledge base de outro:
   ```yaml
   dependencies:
     - analisador-n8n      # Herda conhecimento de producao
   ```
   Seu `testador-n8n` ja faz isso.

2. **Squad Multi-Ambiente**
   Um unico squad que opera em DEV, STAGING e PROD com regras diferentes por ambiente:
   ```yaml
   environments:
     dev: { access: full, guardrails: minimal }
     staging: { access: read-write, guardrails: moderate }
     prod: { access: read-only, guardrails: maximum }
   ```

3. **Squad Composavel (Micro-Squads)**
   Em vez de um mega-squad, crie micro-squads que se compoem:
   ```
   squad-supabase (operacoes de banco)
     + squad-n8n (operacoes de workflow)
     + squad-whatsapp (operacoes de mensagem)
     = Pipeline completo de teste
   ```

4. **Squad com Evolucao (Versioning)**
   Squads que evoluem suas regras baseado em aprendizados:
   ```yaml
   version: 2.0.0
   changelog:
     - "2.0: Adicionado deep-agent para falhas criticas"
     - "1.5: Novo gate de seguranca Swiss Cheese"
     - "1.0: Versao inicial"
   ```

5. **Squad Self-Healing**
   Squad que detecta e corrige problemas na propria execucao:
   - Se um agente falha, tenta com agente alternativo
   - Se um workflow trava, executa recovery automatico
   - Se output esta incompleto, reexecuta etapas faltantes

---

## Guardrails — Defesa em Profundidade

O padrao do `auditor-real` (7 camadas) e o gold standard:

```
Camada 1: AGENTE     — Persona inclui "READ-ONLY supremacy"
Camada 2: CONFIG     — squad.yaml define acesso permitido
Camada 3: API        — Chaves limitadas (anon vs service_role)
Camada 4: SSH        — Whitelist de comandos (docker logs sim, docker stop nao)
Camada 5: OUTPUT     — Diretorio travado (squad/output/ apenas)
Camada 6: WORKFLOW   — Gates que bloqueiam avanco se condicao falhar
Camada 7: RUNTIME    — Checklist de auto-verificacao antes de cada operacao
```

**Regra de ouro:** Cada camada assume que as outras falharam. Se a Camada 1 nao impedir, a Camada 2 impede. Se a 2 falhar, a 3 impede. E assim por diante.

---

## Boas Praticas

1. **Modo de operacao explicito.** Sempre defina `mode` no squad.yaml.
2. **Guardrails em camadas.** Nunca confie em uma unica camada de protecao.
3. **Output isolado.** Todo output vai para `output/` — nunca modifica arquivos do projeto.
4. **Dependencias explicitas.** Se um squad precisa de outro, declare em `dependencies`.
5. **Versionamento semantico.** Versione seu squad (major.minor.patch).
6. **README completo.** Documente como usar o squad no README.md.
7. **Manifesto Synapse.** Sempre crie `.synapse/manifest` para integracao com o motor de contexto.

---

## Resumo

> Squads sao a unidade de extensibilidade do AIOS. Eles empacotam agentes, tasks, workflows e dados em modulos reutilizaveis focados em dominios especificos. A chave para squads eficientes e **foco** (um dominio por squad), **seguranca** (guardrails em camadas) e **composabilidade** (squads que se complementam).
