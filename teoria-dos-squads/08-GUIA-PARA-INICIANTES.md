# Guia para Iniciantes — AIOS Explicado de Forma Simples

## Antes de Tudo: O Que e o AIOS?

Imagine que voce pudesse montar um **time inteiro de especialistas** que trabalham juntos para resolver qualquer problema — mas em vez de pessoas reais, sao **agentes de IA** organizados e coordenados.

O **Synkra AIOS** e exatamente isso: um framework que organiza agentes de IA em times, define regras, processos e automacoes para que eles trabalhem de forma eficiente e segura.

---

## Os 6 Conceitos — Explicados Como Se Voce Tivesse 10 Anos

### 1. SYNAPSE — O Cerebro

**O que e?**
O Synapse e como o **cerebro** do sistema. Ele decide quais informacoes cada agente precisa saber para fazer seu trabalho.

**Analogia do dia a dia:**
Imagine que voce trabalha em uma escola. Quando o professor de matematica entra na sala, ele recebe:
- O planejamento da aula de hoje
- A lista de alunos
- O nivel da turma
- As regras da escola

Ele NAO recebe o planejamento do professor de historia. O Synapse faz exatamente isso — da a cada "professor" (agente) apenas o que ele precisa.

**Por que importa?**
Sem o Synapse, todo agente receberia TODA a informacao de TODOS os dominios — seria como dar o livro inteiro da escola para cada professor. Ineficiente e confuso.

**Exemplo pratico:**
```
Voce: "quero testar o webhook do WhatsApp"

Synapse detecta as palavras "webhook" e "WhatsApp"
    --> Carrega regras do squad de testes
    --> Carrega regras de seguranca (nunca acessar producao)
    --> Carrega informacoes do ambiente de desenvolvimento
    --> Agente recebe APENAS o contexto relevante
```

---

### 2. AGENT — O Especialista

**O que e?**
Um agente e um **especialista virtual** com nome, personalidade e area de atuacao definida. Ele sabe o que pode e o que NAO pode fazer.

**Analogia do dia a dia:**
Em uma empresa, voce tem:
- O **gerente de projeto** (planeja)
- O **programador** (escreve codigo)
- O **testador** (verifica qualidade)
- O **cara do servidor** (faz deploy)

Cada um tem seu papel. O programador nao manda o codigo pro ar (isso e do cara do servidor). O testador nao escreve features (isso e do programador).

No AIOS e igual:
- `@dev` (Dex) — Programa
- `@qa` (Quinn) — Testa
- `@devops` (Gage) — Faz deploy
- `@architect` (Aria) — Projeta a arquitetura

**Regra de ouro:** Ninguem faz o trabalho do outro.

**Exemplo pratico:**
```
Voce pede para o @dev: "faca deploy do projeto"
@dev responde: "Nao posso fazer deploy.
                Isso e responsabilidade do @devops.
                Posso preparar o codigo para deploy."
```

---

### 3. SKILL — A Ferramenta

**O que e?**
Uma skill e uma **capacidade especifica** que um agente pode usar. E como uma ferramenta na caixa de ferramentas.

**Analogia do dia a dia:**
Um mecanico (agente) pode saber:
- Trocar oleo (skill 1)
- Trocar pneu (skill 2)
- Diagnosticar motor (skill 3)

Cada uma e uma habilidade separada. O mesmo mecanico pode ter varias. E um eletricista (outro agente) tambem pode saber "diagnosticar motor" — a skill e **compartilhavel**.

**Diferenca entre Skill e Agent:**
| | Agent | Skill |
|---|---|---|
| E uma pessoa? | Sim (persona completa) | Nao (apenas uma ferramenta) |
| Tem nome? | Sim (@dev, @qa) | Sim (test-generator) |
| Pode ser compartilhada? | Nao (e unica) | Sim (varios agentes usam) |

**Exemplo pratico:**
```
Skill: "analisar seguranca"

Quem pode usar:
  - @dev (enquanto escreve codigo)
  - @qa (enquanto revisa codigo)
  - @devops (antes de fazer deploy)
```

