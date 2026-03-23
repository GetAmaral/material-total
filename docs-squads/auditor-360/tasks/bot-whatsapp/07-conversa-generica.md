# Metodologia de Auditoria — Conversa Genérica (Branch Padrão)

**Funcionalidade:** `bot-whatsapp/07-conversa-generica`
**Versão:** 1.0.0

---

## 1. Mapa do sistema

### O que é

Quando a mensagem do user NÃO se encaixa em nenhuma branch funcional (financeiro, agenda, relatório), o classificador (Escolher Branch) roteia para a branch `padrao`. Esta branch é a **porta de entrada** para a maioria dos users — "oi", "bom dia", "me ajuda" são as mensagens mais comuns.

### Caminho

```
WhatsApp → Main → Fix Conflito v2
  → Escolher Branch (classificador LLM)
    → Branch: padrao (default / catch-all)
      → prompt_padrao (set prompt)
      → AI Agent (GPT-4.1-mini) processa sem tools HTTP
      → Responde com texto genérico (saudação, ajuda, explicação)
  → Envia WhatsApp
  → Log em log_users_messages
```

### Comportamento esperado

| Input do user | O que deveria acontecer |
|--------------|------------------------|
| "oi" / "bom dia" / "eae" | Saudação + breve explicação do que o bot faz |
| "me ajuda" / "o que você faz?" | Listar funcionalidades disponíveis |
| "obrigado" / "valeu" | Resposta cordial, encerrar contexto |
| "hahaha" / "kkk" / "👍" | Resposta leve, não travar, não criar registro |
| Texto em inglês | Responder em português? Entender? Documentar |
| Mensagem sem sentido ("asdfghjkl") | Não travar, não criar registro indevido |

### Regra crítica

A branch `padrao` NÃO deve:
- Criar gasto no `spent`
- Criar evento no `calendar`
- Chamar nenhum workflow secundário (Financeiro, Calendar WebHooks, Report)
- Gerar relatório

É uma resposta PURAMENTE textual do AI Agent, sem side effects.

---

## 2. Algoritmo de execução

```
PASSO 1 — SNAPSHOT ANTES
  1.1  Contar spent → SPENT_ANTES
  1.2  Contar calendar → CAL_ANTES
  1.3  Último log_id → LAST_LOG_ID

PASSO 2 — ENVIAR MENSAGEM

PASSO 3 — POLLAR RESPOSTA (log_users_messages)

PASSO 4 — VERIFICAR ZERO SIDE EFFECTS
  4.1  Contar spent → SPENT_DEPOIS = SPENT_ANTES (nada criado)
  4.2  Contar calendar → CAL_DEPOIS = CAL_ANTES (nada criado)
  4.3  ai_message presente e coerente (não é erro, não é lixo)

PASSO 5 — REGISTRAR
```

---

## 3. Critérios de PASS/FAIL

| # | Critério | PASS | FAIL |
|---|----------|------|------|
| 1 | Resposta existe | ai_message não é null/vazia | Sem resposta (timeout) |
| 2 | Resposta coerente | Texto faz sentido como resposta ao input | Erro, JSON bruto, ou resposta de outra branch |
| 3 | Zero side effects | spent e calendar inalterados | Criou gasto, evento, ou chamou workflow secundário |
| 4 | Idioma correto | Resposta em português | Inglês ou outro idioma |
| 5 | Não confundiu com gasto | "oi" não virou gasto no banco | Registrou "oi" como transação |

---

## 4. Protocolo de diagnóstico de erros

```
CAMADA 1 — CLASSIFICADOR: Foi pra branch padrao?
  Se foi pra criar_gasto ou outra → CLASSIFICACAO_ERRADA
CAMADA 2 — PROMPT: prompt_padrao foi setado corretamente?
CAMADA 3 — AI AGENT: Respondeu sem chamar tools HTTP?
  Se chamou tool financeiro/calendar → TOOL_INDEVIDA
CAMADA 4 — RESPOSTA: Texto é coerente com o input?
CAMADA 5 — SIDE EFFECTS: Algo mudou no banco? (spent/calendar)
```

---

## 5. Testes

**🟢 Quick (4 testes):**

| ID | Input | Verificação |
|----|-------|-------------|
| GEN-Q1 | "oi" | Resposta é saudação. spent e calendar inalterados. Nenhum workflow secundário executou. |
| GEN-Q2 | "o que você faz?" | Resposta lista funcionalidades (gastos, agenda, lembretes, relatórios). Nada criado no banco. |
| GEN-Q3 | "obrigado" | Resposta cordial. Nada criado no banco. |
| GEN-Q4 | "hahaha" | Resposta leve ou vazia. Nada criado no banco. NÃO registrou "hahaha" como gasto. |

**🟡 Broad (Quick + 5 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| GEN-B1 | "bom dia, tudo bem?" | Saudação natural. Sem side effects. |
| GEN-B2 | "me ajuda" | Resposta com instruções de uso. Sem side effects. |
| GEN-B3 | "hello, how are you?" | Mensagem em inglês: IA responde em português? Entende? Documentar. |
| GEN-B4 | "asdfghjkl" | Garbage input: não travar, resposta educada pedindo reformulação. |
| GEN-B5 | "👍" | Emoji sozinho: sem crash, sem registro no banco. |

**🔴 Complete (Broad + 3 testes):**

| ID | Input | O que valida |
|----|-------|-------------|
| GEN-C1 | "oi" seguido de "gastei 30 no almoço" | Transição de padrão→financeiro. Segundo msg cria gasto corretamente. |
| GEN-C2 | "quero saber como funciona o app" | Pergunta genérica longa. Resposta informativa sem side effects. |
| GEN-C3 | "🤣🤣🤣" | Múltiplos emojis: não travar, não interpretar como valor/nome. |

---

## 6. Formato do log

```markdown
| ID | Input | IA respondeu | spent mudou? | calendar mudou? | Veredicto |
```

---

## 7. Melhorias sugeridas

| O que | Impacto |
|-------|---------|
| Menu estruturado no "oi" | User novo sabe o que fazer |
| Detecção de idioma | Responder no idioma do user ou avisar que só fala PT-BR |
| Logar branch=padrao no log_total | Confirmar classificação sem acessar execução N8N |
