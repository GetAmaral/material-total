# HOOK — O Gatilho Automatico

## O Que E

Um **Hook** e um **comando que executa automaticamente em resposta a um evento** do ciclo de vida do sistema. Funciona como um interceptador — ele "escuta" eventos e reage sem que voce precise acionar manualmente.

**Analogia simples:** Hooks sao como **sensores automaticos** — quando a porta abre (evento), a luz acende (hook). Voce nao precisa apertar o interruptor.

---

## Como Funcionam

```
EVENTO acontece (ex: sessao inicia)
    |
    v
Hook Registry detecta o evento
    |
    v
Hook registrado para esse evento e executado
    |
    v
Resultado do hook e processado
    |
    v
Fluxo normal continua
```

---

## Eventos Suportados

| Evento | Quando Dispara | Uso Tipico |
|--------|----------------|------------|
| `sessionStart` | Inicio de sessao | Carregar contexto, verificar ambiente |
| `beforeAgent` | Antes de ativar um agente | Validar permissoes, preparar contexto |
| `beforeTool` | Antes de usar uma ferramenta | Verificar seguranca, logar acao |
| `afterTool` | Apos usar uma ferramenta | Logar resultado, atualizar estado |
| `sessionEnd` | Fim de sessao | Salvar progresso, gerar resumo |
| `precompact` | Antes de compactar contexto | Capturar digest da sessao (memoria) |

---

## Arquitetura de Hooks no AIOS

### Hook Interface Unificada

O AIOS abstrai as diferencas entre CLIs (Claude Code, Gemini, Codex) atraves de uma **interface unificada**:

```
Claude Code hooks    --\
Gemini CLI hooks     ----> Hook Interface Unificada ----> Hook Registry
Codex CLI hooks      --/                                      |
                                                              v
                                                        Hook Runners
                                                     (executam a logica)
```

### Estrutura de Arquivos

```
.aios-core/hooks/
|
|-- unified/                    # Hooks cross-CLI
|   |-- hook-interface.js         Normalizacao entre CLIs
|   |-- hook-registry.js          Descoberta e execucao
|   |-- runners/
|       |-- precompact-runner.js  Captura digest pre-compactacao
|
|-- gemini/                     # Hooks especificos Gemini
|   |-- beforeAgent.js
|   |-- beforeTool.js
|   |-- afterTool.js
```

### Mapeamento de Eventos por CLI

| Evento AIOS | Claude Code | Gemini CLI |
|-------------|-------------|------------|
| sessionStart | hook: sessionStart | Gemini lifecycle |
| beforeAgent | hook: beforeAgent | beforeAgent.js |
| beforeTool | hook: beforeTool | beforeTool.js |
| afterTool | hook: afterTool | afterTool.js |
| sessionEnd | hook: sessionEnd | Gemini lifecycle |
| precompact | hook: precompact | precompact-runner.js |

---

## Hook Especial: Precompact

O hook **precompact** e particularmente importante. Ele dispara **antes da compactacao de contexto** — o momento em que a IA precisa resumir a conversa para liberar espaco.

### O que ele faz:
1. Captura um **digest** (resumo estruturado) da sessao atual
2. Identifica **informacoes criticas** que nao podem ser perdidas
3. Formata o digest para preservacao
4. Alimenta o Memory Intelligence System (Pro)

### Por que importa:
Sem precompact, informacoes importantes podem ser perdidas quando o contexto e compactado. Com ele, o sistema **preserva** decisoes, descobertas e contexto critico.

---

## Hooks no Claude Code

No Claude Code, hooks sao configurados em `settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "echo 'Verificando seguranca antes de executar Bash'"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "echo 'Arquivo escrito, verificando formatacao'"
      }
    ],
    "Notification": [
      {
        "matcher": "",
        "command": "echo 'Tarefa concluida!'"
      }
    ]
  }
}
```

### Tipos de Hook no Claude Code

| Tipo | Quando | Exemplo |
|------|--------|---------|
| `PreToolUse` | Antes de usar ferramenta | Validar comando, verificar permissao |
| `PostToolUse` | Apos usar ferramenta | Formatar output, logar resultado |
| `Notification` | Quando ha notificacao | Alertar usuario, tocar som |
| `Stop` | Quando agente para | Salvar estado, gerar resumo |

### Matcher
O `matcher` filtra **qual ferramenta** dispara o hook:
- `"Bash"` — Apenas quando Bash e usado
- `"Write"` — Apenas quando Write e usado
- `""` (vazio) — Qualquer ferramenta

---

## Casos de Uso

