# SKILL — A Capacidade Especializada

## O Que E

Uma **Skill** e uma **capacidade ou modulo de conhecimento especializado** que pode ser disponibilizada para agentes. Diferente de um agente (que e uma persona completa com personalidade e autoridade), uma skill e uma **ferramenta reutilizavel** que qualquer agente pode acessar.

**Analogia simples:**
- **Agente** = Um medico (pessoa completa com formacao e especialidade)
- **Skill** = Uma tecnica medica (ex: sutura, intubacao) que varios medicos podem dominar

---

## Diferenca Entre Skill e Agent

| Aspecto | Agent | Skill |
|---------|-------|-------|
| **O que e** | Persona completa | Capacidade isolada |
| **Tem personalidade?** | Sim | Nao |
| **Tem autoridade exclusiva?** | Pode ter | Nao |
| **Reutilizavel?** | Nao (e unico) | Sim (multiplos agentes) |
| **Escopo** | Amplo (papel no time) | Focado (uma acao/conhecimento) |
| **Ativacao** | `@nome-do-agente` | Via agente ou diretamente |
| **Exemplo** | @dev (Developer) | Synapse management |

---

## Estrutura de uma Skill

No AIOS, skills sao definidas em pastas dedicadas com documentacao e logica:

```
.claude/skills/
  nome-da-skill/
    SKILL.md          # Documentacao e instrucoes da skill
    references/       # Arquivos de referencia e conhecimento
      arquivo1.md
      arquivo2.md
    examples/         # Exemplos de uso (opcional)
```

O arquivo `SKILL.md` define:
- **Nome e descricao** da skill
- **Comandos disponiveis** (como acionar)
- **Instrucoes de execucao** (o que fazer quando acionada)
- **Restricoes** (limites de atuacao)
- **Referencias** (arquivos de apoio)

---

## Skill Nativa — SYNAPSE Skill

O AIOS vem com uma skill nativa para gerenciamento do Synapse:

### Comandos
| Comando | Descricao |
|---------|-----------|
| `*synapse status` | Estado atual do motor de contexto |
| `*synapse domains` | Lista dominios registrados |
| `*synapse debug` | Mostra o que esta sendo injetado |
| `*synapse create` | Cria novo manifest de dominio |
| `*synapse add` | Adiciona regra a dominio existente |
| `*synapse edit` | Edita regras de dominio |
| `*synapse toggle` | Ativa/desativa dominio |

### Referencia
A skill Synapse tem sua propria pasta de referencias:
```
.claude/skills/synapse/
  SKILL.md              # Documentacao completa
  references/
    layers.md           # Detalhamento das 8 camadas
    manifest-format.md  # Formato do manifest
```

---

## Como Criar uma Skill

### Passo 1: Definir o Proposito
Identifique uma capacidade que:
- E reutilizavel por multiplos agentes
- Tem escopo bem definido
- Pode ser documentada de forma autonoma

### Passo 2: Criar a Estrutura

```bash
mkdir -p .claude/skills/minha-skill/references
```

### Passo 3: Escrever o SKILL.md

```markdown
# Minha Skill

## Descricao
O que esta skill faz e quando usar.

## Comandos
- `*minha-skill comando1` — Descricao
- `*minha-skill comando2` — Descricao

## Instrucoes de Execucao
Quando acionada, execute os seguintes passos:
1. ...
2. ...
3. ...

## Restricoes
- O que NAO fazer
- Limites de atuacao

## Referencias
- `references/guia.md` — Guia detalhado
```

### Passo 4: Adicionar Referencias
Coloque arquivos de referencia com conhecimento aprofundado que a skill precisa para funcionar.

---

## Casos de Uso

### 1. Skill de Geracao de Testes
```
Skill: test-generator
Comandos:
  *test unit <arquivo>    — Gera testes unitarios
  *test integration       — Gera testes de integracao
  *test coverage          — Analisa cobertura e sugere testes faltantes
```
**Quem usa:** @dev (durante implementacao), @qa (durante revisao)

### 2. Skill de Analise de Dependencias
```
Skill: dep-analyzer
Comandos:
  *deps audit             — Audita vulnerabilidades
  *deps outdated          — Lista dependencias desatualizadas
  *deps tree <pacote>     — Mostra arvore de dependencias
  *deps license           — Verifica licencas
```
**Quem usa:** @devops (antes de deploy), @qa (durante revisao)

