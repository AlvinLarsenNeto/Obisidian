---
title: Portal ASV
tags: [portal-asv, projeto, visao-geral]
created: 2026-04-16
updated: 2026-04-16
status: active
type: permanent
---

# Portal ASV

Sistema interno da ASV Tecnologia para gestão operacional, atendimento e RH.

## Stack
- **Frontend:** React 18 + TypeScript
- **Backend:** Supabase (Edge Functions + Postgres + pg_cron)
- **State:** TanStack Query (React Query)
- **UI:** shadcn/ui + Tailwind CSS
- **Auth:** Supabase Auth + Microsoft SSO (M365)

## Módulos Principais

### Assistência (GLPI)
- Lista de Chamados
- Planejamento
- Controle de IA

### Backup
- Monitoramento e abertura automática de chamados (Cove → GLPI)
- Ver: [[modules/modulo-backup-cove-glpi]]

### RH
- People OS — 56 tabelas, 10 edge functions, ciclo completo de colaborador

## Rotas Principais
- `/tickets/team-dashboard` → TeamDashboardPage.tsx
- `/tickets/workload` → TechnicianWorkloadPage.tsx
- `/tickets/goals` → TechnicianGoalsPage.tsx
- `/operations` → Dashboard operacional para diretoria (em construção)
- `/people/*` → Módulo People OS

## Fontes de Dados Operacionais
- `glpi_tickets_cache` — chamados sincronizados do GLPI
- `glpi_users_cache` — técnicos (lista canônica)
- `glpi_planning_cache` — apontamentos por técnico
- `technician_goals` — metas configuradas por técnico

## Referências
- [[modules/modulo-backup-cove-glpi]] — integração Cove + GLPI
