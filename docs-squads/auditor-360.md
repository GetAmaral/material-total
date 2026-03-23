# auditor-360

**Squad de Auditoria End-to-End do Total Assistente**

---

## Resumo

O auditor-360 testa as funcionalidades do Total Assistente **de ponta a ponta no ambiente DEV**. Ele envia mensagens reais via webhook, captura a resposta da IA, e cruza com o banco de dados (Supabase) para verificar se o que a IA disse ao usuario corresponde ao que foi realmente persistido.

**Em uma frase:** Dispara teste no DEV, verifica no banco, diagnostica se falhou.

---

## Identidade

| Campo | Valor |
|-------|-------|
| Nome | auditor-360 |
| Versao | 1.0.0 |
| Criado em | 19/Mar/2026 |
| Modo | **DEV-ONLY** (read-write no DEV, bloqueio total em producao) |
| Agentes | 2 (Lupa + Detetive) |
| Tasks | 28 funcionalidades em 7 categorias |
| Workflows | 0 (execucao manual por task) |
| Dependencia | testador-n8n |

---

## O que ele FAZ

1. **Le a metodologia** do .md da funcionalidade a ser testada
2. **Envia mensagem** via webhook DEV (`http://76.13.172.17:5678/webhook/dev-whatsapp`)
3. **Captura a resposta da IA** via polling em `log_users_messages`
4. **Cruza com o banco** — verifica se os dados foram persistidos corretamente (valor, categoria, data, campos)
5. **Se FAIL:** executa protocolo de diagnostico em 5 camadas antes de reportar
6. **Gera relatorio** com evidencias (snapshot antes/depois)

---

## O que ele NAO faz

- NAO acessa producao (188.245.190.178 e bloqueado)
- NAO audita usuarios reais (isso e o auditor-real)
- NAO faz testes rapidos ad-hoc (isso e o testador-n8n)
- NAO investiga infraestrutura (isso e o analisador-n8n)

---

## Agentes

### Lupa (auditor)
- **Papel:** End-to-End Audit Specialist
- **O que faz:** Executa as 28 funcionalidades seguindo a metodologia de cada .md, verifica no banco, diagnostica falhas
- **Arquetipo:** Inspector (Virgo)
- **Principio central:** "Verificacao REAL — SEMPRE cruzar resposta da IA com estado do banco"

### Detetive (investigador)
- **Papel:** Deep Error Investigator
- **O que faz:** Investiga UM erro por vez em profundidade (execucao N8N nodo por nodo, banco, edge functions)
- **Arquetipo:** Detective (Scorpio)
- **Principio central:** "Toda informacao tem fonte. Toda conclusao tem evidencia."

---

## As 28 Funcionalidades (7 Categorias)

### Financeiro (4)
| # | Funcionalidade | Arquivo |
|---|----------------|---------|
| 01 | Despesas e Receitas (CRUD) | `tasks/financeiro/01-despesas-receitas.md` |
| 02 | Limites por Categoria | `tasks/financeiro/02-limites-categoria.md` |
| 03 | Metas Financeiras | `tasks/financeiro/03-metas-financeiras.md` |
| 04 | Limite Mensal de Gasto | `tasks/financeiro/04-limite-mensal-gasto.md` |

### Agenda (7)
| # | Funcionalidade | Arquivo |
|---|----------------|---------|
| 01 | Agendamento Proprio | `tasks/agenda/01-agendamento-proprio.md` |
| 02 | Consulta de Compromissos | `tasks/agenda/02-consulta-compromissos.md` |
| 03 | Modificacao de Compromissos | `tasks/agenda/03-modificacao-compromissos.md` |
| 04 | Exclusao de Compromissos | `tasks/agenda/04-exclusao-compromissos.md` |
| 06 | Lembretes Recorrentes | `tasks/agenda/06-lembretes-recorrentes.md` |
| 07 | Agenda Diaria Automatica | `tasks/agenda/07-agenda-diaria-automatica.md` |
| 08 | VIP Calendar | `tasks/agenda/08-vip-calendar.md` |

### Bot WhatsApp (6)
| # | Funcionalidade | Arquivo |
|---|----------------|---------|
| 01 | Roteador Principal | `tasks/bot-whatsapp/01-roteador-principal.md` |
| 02 | Fluxo Premium | `tasks/bot-whatsapp/02-fluxo-premium.md` |
| 03 | Fluxo Standard | `tasks/bot-whatsapp/03-fluxo-standard.md` |
| 04 | Transcricao de Audio | `tasks/bot-whatsapp/04-transcricao-audio.md` |
| 05 | OCR de Imagem/PDF | `tasks/bot-whatsapp/05-ocr-imagem-pdf.md` |
| 06 | Bot Guard | `tasks/bot-whatsapp/06-bot-guard.md` |

### Autenticacao (5)
| # | Funcionalidade | Arquivo |
|---|----------------|---------|
| 01 | Login OTP | `tasks/autenticacao/01-login-otp.md` |
| 02 | Google OAuth | `tasks/autenticacao/02-google-oauth.md` |
| 03 | 2FA Legado | `tasks/autenticacao/03-2fa-legado.md` |
| 04 | RBAC por Planos | `tasks/autenticacao/04-rbac-planos.md` |
| 05 | Gestao de Conta | `tasks/autenticacao/05-gestao-conta.md` |

