# Testes Auditor-360 — 23-24/03/2026

Auditoria automatizada end-to-end do assistente WhatsApp (N8N DEV).
97 testes em 3 fases. Taxa global: **80%** (78/97 PASS).

---

## Estrutura

```
testes-23-03-2026/
├── README.md                          ← Este arquivo
├── 01-quick-smoke/
│   └── RELATORIO.md                   ← Fase 1: 33 testes básicos (85%)
├── 02-broad/
│   └── RELATORIO.md                   ← Fase 2: 39 testes edge cases (67%)
├── 03-complete/
│   └── RELATORIO.md                   ← Fase 3: 25 testes stress/segurança (96%)
├── 04-consolidado/
│   └── RELATORIO-FINAL.md             ← Relatório final com catálogo de bugs
└── logs/
    ├── raw-outputs/
    │   ├── fase1-quick-rodada1.txt    ← Output bruto do script Python (rodada 1)
    │   ├── fase1-quick-rodada2.txt    ← Output bruto (rodada 2 — timeouts)
    │   └── fase2-broad-agenda-bot.txt ← Output bruto Broad agenda+bot
    ├── ai-messages/
    │   └── log-completo.md            ← 152 mensagens IA do período
    └── n8n-executions/
        └── executions-summary.md      ← 250 execuções N8N do período
```

## Como reproduzir

1. Garantir N8N DEV online: `curl http://76.13.172.17:5678/healthz`
2. Garantir user de teste ativo: plan_status=true no Supabase
3. Executar scripts Python em `/tmp/auditor_*.py` ou `/tmp/broad_*.py` ou `/tmp/fase3_complete.py`
4. Variáveis necessárias: `SUPABASE_PRINCIPAL_SERVICE_KEY`, `SUPABASE_AI_MESSAGES_SERVICE_KEY`, `N8N_DEV_API_KEY`

## Bugs encontrados

| # | Bug | Severidade |
|---|-----|-----------|
| B001 | Exclusão em massa financeira sem confirmação | CRITICO |
| B002 | Multi-turno financeiro não mantém contexto | MEDIO |
| B003 | Classificador rejeita "ontem" | MEDIO |
| B005 | Multi-turno exclusão não funciona | MEDIO |
| B004 | Conflito de horário não avisado | BAIXO |
| B006 | Gíria de receita não reconhecida | BAIXO |
| EDG-1 | Valor zero aceito | BAIXO |
