# WhatsApp Utility Templates — Guia Completo para Mensagens Dinâmicas

## Contexto

Ao usar a **WhatsApp Business API (Cloud API)**, mensagens proativas (business-initiated) precisam de **templates aprovados pela Meta**. Templates de **utility** são mais baratos que marketing e servem para: confirmações, lembretes, atualizações de pedido, alertas de conta, etc.

Este guia foca em como enviar **conteúdo dinâmico de tamanho variável** (ex: lista de lembretes que muda por usuário) dentro de um único template utility.

---

## 1. Regras Fundamentais

### 1.1 Categorias de template

| Categoria | Uso | Custo |
|---|---|---|
| **Utility** | Transações, lembretes, atualizações, alertas | Mais barato |
| **Marketing** | Promoções, ofertas, novidades | Mais caro (~2x utility) |
| **Authentication** | OTPs / códigos de verificação | Variável |

> **A Meta classifica o template automaticamente** com base no conteúdo. Se o texto parecer promocional, será classificado como marketing mesmo que você selecione utility.

### 1.2 Limites técnicos do template

| Recurso | Limite |
|---|---|
| Caracteres no **body** | **1024 chars** (texto fixo + variáveis preenchidas) |
| Caracteres no **header** (texto) | 60 chars |
| Caracteres no **footer** | 60 chars |
| Número de variáveis por seção | Até **99** (`{{1}}` a `{{99}}`) |
| Quebra de linha | Suportada via `\n` |
| Formatação | `*negrito*`, `_itálico_`, `~tachado~`, `` ```código``` `` |

### 1.3 O que conta no limite de 1024

O limite é do **body renderizado final**, ou seja:

```
texto fixo do template + conteúdo das variáveis preenchidas = máx 1024 chars
```

Exemplo: se o texto fixo tem 120 chars, sobram **~900 chars** para as variáveis.

---

## 2. Estratégia: Lista Dinâmica em Uma Variável

### 2.1 O problema

Templates têm variáveis **fixas** (`{{1}}`, `{{2}}`). Não existe loop ou variável dinâmica. Se um user tem 3 lembretes e outro tem 15, o template é o mesmo.

### 2.2 A solução

Montar toda a lista como **uma única string** antes de enviar, e injetar numa variável só.

**Template aprovado:**

```
Olá {{1}}, seus lembretes para hoje:

{{2}}

Responda se precisar de ajuda.
```

- `{{1}}` = nome do usuário
- `{{2}}` = string com toda a lista já formatada

**Exemplo de `{{2}}` preenchido:**

```
1. Reunião com cliente às 9h
2. Enviar relatório mensal
3. Revisar contrato do fornecedor
4. Ligar para suporte técnico
```

O WhatsApp renderiza as quebras de linha normalmente.

---

## 3. Falhas Conhecidas e Como Contornar

### 3.1 Estouro de caracteres (>1024)

**Problema:** Usuário com muitos itens ou itens com descrições longas estoura o limite.

**Solução — Truncar com aviso:**

```javascript
// Code Node no n8n
const MAX_CHARS = 900;
const lembretes = $input.all();
let texto = '';

for (let i = 0; i < lembretes.length; i++) {
  const linha = `${i + 1}. ${lembretes[i].json.descricao}\n`;

  if ((texto + linha).length > MAX_CHARS) {
    const restante = lembretes.length - i;
    texto += `\n... e mais ${restante} item(ns). Responda "ver tudo" para a lista completa.`;
    break;
  }
  texto += linha;
}

return [{ json: { listaFormatada: texto.trim() } }];
```

> **Dica:** Ao truncar, convide o usuário a responder. Isso abre a **janela de serviço (24h)** e você pode mandar o restante como mensagem free-form **gratuita**.

**Solução alternativa — Dividir em múltiplas mensagens:**

Se o usuário realmente precisa ver tudo, divida em blocos de ~900 chars. Porém, **cada envio de template é uma conversa cobrada** se não houver janela aberta. Use com cuidado.

### 3.2 Template classificado como Marketing pela Meta

**Problema:** Você submete como utility mas a Meta reclassifica como marketing (mais caro).

**Como evitar — Regras para aprovação como Utility:**

1. **O texto deve estar ligado a uma ação/transação existente do usuário**
   - Bom: "Seus lembretes para hoje" (o user cadastrou os lembretes)
   - Ruim: "Veja o que preparamos para você" (parece promoção)

2. **Não use linguagem promocional**
   - Evite: "aproveite", "oferta", "exclusivo", "imperdível", "desconto"
   - Use: "lembrete", "atualização", "confirmação", "agendamento"

3. **Não inclua CTA de venda**
   - Evite: "Compre agora", "Saiba mais sobre nossos planos"
   - Pode: "Responda se precisar de ajuda"