---

### 4. SQUAD — O Time

**O que e?**
Um squad e um **time completo** montado para resolver um tipo especifico de problema. Ele tem seus proprios agentes, tarefas, processos e regras.

**Analogia do dia a dia:**
Uma escola tem **departamentos**:
- Departamento de Matematica (professores de mat, provas de mat, regras de mat)
- Departamento de Portugues (professores de pt, provas de pt, regras de pt)
- Departamento de Educacao Fisica (professores de ed. fisica, atividades, regras)

Cada departamento e um "squad". Ele tem tudo que precisa para funcionar de forma independente.

**No AIOS:**
```
Squad "auditor-real" (time de auditoria):
  Agentes:
    - Commander (organiza a auditoria)
    - Coletor (busca dados)
    - Avaliador (avalia qualidade)
    - Classificador (classifica problemas)
    - Deep-Agent (investiga falhas graves)
    - Safety-Officer (garante seguranca)

  Tarefas:
    - Resolver usuario
    - Coletar mensagens
    - Avaliar interacoes
    - Gerar relatorio

  Regras:
    - NUNCA modificar dados
    - Apenas LEITURA
    - Resultado vai para pasta output/
```

**Por que usar squads?**
Sem squads, voce teria que configurar tudo do zero toda vez. Com squads, voce "instala" o time inteiro de uma vez — plug and play.

---

### 5. WORKFLOW — O Processo

**O que e?**
Um workflow e o **passo-a-passo** que define como os agentes trabalham juntos para completar uma tarefa complexa.

**Analogia do dia a dia:**
Quando voce vai ao medico:
1. **Recepcao** — Se identifica, preenche ficha
2. **Triagem** — Enfermeira mede pressao, temperatura
3. **Consulta** — Medico examina
4. **Exames** — Se necessario, faz exames (condicional!)
5. **Diagnostico** — Medico analisa resultados
6. **Prescricao** — Medico receita tratamento

Perceba:
- As etapas tem **ordem** (nao faz exame antes da consulta)
- Algumas sao **condicionais** (so faz exame se medico pedir)
- Algumas poderiam ser **paralelas** (medir pressao E temperatura ao mesmo tempo)

**No AIOS e igual:**
```
Workflow "audit-user":
  Passo 1: Commander identifica o usuario
  Passo 2: Coletor busca mensagens
  Passo 3: Avaliador + Classificador trabalham AO MESMO TEMPO
  Passo 4: Deep-Agent investiga (SO SE houver falha grave)
  Passo 5: Commander gera relatorio
  Passo 6: Safety-Officer valida tudo
```

**Conceitos importantes:**
- **Sequencial:** Um depois do outro (Passo 1 -> 2 -> 3)
- **Paralelo:** Dois ao mesmo tempo (Avaliador + Classificador)
- **Condicional:** So acontece SE... (Deep-Agent so se falha critica)
- **Gate:** Checkpoint — so avanca se tudo estiver OK

---

### 6. HOOK — O Alarme Automatico

**O que e?**
Um hook e uma **acao automatica** que dispara quando algo acontece. Voce nao precisa apertar um botao — ele reage sozinho.

**Analogia do dia a dia:**
- **Sensor de presenca:** Voce entra na sala, a luz acende
- **Alarme de incendio:** Detecta fumaca, toca sirene
- **Notificacao do celular:** Recebe email, celular apita

**No AIOS:**
```
Quando o agente termina uma tarefa (EVENTO)
    --> Hook envia notificacao para voce (ACAO)

Quando alguem tenta deletar um arquivo (EVENTO)
    --> Hook bloqueia e avisa: "acao perigosa!" (ACAO)

Quando a sessao esta acabando (EVENTO)
    --> Hook salva um resumo do que foi feito (ACAO)
```

**Por que importa?**
Hooks fazem coisas que voce **esqueceria de fazer manualmente** — como salvar progresso, verificar seguranca, ou formatar arquivos.

---

## Como Tudo Trabalha Junto — Um Exemplo Completo

Vamos seguir um exemplo real, passo a passo:

### Cenario: Voce quer auditar as interacoes do usuario "Maria"

**Voce digita:** `"auditar as interacoes da Maria"`

### O Que Acontece Por Tras

```
PASSO 1: HOOKS verificam
  "O usuario tem permissao para isso? SIM"
  "O comando e seguro? SIM"
  "Vou registrar essa acao no log."

PASSO 2: SYNAPSE processa seu texto
  "Detectei 'auditar' -> dominio de auditoria"
  "Detectei 'interacoes' -> coleta de mensagens"
  "Vou carregar as regras do squad auditor-real"
  "Vou carregar as regras do agente Commander"
  "Vou garantir que a Constitution (READ-ONLY) esta ativa"
  --> Monta pacote de contexto

PASSO 3: AGENT (@commander) recebe o contexto
  "Sou o Commander (Argus). Minha missao: orquestrar esta auditoria."
  "Regras: apenas leitura, nunca modificar dados."
  "Vou iniciar o workflow 'audit-user'."

PASSO 4: WORKFLOW comeca

  Fase 1 - Resolver Usuario
  @commander usa TASK "resolve-user"
  --> Busca "Maria" no banco
  --> Encontra: Maria Silva, tel: 5543999999, plano: Premium

  Fase 2 - Coletar Dados
  @coletor usa TASK "collect-messages"
  --> Busca mensagens no Supabase
  --> Busca logs no N8N
  --> Busca registros no OpenAI
  --> Monta timeline com 47 interacoes

  Fase 3 - Analisar (PARALELO!)
  @avaliador avalia cada interacao (nota 0-100)
  @classificador classifica problemas encontrados
  (Os dois trabalham ao mesmo tempo - mais rapido!)

  Resultados:
  - 40 interacoes OK (score > 80)
  - 5 interacoes parciais (score 50-80)
  - 2 falhas criticas (score < 50) !!

  Fase 4 - Investigacao Profunda (CONDICIONAL)
  "Tem falhas criticas? SIM (2 encontradas)"
  @deep-agent investiga as 2 falhas
  --> Analise Swiss Cheese (6 camadas)
  --> Encontra: timeout no N8N causou perda de mensagem
  --> Encontra: modelo AI nao entendeu guria (erro de contexto)

  Fase 5 - Consolidar
  @commander junta todos os resultados
  --> Gera relatorio completo em .md
  --> Gera timeline em .json
  --> Gera ranking de problemas

  Fase 6 - Validar
  @safety-officer verifica 7 gates de seguranca
  --> CVR: mensagens capturadas? SIM
  --> HFACS: severidade classificada? SIM
  --> RCA: causa raiz identificada? SIM
  --> SMS: ciclo PDCA registrado? SIM
  --> Swiss Cheese: rastreabilidade? SIM
  --> LOSA: padroes detectados? SIM
  --> Kaizen: melhoria sugerida? SIM
  --> RESULTADO: AUDITORIA VALIDA

PASSO 5: OUTPUT gerado
  output/2024-03-23-audit-maria-silva.md
  output/2024-03-23-timeline-maria-silva.json
  output/2024-03-23-ranking-maria-silva.md
  output/2024-03-23-deep-timeout-n8n.md
  output/2024-03-23-deep-contexto-ai.md

PASSO 6: HOOKS finalizam
  "Auditoria concluida! Notificando usuario."
  "Registrando no log de auditoria."
```

---

## Perguntas Frequentes (FAQ)

### "Preciso saber programar para usar?"
**Nao necessariamente.** Para USAR squads existentes, basta saber os comandos. Para CRIAR novos squads, algum conhecimento tecnico ajuda, mas os arquivos sao principalmente em Markdown (texto formatado) e YAML (listas organizadas).

### "Posso usar so uma parte? Tipo, so o Synapse?"
**Sim.** Os componentes sao modulares. Voce pode usar apenas agentes sem workflows, ou squads sem hooks. Mas eles funcionam melhor juntos.