### Pagamentos (3)
| # | Funcionalidade | Arquivo |
|---|----------------|---------|
| 01 | Hotmart Webhook | `tasks/pagamentos/01-hotmart-webhook.md` |
| 02 | Checkout de Planos | `tasks/pagamentos/02-checkout-planos.md` |
| 03 | Gestao de Assinatura | `tasks/pagamentos/03-gestao-assinatura.md` |

### Investimentos (1)
| # | Funcionalidade | Arquivo |
|---|----------------|---------|
| 01 | Portfolio | `tasks/investimentos/01-portfolio.md` |

### Relatorios (2)
| # | Funcionalidade | Arquivo |
|---|----------------|---------|
| 01 | Relatorio PDF via WhatsApp | `tasks/relatorios/01-relatorio-pdf-whatsapp.md` |
| 02 | Export no Frontend | `tasks/relatorios/02-export-frontend.md` |

---

## Protocolo de Diagnostico (5 Camadas)

Executado automaticamente quando um teste retorna FAIL ou PARTIAL:

| Camada | Nome | Verificacao |
|--------|------|-------------|
| 1 | CLASSIFICADOR | Escolher Branch decidiu a branch correta? |
| 2 | AI AGENT | Agent chamou as tools corretas? |
| 3 | TOOL HTTP | Webhook do workflow secundario retornou sucesso? |
| 4 | SUPABASE | Operacao no banco executou? RLS? Constraint? |
| 5 | RESPOSTA | IA formatou resposta corretamente? Dados batem com banco? |

**Causas possiveis:** CLASSIFICACAO_ERRADA, TOOL_NAO_CHAMADA, TOOL_FALHOU, BANCO_REJEITOU, RESPOSTA_ERRADA, ASYNC_INCOMPLETO, COMPORTAMENTO_NAO_DOCUMENTADO

---

## Niveis de Auditoria

| Nivel | Profundidade | Tempo estimado |
|-------|-------------|----------------|
| `quick` | Smoke test — funciona ou nao? | 2-5 min/feature |
| `broad` | Cenarios principais + edge cases criticos | 10-20 min/feature |
| `complete` | Todos os angulos, todos os campos | 30-60 min/feature |

---

## Metodologia de Verificacao (Checklist 360)

Cada teste segue 3 fases obrigatorias:

1. **SNAPSHOT ANTES** — Contar registros, salvar estado, registrar hora UTC
2. **EXECUTAR ACAO** — Enviar via webhook, aguardar, capturar resposta da IA
3. **SNAPSHOT DEPOIS + CRUZAMENTO** — Contar registros, comparar campo a campo com banco

Cruzamento obrigatorio: cada campo que a IA menciona na resposta e comparado com o valor real no Supabase.

---

## Acessos

| Recurso | Acesso | Endereco |
|---------|--------|----------|
| N8N DEV | read-write | `http://76.13.172.17:5678` |
| Webhook DEV | POST | `http://76.13.172.17:5678/webhook/dev-whatsapp` |
| Supabase Principal | read-write (teste) | `ldbdtakddxznfridsarn.supabase.co` |
| Supabase AI Messages | read-write | `hkzgttizcfklxfafkzfl.supabase.co` |
| Producao | **BLOQUEADO** | — |

**Usuario de teste:** Luiz Felipe (554391936205)

---

## Bugs Confirmados

| Bug | Severidade | Status |
|-----|-----------|--------|
| UPDATE de valor nao altera o banco | CRITICO | Documentado |
| UPDATE de nome nao altera o banco | CRITICO | Documentado |
| UPDATE de data de evento nao altera o banco | CRITICO | Documentado |
| Categoria da IA difere do banco | MEDIO | Documentado |
| Consulta de saldo ignora entradas | MEDIO | Documentado |
| Recorrente pode nao criar se combinada | BAIXO | Documentado |

---

## Relacao com Outros Squads

| Squad | Relacao |
|-------|---------|
| **testador-n8n** | Primo. Watson faz testes rapidos ad-hoc. Lupa faz auditoria sistematica. |
| **analisador-n8n** | Primo. Sherlock investiga producao. Lupa audita no DEV. |
| **auditor-real** | Complementar. auditor-real audita usuarios REAIS em producao (read-only). auditor-360 testa features no DEV. |

---

## Comandos Principais

```
*audit financeiro/01 --level broad    # Auditar uma funcionalidade
*audit-category financeiro            # Auditar categoria inteira
*audit-all --level quick              # Smoke test de tudo
*diagnose FIN-Q4                      # Diagnosticar teste que falhou
*status                               # Progresso da auditoria
*coverage                             # Cobertura por categoria
*bugs                                 # Listar bugs encontrados
*report all                           # Relatorio completo
*list                                 # Listar as 28 funcionalidades
```

---

## Output

```
squads/auditor-360/output/
├── logs/              # Logs de execucao de testes
├── errors/            # Documentos de erro (ERR-NNN)
├── investigations/    # Investigacoes profundas do Detetive
└── diagnostics/       # Diagnosticos automaticos (5 camadas)
```

---

*Squad auditor-360 v1.0.0 — DEV-ONLY | 28 features | 7 categorias*
