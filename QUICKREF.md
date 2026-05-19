---
title: QUICKREF — Consulta Rápida
tags: [quickref, referencia-rapida]
created: 2026-05-19
status: active
type: permanent
---

# QUICKREF — Responde 80% das Perguntas em Uma Página

---

## 🚀 "Onde eu começo?"

| Preciso de | Link |
|-----------|------|
| **Info sobre API Cove** | [[Covebackup/INDEX]] |
| **Info sobre GLPI** | [[GLPI/00 - Índice]] |
| **Visão geral Portal ASV** | [[projects/portal-asv]] |
| **Mapa completo** | [[INDEX]] |

---

## 🔑 Constantes Críticas (Portal ASV)

| Constante | Valor |
|-----------|-------|
| `ASV_TECNOLOGIA_ID` | `00000000-0000-0000-0000-000000000001` |
| `ASV_SISTEMAS_ID` | `2e8b4589-876e-4869-84df-4a3d7d95e212` |
| `GRUPO_ASV_ID` | `00000000-0000-0000-0000-000000000000` |
| Entity GLPI Tecnologia | `266` (CLIENTES COM CONTRATO) |
| Entity GLPI Sistemas | `77` (ASV Sistemas) |

---

## 📡 API Cove — Resumo

### O que é
Sistema de backup em nuvem. O Portal ASV sincroniza dados via Edge Functions → Supabase → GLPI.

### Campos Principais Capturados
- `Status Code` (1-12): situação do backup
- `Last Session Time` (D09F15): última sessão
- `Erros por Data Source` (SK/FK/GK/JK): quais tipos falharam
- Mais → [[Covebackup/01-API-Fields]]

### Implementação Pronta
[[Covebackup/04-Sprint-1-Implementation]] — código SQL/TypeScript/React

### Roadmap
- **Sprint 1:** Status Code, Last Session Time, Erros
- **Sprint 2:** Storage Breakdown, Partner Hierarchy
- Mais → [[Covebackup/03-Opportunities]]

---

## 🎫 GLPI — Resumo

### O que é
Sistema de tickets, entidades, usuários, SLA/OLA. Fonte de verdade para chamados.

### Tabelas Essenciais
| Tabela | Propósito |
|--------|-----------|
| `glpi_tickets` | Chamados |
| `glpi_entities` | Clientes/Departamentos |
| `glpi_users` | Técnicos/Usuários |
| `glpi_slas` | SLAs (tempo de resolução) |
| `glpi_planning` | Apontamentos (planning) |

### Campos SLA Importantes
- `date` (criação), `due_date` (prazo), `closedate` (fechamento)
- `status`, `priority` (1=baixa, 5=alta)
- `glpi_tickets_users.type` (1=requestor, 2=observer, 4=assign)

### Documentação
- Tickets/SLA → [[GLPI/04 - Módulos/04.03 - Assistance]]
- Schema completo → [[GLPI/10 - Banco de Dados]]
- Termos → [[GLPI/11 - Glossário]]

---

## 💻 Portal ASV — Stack & Rotas

### Stack
- **Frontend:** React 18 + TypeScript
- **Backend:** Supabase Edge Functions + Postgres + pg_cron
- **UI:** shadcn/ui + Tailwind CSS
- **Query:** TanStack Query (React Query)
- **Auth:** Supabase Auth + SSO Microsoft M365

### Rotas Principais
```
/tickets/team-dashboard    → TeamDashboardPage.tsx
/tickets/workload          → TechnicianWorkloadPage.tsx
/tickets/goals             → TechnicianGoalsPage.tsx
/operations                → Dashboard Diretoria
/people/*                  → Módulo People OS (RH)
```

### Tabelas de Cache (Supabase)
- `glpi_tickets_cache` → Chamados sincronizados
- `glpi_users_cache` → Técnicos canônicos
- `glpi_planning_cache` → Apontamentos
- `technician_goals` → Metas por técnico

---

## 🔗 Módulos & Scripts

### Módulo: Backup Cove + GLPI
**Arquivo:** [[modules/modulo-backup-cove-glpi]]

**O que faz:** Sincroniza backups Cove → abre/fecha chamados GLPI automaticamente

**Arquitetura:** COVE API → Edge Functions → Supabase → GLPI API

### Script: SharePoint → OneDrive
**Arquivo:** [[scripts/migrar-sharepoint-onedrive]]

**O que faz:** Copia pastas SharePoint Riffel para OneDrive "arquivo morto"

**Pré-requisitos:** PowerShell 5.1+, PnP.PowerShell, MFA interativo

---

## 🔍 Termos GLPI ↔ Portal ASV

| GLPI | Portal ASV | Significado |
|------|-----------|-------------|
| `glpi_tickets` | Chamado | Ticket, issue |
| `glpi_users` | Técnico | Usuário atribuído |
| `glpi_entities` | Cliente/Empresa | Entidade GLPI |
| `glpi_planning` | Apontamento | Tempo gasto registrado |
| `glpi_slas` | SLA | Prazo de resolução |
| `due_date` | Vencimento | Quando deve ser resolvido |

---

## ⚡ Atalhos de Busca

| Procuro | Link | Tópico |
|--------|------|--------|
| Campos API Cove | [[Covebackup/01-API-Fields]] | D09F00, D09F15, etc |
| Code Cove Sprint 1 | [[Covebackup/04-Sprint-1-Implementation]] | SQL/TS/React |
| Schema GLPI | [[GLPI/10 - Banco de Dados]] | Tabelas, campos |
| SLA GLPI | [[GLPI/04 - Módulos/04.03 - Assistance]] | Regras, prazos |
| Rotas Portal | [[projects/portal-asv]] | /tickets/*, /operations, /people |

---

**Última atualização:** 2026-05-19
