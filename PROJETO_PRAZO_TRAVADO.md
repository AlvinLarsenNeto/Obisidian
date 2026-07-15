---
title: Prazo final de projeto — travado + prorrogação com motivo
tags: [projects, prazo, deadline, glpi, trigger, bug-fix]
created: 2026-07-14
status: active
type: permanent
---

# Prazo Final de Projeto — Travado

> **TL;DR:** Prazo (`projects.plan_end_date`) sumia porque trigger `sync_project_dates_on_ticket_update` sobrescrevia com SLA do chamado GLPI vinculado a cada sync do cache. Fix 2026-07-14: prazo travado após definido; muda só via RPC `extend_project_deadline` (nova data + motivo ≥5 chars, histórico em `project_deadline_extensions`).

## Causa raiz (bug "data final não fica salva")

- Trigger em `glpi_tickets_cache` (migration `20260222130156`): a cada INSERT/UPDATE no cache, fazia `plan_end_date = COALESCE(ticket.time_to_resolve, ticket.sla_solution_due, ...)` em projetos com `linked_glpi_ticket_id`.
- Cache sincroniza periodicamente → prazo do usuário era sobrescrito "sozinho". Usuário percebia correlação com edição de tarefa, mas tarefas nunca gravaram prazo do projeto (frontend só grava `percent_done`).
- Edge fn `glpi-projects` já preservava edits locais (`existingProject?.plan_end_date ?? ...`) — não era o culpado.

## Solução (migration `20260714120000_project_deadline_lock.sql`)

1. **`project_deadline_extensions`** — histórico: old/new date, reason (CHECK ≥5 chars), created_by(_name). RLS SELECT via `get_accessible_companies`; INSERT só via função (SECURITY DEFINER).
2. **Trigger `protect_project_deadline`** (BEFORE UPDATE em `projects`): se `OLD.plan_end_date IS NOT NULL` e mudou → EXCEPTION, exceto com `current_setting('app.allow_deadline_change')='on'` (setado só dentro do RPC, transaction-local).
3. **RPC `extend_project_deadline(project_id, new_date, reason)`**: valida permissão + motivo + `new_date > atual`, atualiza e loga.
4. **Trigger GLPI reescrito**: agora só **preenche datas NULL**, nunca sobrescreve (start e end).

## Frontend

- `EditProjectModal.tsx`: prazo definido → campo desabilitado (ícone Lock) + botão **Prorrogar** → `ExtendDeadlineDialog.tsx` (nova data com min = prazo atual, motivo, lista de prorrogações anteriores). Submit **omite** `plan_end_date` quando travado.
- `ExtendDeadlineDialog` usa `(supabase as any)` (padrão do repo p/ tabelas fora dos types gerados).

## Gotchas

- **Migration precisa ser aplicada no Supabase** (`vaklqjseqtoseqhaugpe`) via SQL editor Lovable — MCP `portal-asv-ro` caiu de novo (2026-07-14: "tenant/user not found", credencial `claude_external_ro` morta). Sem migration, front funciona mas prazo continua vulnerável ao sync GLPI.
- Enquanto types não forem regenerados, `project_deadline_extensions`/RPC ficam com cast `as any`.
- Qualquer UPDATE futuro em `projects` que mande `plan_end_date` diferente vai dar EXCEPTION `P0001` — é intencional; usar o RPC.
- Primeiro set do prazo (NULL → data) continua livre (criar/editar projeto sem prazo).

Relacionado: [[OPS_QUICKREF]], [[projects/portal-asv]]
