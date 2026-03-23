# AGENT — O Especialista

## O Que E

Um **Agent** (agente) e uma **persona de IA com expertise, personalidade e autoridade definidas**. Cada agente tem um papel claro, comandos exclusivos e limites de atuacao. Eles sao os "membros do time" que executam o trabalho real.

Diferente de usar uma IA generica, um agente sabe **quem ele e, o que pode fazer e o que NAO pode fazer**.

---

## Anatomia de um Agente

Todo agente e definido por um arquivo `.md` com a seguinte estrutura:

```markdown
# @nome-do-agente (Persona)

## Identidade
- Nome: Persona do agente
- Papel: O que ele faz
- Expertise: Areas de conhecimento

## Autoridade
- O que este agente pode fazer EXCLUSIVAMENTE
- O que ele NAO pode fazer

## Comandos
- Lista de comandos disponiveis (*comando)

## Comportamento
- Como ele se comunica
- Que tom usa
- Como aborda problemas

## Restricoes
- Limites claros de atuacao
- Guardrails de seguranca
```

---

## Os 11 Agentes Core do AIOS

### Agentes de Planejamento (Fase Web/Planning)

#### @analyst (Alex)
- **Papel:** Pesquisa de mercado, analise competitiva, fundacao do PRD
- **Quando usar:** Antes de comecar qualquer projeto — para entender o mercado e validar a ideia
- **Output:** Relatorios de pesquisa, analises competitivas, insights de mercado

#### @pm (Morgan)
- **Papel:** Product Manager — cria PRD, define prioridades, roadmap
- **Quando usar:** Para transformar pesquisa em especificacao de produto
- **Output:** PRD (Product Requirements Document), roadmap, priorizacao de features

#### @architect (Aria)
- **Papel:** Arquitetura de sistemas, design tecnico, decisoes de tecnologia
- **Autoridade exclusiva:** Decisoes de arquitetura
- **Quando usar:** Para definir como o sistema sera construido tecnicamente
- **Output:** Documento de arquitetura, diagramas, ADRs

#### @ux-design-expert (Uma)
- **Papel:** Design de UX/UI, usabilidade, interacao
- **Quando usar:** Para definir como o usuario interage com o produto
- **Output:** Wireframes, fluxos de usuario, guidelines de design

### Agentes de Desenvolvimento (Fase IDE)

#### @sm (River) — Scrum Master
- **Papel:** Gestao de sprint, criacao de stories, facilitacao
- **Autoridade:** Criacao de stories
- **Quando usar:** Para organizar o trabalho em stories executaveis
- **Output:** Stories detalhadas com contexto completo de implementacao

#### @po (Pax) — Product Owner
- **Papel:** Backlog, priorizacao, gestao de epicos e stories
- **Autoridade:** Criacao de stories, gestao de backlog
- **Quando usar:** Para decidir O QUE construir e em que ordem
- **Output:** Epicos, stories priorizadas, criterios de aceitacao

#### @dev (Dex) — Developer
- **Papel:** Implementacao de codigo, execucao de stories
- **Quando usar:** Para escrever codigo, implementar features, corrigir bugs
- **Output:** Codigo funcional, testes unitarios

#### @qa (Quinn) — Quality Assurance
- **Papel:** Testes, revisao de codigo, garantia de qualidade
- **Autoridade exclusiva:** Vereditos de qualidade
- **Quando usar:** Para revisar codigo, validar implementacoes, criar testes
- **Output:** Relatorios de qualidade, testes, vereditos (PASS/FAIL/CONCERNS)

#### @data-engineer (Dara)
- **Papel:** Design de banco de dados, modelagem de dados, ETL
- **Quando usar:** Para projetar schemas, migrations, queries complexas
- **Output:** Schemas de banco, migrations, documentacao de dados

#### @devops (Gage)
- **Papel:** CI/CD, infraestrutura, deploy
- **Autoridade EXCLUSIVA:** git push, criacao de PR, releases, tags
- **Quando usar:** Para deploy, configuracao de CI/CD, infraestrutura
- **Output:** Pipelines CI/CD, configs de infra, releases

