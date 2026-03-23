# Teoria dos Squads — Manual Completo do Synkra AIOS

Um guia completo e didatico sobre todos os conceitos fundamentais do Synkra AIOS: Synapse, Agent, Skill, Squad, Workflow e Hook.

---

## Indice

### Conceitos Individuais (Deep Dive)

| # | Documento | Conceito | Descricao |
|---|-----------|----------|-----------|
| 01 | [SYNAPSE](01-SYNAPSE.md) | O Cerebro | Motor de contexto inteligente com pipeline de 8 camadas |
| 02 | [AGENT](02-AGENT.md) | O Especialista | Persona IA com expertise, autoridade e restricoes |
| 03 | [SKILL](03-SKILL.md) | A Capacidade | Modulo de conhecimento reutilizavel por agentes |
| 04 | [SQUAD](04-SQUAD.md) | O Time | Pacote modular de agentes, tasks e workflows por dominio |
| 05 | [WORKFLOW](05-WORKFLOW.md) | A Orquestracao | Sequencia coordenada de etapas com paralelismo e gates |
| 06 | [HOOK](06-HOOK.md) | O Gatilho | Automacao que reage a eventos do ciclo de vida |

### Guias de Integracao

| # | Documento | Descricao |
|---|-----------|-----------|
| 07 | [GUIA COMPLETO — Relacoes](07-GUIA-COMPLETO-RELACOES.md) | Como todos os conceitos se conectam, fluxo completo, analogias, anti-patterns |
| 08 | [GUIA PARA INICIANTES](08-GUIA-PARA-INICIANTES.md) | Explicacao simples e didatica para quem nunca usou o AIOS |

---

## Como Usar Este Material

- **Se voce e iniciante:** Comece pelo [Guia para Iniciantes](08-GUIA-PARA-INICIANTES.md)
- **Se quer entender um conceito:** Leia o documento individual (01-06)
- **Se quer ver como tudo se conecta:** Leia o [Guia Completo](07-GUIA-COMPLETO-RELACOES.md)
- **Se quer criar algo novo:** Cada documento individual tem secoes de "Casos de Uso" e "Area de Criatividade"

---

## Resumo Visual

```
SQUAD (Time)
  |-- AGENT (Especialista)
  |     |-- SKILL (Ferramenta)
  |
  |-- WORKFLOW (Processo)
  |     |-- TASK (Passo)
  |           |-- GATE (Checkpoint)
  |
  |-- SYNAPSE (Cerebro/Contexto)
  |     |-- L0-L7 (8 Camadas)
  |
  |-- HOOK (Automacao)
        |-- Eventos (Pre/Post)
```

---

*Material produzido com base no estudo completo do Synkra AIOS Core v4.2.13*