### 1. Validacao de Seguranca Pre-Execucao
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "if echo '$TOOL_INPUT' | grep -q 'rm -rf'; then echo 'BLOCKED: comando perigoso'; exit 1; fi"
      }
    ]
  }
}
```
**O que faz:** Antes de qualquer comando Bash, verifica se contem `rm -rf` e bloqueia.

### 2. Formatacao Automatica Pos-Escrita
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "npx prettier --write '$TOOL_INPUT_PATH' 2>/dev/null || true"
      }
    ]
  }
}
```
**O que faz:** Apos escrever qualquer arquivo, roda Prettier automaticamente.

### 3. Notificacao de Conclusao
```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "command": "osascript -e 'display notification \"Claude terminou!\" with title \"AIOS\"'"
      }
    ]
  }
}
```
**O que faz:** Mostra notificacao do sistema quando Claude termina uma tarefa.

### 4. Log de Auditoria
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "",
        "command": "echo \"$(date) | $TOOL_NAME | $TOOL_INPUT\" >> ~/.aios-audit.log"
      }
    ]
  }
}
```
**O que faz:** Registra toda acao em um log de auditoria.

### 5. Verificacao de Branch Pre-Push
Um hook que verifica se voce esta na branch correta antes de permitir push:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "if echo '$TOOL_INPUT' | grep -q 'git push' && git branch --show-current | grep -q 'main'; then echo 'BLOCKED: nao faca push direto na main'; exit 1; fi"
      }
    ]
  }
}
```

### 6. Auto-Commit de Progresso
```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "command": "cd /path/to/project && git add -A && git commit -m 'auto: save progress' 2>/dev/null || true"
      }
    ]
  }
}
```
**O que faz:** Quando o agente para, salva automaticamente o progresso com commit.

---

## Hooks no Contexto de Squads

Squads podem definir hooks especificos para seu dominio:

### Hook de Guardrail de Squad
```
Antes de qualquer acao do squad:
  1. Verificar se o modo e READ-ONLY
  2. Verificar se o ambiente e DEV
  3. Verificar se o usuario tem permissao
  4. Logar a acao no audit trail
```

### Hook de Output de Squad
```
Apos gerar output:
  1. Validar formato do output
  2. Verificar se esta no diretorio correto (output/)
  3. Gerar hash de integridade
  4. Notificar conclusao
```

---

## Area de Criatividade

### Hooks Inovadores para Criar

1. **Hook de Custo**
   Estima o custo de cada chamada de API e bloqueia se exceder limite:
   ```
   PreToolUse: Se custo_estimado > $0.10, pede confirmacao
   ```

2. **Hook de Qualidade Continua**
   Roda linter apos cada edicao de arquivo:
   ```
   PostToolUse(Write): eslint --fix arquivo_editado
   ```

3. **Hook de Contexto**
   Antes de cada acao, injeta contexto relevante do projeto:
   ```
   PreToolUse: Le README.md e injeta resumo no contexto
   ```

4. **Hook de Metricas**
   Coleta metricas de performance de cada acao:
   ```
   PostToolUse: Registra tempo_execucao, tokens_usados, resultado
   ```

5. **Hook de Rollback**
   Cria snapshot antes de mudancas destrutivas:
   ```
   PreToolUse(Bash): Se comando modifica arquivos, cria backup
   ```

6. **Hook de Compliance**
   Verifica se cada acao esta em conformidade com politicas:
   ```
   PreToolUse: Verifica se acao viola politica de privacidade
   ```

7. **Hook de Aprendizado**
   Registra padroes de uso para melhorar sugestoes futuras:
   ```
   PostToolUse: Registra comando + resultado para pattern matching
   ```

---

## Boas Praticas

1. **Hooks devem ser rapidos.** Se um hook demora, ele atrasa toda a execucao.
2. **Hooks devem ser idempotentes.** Executar duas vezes deve ter o mesmo efeito.
3. **Hooks nao devem falhar silenciosamente.** Se algo der errado, registre.
4. **Use matchers especificos.** Nao use matcher vazio se so precisa de um tipo.
5. **Teste hooks isoladamente.** Execute o comando do hook manualmente antes.
6. **Documente seus hooks.** Outros membros do time precisam entender o que cada hook faz.
7. **Nao bloqueie sem motivo.** Hooks que bloqueiam devem ter justificativa clara.

---

## Resumo

> Hooks sao automacoes que reagem a eventos do ciclo de vida. Eles permitem **validar, logar, formatar, notificar e proteger** sem intervencao manual. A chave para bons hooks e serem **rapidos, especificos e transparentes** — o usuario deve saber que eles existem e o que fazem.