### 3. Skill de Documentacao
```
Skill: doc-generator
Comandos:
  *doc api <path>         — Gera documentacao de API
  *doc component <path>   — Documenta um componente
  *doc changelog          — Gera changelog a partir de commits
  *doc adr <titulo>       — Cria Architecture Decision Record
```
**Quem usa:** @dev (pos-implementacao), @architect (decisoes)

### 4. Skill de Migracao de Banco
```
Skill: db-migration
Comandos:
  *migrate create <nome>  — Cria nova migration
  *migrate plan           — Mostra plano de execucao
  *migrate validate       — Valida migrations pendentes
  *migrate rollback-plan  — Gera plano de rollback
```
**Quem usa:** @data-engineer (design), @dev (implementacao)

### 5. Skill de Performance
```
Skill: perf-profiler
Comandos:
  *perf analyze <path>    — Analisa performance do codigo
  *perf query <sql>       — Analisa plano de execucao da query
  *perf bundle            — Analisa tamanho do bundle
  *perf lighthouse        — Gera relatorio Lighthouse
```
**Quem usa:** @dev (otimizacao), @qa (validacao)

### 6. Skill de Seguranca
```
Skill: security-scan
Comandos:
  *sec scan <path>        — Escaneia vulnerabilidades no codigo
  *sec owasp              — Verifica OWASP Top 10
  *sec secrets            — Detecta secrets no codigo
  *sec headers            — Valida headers de seguranca
```
**Quem usa:** @qa (revisao), @devops (pre-deploy)

---

## Skill vs Task — Qual a Diferenca?

| Aspecto | Skill | Task |
|---------|-------|------|
| **Natureza** | Capacidade/conhecimento | Procedimento passo-a-passo |
| **Duracao** | Persistente (sempre disponivel) | Pontual (executada e concluida) |
| **Interatividade** | Pode ser interativa | Geralmente sequencial |
| **Reutilizacao** | Alta (chamada a qualquer momento) | Media (dentro de workflows) |
| **Escopo** | Uma acao/analise | Um fluxo completo |
| **Exemplo** | "Analise este codigo" | "Execute o ciclo completo de testes" |

**Regra pratica:**
- Se e algo que voce **faz repetidamente em momentos diferentes** -> Skill
- Se e algo que **segue uma sequencia de passos** -> Task

---

## Area de Criatividade

### Skills Inovadoras para Criar

1. **Skill de Revisao de PR Automatizada**
   - Analisa diff, verifica padroes, sugere melhorias
   - Gera checklist de revisao pre-preenchido
   - Compara com PRs similares anteriores

2. **Skill de Monitoramento de Custos**
   - Estima custo de queries, chamadas de API, storage
   - Alerta quando mudancas podem aumentar custos
   - Sugere otimizacoes de custo

3. **Skill de Acessibilidade**
   - Verifica conformidade WCAG
   - Sugere melhorias de acessibilidade
   - Gera relatorio de score

4. **Skill de Internacionalizacao**
   - Detecta strings hardcoded
   - Sugere estrutura i18n
   - Valida cobertura de traducoes

5. **Skill de Onboarding**
   - Explica qualquer parte do codigo
   - Gera "mapa do tesouro" do projeto
   - Cria guias de contribuicao personalizados

6. **Skill de Retrospectiva**
   - Analisa commits do sprint
   - Identifica padroes (velocidade, bugs, areas problematicas)
   - Sugere melhorias de processo

7. **Skill de Estimativa**
   - Analisa complexidade de uma story
   - Compara com stories similares ja completadas
   - Sugere story points baseado em historico

---

## Boas Praticas

1. **Uma skill, um dominio.** Nao crie skills que cobrem tudo — foque em uma area.
2. **Comandos intuitivos.** O nome do comando deve deixar claro o que ele faz.
3. **Documentacao completa no SKILL.md.** A skill deve ser autoexplicativa.
4. **Referencias ricas.** Inclua guias detalhados na pasta `references/`.
5. **Restricoes explicitas.** Documente o que a skill NAO faz.
6. **Testavel.** Cada comando da skill deve ter um resultado verificavel.

---

## Resumo

> Skills sao modulos de capacidade reutilizaveis que ampliam o que os agentes podem fazer. Enquanto agentes sao "quem faz", skills sao "como faz". Uma boa arquitetura de skills permite que agentes diferentes compartilhem capacidades sem duplicar logica.