#### @aios-master
- **Papel:** Meta-orquestracao do framework, desenvolvimento do proprio AIOS
- **Quando usar:** Para modificar, estender ou debugar o proprio framework
- **Output:** Melhorias no framework, novos agentes, novas capacidades

---

## Matriz de Autoridade

Esta tabela define **quem pode fazer o que** — e e inviolavel:

| Acao | Agente Exclusivo | Outros Agentes |
|------|-----------------|----------------|
| git push | @devops | BLOQUEADO |
| Criar PR | @devops | BLOQUEADO |
| Release/Tag | @devops | BLOQUEADO |
| Criar stories | @sm, @po | — |
| Decisoes de arquitetura | @architect | — |
| Vereditos de qualidade | @qa | — |
| Implementar codigo | @dev | — |

**Regra inviolavel:** Nenhum agente pode assumir a autoridade de outro. Se @dev tentar fazer push, ele sera BLOQUEADO pelo Synapse (L0 — Constitution).

---

## Pipeline de Ativacao de Agentes

Quando voce ativa um agente, o sistema executa automaticamente:

```
Ativacao (@dev, /agent-name, etc.)
    |
    v
Fase 1: Carregamento Paralelo (80-180ms)
  |-- AgentConfigLoader (definicao do agente)
  |-- SessionContextLoader (estado da sessao)
  |-- ProjectStatusLoader (branch git, arquivos modificados)
  |-- GitConfigDetector (tipo de repositorio)
  |-- PermissionMode (nivel de autorizacao)
    |
    v
Fase 2: Construcao da Definicao do Agente
    |
    v
Fase 3: Deteccao Sequencial
  |-- GreetingPreferenceManager (consciencia de sessao)
  |-- ContextDetector (contexto novo/existente/workflow)
  |-- WorkflowNavigator (deteccao de estado)
    |
    v
Fase 4: Montagem do Contexto Enriquecido
    |
    v
Fase 5: Construcao do Greeting
    |
    v
Agente ativado com contexto completo
```

### Secoes do Greeting

Quando um agente e ativado, ele se apresenta com:

1. **Apresentacao** — Nome + badge de permissao ([Ask], [Auto], [Explore])
2. **Descricao do Papel** — O que ele faz e sua expertise
3. **Status do Projeto** — Branch atual, arquivos modificados, commits recentes
4. **Contexto** — Narrativa inteligente baseada em acoes anteriores
5. **Sugestoes de Workflow** — Proximos passos baseados no historico
6. **Comandos** — Lista de comandos disponiveis (filtrados por visibilidade)
7. **Rodape** — Assinatura de encerramento

---

## Modos de Permissao

| Modo | Badge | Escreve | Executa | Deleta | Padrao |
|------|-------|---------|---------|--------|--------|
| `explore` | [Explore] | Bloqueado | Bloqueado | Bloqueado | Nao |
| `ask` | [Ask] | Confirma | Confirma | Confirma | **Sim** |
| `auto` | [Auto] | Permitido | Permitido | Permitido | Nao |

**Comando para alternar:** `*yolo` (cicla: ask -> auto -> explore -> ask)

---

## Agentes de Squad (Customizados)

Alem dos 11 agentes core, cada squad pode definir seus proprios agentes especializados. Exemplos dos seus squads:

| Squad | Agente | Persona | Papel |
|-------|--------|---------|-------|
| analisador-n8n | @analisador | Sherlock | Investigador read-only |
| auditor-real | @commander | Argus | Orquestrador de auditoria |
| auditor-real | @coletor | Reaper | Coletor de evidencias |
| auditor-real | @avaliador | Judge | Avaliador de interacoes |
| auditor-real | @classificador | Ranker | Classificador de problemas |
| auditor-real | @deep-agent | Pathfinder | Investigador profundo |
| auditor-real | @safety-officer | — | Validador de seguranca |
| deploy-review | @diff-analyst | — | Analista de diferencas |
| deploy-review | @error-hunter | — | Cacador de erros |
| squad-pixel | @pixel-lead | Pixel | Lider de pixel tracking |
| squad-pixel | @pixel-compliance | Shield | Compliance LGPD |
| testador-n8n | @testador | Watson | Testador DEV-only |

