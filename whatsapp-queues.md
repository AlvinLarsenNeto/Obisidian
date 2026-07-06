---
title: WhatsApp Queues — Portal ASV
tags: [whatsapp, filas, atendimento, horarios]
created: 2026-07-06
updated: 2026-07-06
status: active
type: permanent
---

# WhatsApp Queues — Portal ASV

Portal ASV WhatsApp integra **3 filas de atendimento** com horários comerciais distintos.

## Filas Ativas

### 1. Fila Sistemas
- **Horário comercial:** 07:30–12:00 e 13:00–17:18 (BRT)
- **Mensagem fora do horário:** "Fora do horário comercial. Sistemas: 07:30-12h, 13-17:18. Voltamos em breve."
- **Operadores:** Jhenifer Lamim (confirmado em casos)
- **Contatos:** Alisson (47 99160-7080), outros
- **Status:** Ativo

### 2. Fila Tecnologia
- **Horário comercial:** 07:00–12:00 e 13:00–17:48 (BRT)
- **Mensagem fora do horário:** "Fora do horário comercial. Tecnologia: 07-12h, 13-17:48. Voltamos em breve."
- **Operadores:** (TBD — validar)
- **Contatos:** Alisson (47 99160-7080), outros
- **Status:** Ativo

### 3. Fila Inteligência de Dados
- **Horário comercial:** (não especificado — validar em prod)
- **Mensagem fora do horário:** (herda regra geral ou custom?)
- **Operadores:** Mayara (confirmado em casos)
- **Contatos:** Alisson (47 99160-7080), outros
- **Status:** Ativo

---

## Isolamento por Fila (v2.1)

Cada contato pode ter **conversas separadas** por fila:
- Alisson em Sistemas = conversa A (28 msgs, Jhenifer)
- Alisson em Tecnologia = conversa B (tbd)
- Alisson em Inteligência de Dados = conversa C (nova, Mayara)

**Regra:** Operador de uma fila NÃO vê histórico de outra fila. RLS + 4-tupla `(contact_id, company_id, channel_id, queue_id)` enforça isolamento.

Ver: [[../OPS_QUICKREF#Desambiguação de repo]] para WhatsApp uazapi vs whatscrmsac.

---

## Auto-responder "Fora do Horário"

**Feature:** Mensagem automática quando contato escreve fora do expediente.

**Implementação:**
- Arquivo: `supabase/functions/_shared/business-hours.ts`
- Dedup: 4h por conversa (evita spam)
- Envio: UaZapi direto, sem persistir em histórico
- Ativado: 2026-06-30 (test com 4733513901)

---

## Horários BRT (Timezone)

Todos os horários em **America/Sao_Paulo** (UTC-3/-2 conforme DST).

Conversão:
- 07:30 BRT = 10:30 UTC (inverno) / 09:30 UTC (verão)
- 17:18 BRT = 20:18 UTC (inverno) / 19:18 UTC (verão)

---

## Referências

- **OPS_QUICKREF:** [[../OPS_QUICKREF#Desambiguação de repo]]
- **Isolamento fila:** `whatsapp_conversations` 4-tupla + RLS
- **Business hours:** `_shared/business-hours.ts`
- **Bugs resolvidos:** Alisson cross-fila (2026-06-25), @mention em grupo (2026-06-16)

---

**Status:** Documentação completa para v2.1 + auto-responder (2026-07-06)