4. **Seja direto e informativo**
   - O template deve **informar**, não **persuadir**

**Exemplo de template que passa como Utility:**

```
Olá {{1}}, aqui está seu resumo diário:

{{2}}

Se houver algum erro, responda esta mensagem.
```

**Exemplo que será reclassificado como Marketing:**

```
Olá {{1}}! 🎉 Confira as novidades que separamos para você:

{{2}}

Não perca! Acesse agora e aproveite.
```

### 3.3 Variável vazia / usuário sem itens

**Problema:** Se o user não tem lembretes, enviar o template com `{{2}}` vazio fica estranho.

**Solução:** No n8n, usar um nó **IF** antes do envio:

```
IF → $json.listaFormatada is not empty
  → Sim: envia template
  → Não: não envia (ou envia template diferente)
```

### 3.4 Quebra de linha não renderiza

**Problema:** Às vezes `\n` é enviado como texto literal em vez de quebra de linha.

**Causas comuns:**

- Escapamento duplo: `\\n` em vez de `\n`
- Enviar via JSON sem parsear a string corretamente

**Solução no Code Node:**

```javascript
// Garanta que é um \n real, não escaped
const texto = lembretes
  .map((item, i) => `${i + 1}. ${item.json.descricao}`)
  .join('\n');

// NÃO faça JSON.stringify() no texto antes de passar pro template
return [{ json: { listaFormatada: texto } }];
```

### 3.5 Formatação rica limitada

**Problema:** WhatsApp templates suportam apenas formatação básica.

**O que funciona:**

```
*negrito*
_itálico_
~tachado~
```

**O que NÃO funciona:**

- Markdown completo (headers, links em markdown, etc.)
- HTML
- Bullet points (•) — use numeração ou hífen (-)
- Emojis funcionam mas evite no template utility (pode ser reclassificado)

---

## 4. Como Criar o Template na Meta

### 4.1 Passo a passo

1. Acesse o **Meta Business Suite** → **WhatsApp Manager**
2. Vá em **Ferramentas da conta** → **Modelos de mensagem**
3. Clique em **Criar modelo**
4. Preencha:
   - **Categoria:** `Utility`
   - **Nome:** `lembrete_diario` (snake_case, sem espaços)
   - **Idioma:** `Português (BR)`
5. No **Body**, escreva:

```
Olá {{1}}, seus lembretes para hoje:

{{2}}

Responda se precisar alterar algo.
```

6. Adicione **exemplos** para cada variável (obrigatório):
   - `{{1}}` → `João`
   - `{{2}}` → `1. Reunião às 9h\n2. Enviar relatório\n3. Revisar contrato`
7. Submeta para aprovação (geralmente leva minutos a poucas horas)

### 4.2 Dicas para aprovação rápida

- Forneça exemplos realistas e claros
- Mantenha tom informativo, não promocional
- Inclua frase de saída: "responda PARAR para não receber mais"
- Não use CAPS LOCK no texto fixo

---

## 5. Fluxo Completo no n8n

```
Schedule Trigger (ex: todo dia 8h)
    │
    ▼
Banco/API ── busca users ativos com lembretes
    │
    ▼
SplitInBatches ── 1 user por vez
    │
    ▼
Banco/API ── busca lembretes DO user
    │
    ▼
Code Node ── monta string formatada (com truncamento)
    │
    ▼
IF ── tem lembretes?
    │         │
    ▼         ▼
  Sim       Não → (ignora)
    │
    ▼
HTTP Request ── POST para WhatsApp Cloud API
```

### 5.1 HTTP Request para envio do template

**URL:**
```
https://graph.facebook.com/v21.0/SEU_PHONE_NUMBER_ID/messages
```

**Body (JSON):**

```json
{
  "messaging_product": "whatsapp",
  "to": "{{$json.telefone}}",
  "type": "template",
  "template": {
    "name": "lembrete_diario",
    "language": {
      "code": "pt_BR"
    },
    "components": [
      {
        "type": "body",
        "parameters": [
          {
            "type": "text",
            "text": "{{$json.nome}}"
          },
          {
            "type": "text",
            "text": "{{$json.listaFormatada}}"
          }
        ]
      }
    ]
  }
}
```

---

## 6. Resumo de Limites

| Item | Valor |
|---|---|
| Body total renderizado | 1024 chars |
| Margem segura para variáveis | ~900 chars |
| Quebra de linha | `\n` (funciona) |
| Formatação | `*negrito*`, `_itálico_`, `~tachado~` |
| Variáveis por seção | Até 99 |
| Custo utility vs marketing | Utility ~50% mais barato |
| Aprovação do template | Minutos a horas |
| Janela após envio proativo | 24h, mas SÓ permite templates |
| Janela de serviço (free-form) | Só se o USER responder primeiro |