### "E seguro? Nao vai deletar meus dados?"
**Sim, se configurado corretamente.** O AIOS tem ate 7 camadas de protecao (guardrails). No modo `strict-read-only`, nenhuma escrita e permitida. No modo `dev-write-prod-block`, so escreve no ambiente de desenvolvimento.

### "Qual a diferenca entre AIOS e usar o ChatGPT/Claude direto?"
| Aspecto | ChatGPT/Claude direto | AIOS |
|---------|----------------------|------|
| Contexto | Voce fornece manualmente | Synapse fornece automaticamente |
| Especialidade | Generica | Agentes especializados |
| Seguranca | Depende de voce | Guardrails automaticos |
| Processo | Ad-hoc | Workflows estruturados |
| Reproducibilidade | Baixa | Alta (tudo documentado) |
| Escalabilidade | Limitada | Squads composiveis |

### "Posso criar meu proprio squad?"
**Sim!** Basta criar uma pasta com a estrutura correta, definir agentes, tasks, workflows e um manifesto. O comando `*create-squad` guia voce interativamente.

### "O que e o .synapse/manifest?"
E um arquivo de texto simples que registra as **regras de dominio** do seu squad no motor de contexto (Synapse). Sem ele, o squad funciona mas nao se integra automaticamente com o contexto.

### "O que e um bracket (FRESH, MODERATE, DEPLETED, CRITICAL)?"
E uma forma de medir **quanto da conversa ja foi consumida**. Como uma barra de bateria:
- FRESH (60-100%) = Bateria cheia, injecao leve
- MODERATE (40-60%) = Meia carga, injecao normal
- DEPLETED (25-40%) = Bateria baixa, reforco de regras criticas
- CRITICAL (0-25%) = Quase sem bateria, alerta de handoff

### "O que e um gate?"
E um **checkpoint de qualidade** dentro de um workflow. Antes de avancar para a proxima fase, o gate verifica se tudo esta OK. Se nao estiver, pode:
- Repetir a fase (retry)
- Escalar para outro agente (escalate)
- Parar o workflow (fail)

### "Posso usar com qualquer IA?"
O AIOS foi projetado para funcionar com **Claude Code** (melhor integracao), **Gemini CLI**, **Codex CLI**, **Cursor** e **GitHub Copilot**. A cobertura de features varia por IDE.

---

## Glossario Rapido

| Termo | Significado |
|-------|------------|
| **Agent** | Especialista virtual com papel definido |
| **Bracket** | Nivel de consumo do contexto (bateria) |
| **CLI** | Interface de linha de comando (terminal) |
| **Constitution** | Principios inviolaveis do sistema |
| **Flow-State** | Estado atual de um workflow em execucao |
| **Gate** | Checkpoint de qualidade no workflow |
| **Guardrail** | Camada de protecao/seguranca |
| **Hook** | Acao automatica em resposta a evento |
| **L0-L7** | As 8 camadas do Synapse |
| **Manifest** | Arquivo de configuracao do squad/dominio |
| **Pipeline** | Sequencia de processamento |
| **Skill** | Capacidade reutilizavel por agentes |
| **Squad** | Time modular de agentes para um dominio |
| **Synapse** | Motor de contexto inteligente |
| **Task** | Procedimento passo-a-passo |
| **Workflow** | Orquestracao de multiplas etapas |

---

## Proximos Passos

1. **Explore os squads existentes** — Leia os README.md de cada squad em `squads/`
2. **Ative um agente** — Use `/agent-name` ou `@agent-name` para experimentar
3. **Execute um workflow** — Siga os comandos do squad (ex: `*audit "Maria"`)
4. **Crie seu primeiro squad** — Use `*create-squad` para comecar
5. **Customize regras** — Edite o `.synapse/manifest` para ajustar o contexto

---

## Resumo em Uma Frase

> O AIOS e um sistema que organiza agentes de IA em **times especializados** (squads) com **processos definidos** (workflows), **ferramentas compartilhadas** (skills), **contexto inteligente** (synapse), e **automacoes de seguranca** (hooks) — tudo coordenado para resolver problemas complexos de forma escalavel e segura.