---

## Casos de Uso

### 1. Desenvolvimento Completo de Feature
```
@analyst -> Pesquisa de mercado
@pm -> Cria PRD
@architect -> Define arquitetura
@sm -> Cria stories
@dev -> Implementa
@qa -> Revisa e testa
@devops -> Deploy
```

### 2. Hotfix de Bug
```
@dev -> Identifica e corrige o bug
@qa -> Valida a correcao
@devops -> Push e deploy
```

### 3. Auditoria de Sistema
```
@analisador -> Investiga producao (read-only)
@commander -> Orquestra auditoria
@avaliador + @classificador -> Avaliam em paralelo
@deep-agent -> Investiga falhas criticas
@safety-officer -> Valida completude
```

### 4. Refatoracao de Codigo
```
@architect -> Define estrategia de refatoracao
@dev -> Executa refatoracao
@qa -> Valida que nada quebrou
@devops -> Faz merge e deploy
```

### 5. Novo Projeto (Greenfield)
```
@analyst -> Pesquisa e validacao
@pm -> PRD completo
@architect -> Arquitetura do zero
@ux-design-expert -> Design de interfaces
@sm -> Sharda em epicos e stories
@dev -> Implementacao iterativa
```

---

## Area de Criatividade

### Ideias para Novos Agentes

1. **@security-auditor** — Agente especializado em seguranca que analisa codigo buscando vulnerabilidades OWASP, verifica dependencias, e audita configuracoes.

2. **@tech-writer** — Agente focado em documentacao tecnica, gera docs automaticamente a partir do codigo, mantem READMEs atualizados.

3. **@performance-engineer** — Analisa performance de queries, endpoints, componentes. Sugere otimizacoes baseadas em benchmarks.

4. **@incident-responder** — Agente para resposta a incidentes que segue runbooks, coleta logs, e gera post-mortems.

5. **@cost-optimizer** — Analisa custos de infraestrutura cloud, sugere rightsizing, identifica recursos ociosos.

6. **@migration-specialist** — Especialista em migracoes de banco, APIs, frameworks. Gera planos de migracao com rollback.

7. **@compliance-officer** — Verifica aderencia a regulacoes (LGPD, SOC2, HIPAA), gera relatorios de compliance.

8. **@onboarding-buddy** — Agente que ajuda novos membros do time a entender o projeto, guiando-os pelo codigo e arquitetura.

### Combinacoes Criativas

- **@analyst + @data-engineer:** Pesquisa de mercado informada por dados reais do banco
- **@qa + @security-auditor:** Review de qualidade E seguranca em uma unica passada
- **@architect + @cost-optimizer:** Arquitetura que considera custo desde o design
- **@dev + @tech-writer:** Implementacao com documentacao automatica

---

## Boas Praticas

1. **Um agente, uma responsabilidade.** Nao crie agentes que fazem tudo — divida em especialistas.
2. **Autoridade clara.** Sempre defina o que o agente pode e NAO pode fazer.
3. **Persona consistente.** A personalidade do agente ajuda o LLM a manter o foco.
4. **Guardrails explicitos.** Documente restricoes no proprio arquivo do agente.
5. **Comandos nomeados com clareza.** `*investigate` e melhor que `*run-task-1`.

---

## Resumo

> Agentes sao a forca de trabalho do AIOS. Cada um e um especialista com papel claro, autoridade definida e restricoes explicitas. A chave para um sistema eficiente e criar agentes focados, com responsabilidades que nao se sobrepoe, e autoridades que nao conflitam.
