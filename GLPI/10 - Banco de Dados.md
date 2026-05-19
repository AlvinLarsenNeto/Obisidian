---
title: GLPI - Banco de Dados
source: https://dbdocs.glpi-project.org/
updated: 2026-04-20
status: oficial
area: glpi
tags: [glpi, banco-de-dados, schema, mysql]
---

# GLPI — Banco de Dados

## Fontes oficiais
- Schema interativo: https://dbdocs.glpi-project.org/
- GLPI 10: https://dbdocs.io/glpi/GLPI-10
- GLPI 11: https://dbdocs.io/glpi/GLPI-11
- Diferenças 10 → 11: https://dbdocs.io/glpi/GLPI-10-to-11

> Os diagramas são gerados automaticamente e podem conter erros. Validar sempre em ambiente real antes de usar para integração ou consultas críticas.

---

## Tabelas principais relevantes para o Portal ASV

### Tickets
| Tabela | Descrição |
|---|---|
| `glpi_tickets` | Tabela principal de tickets |
| `glpi_tickets_users` | Relação ticket ↔ usuário (requester, watcher, assigned) |
| `glpi_tickettasks` | Tarefas/apontamentos vinculados ao ticket |
| `glpi_ticketfollowups` | Acompanhamentos/comentários |
| `glpi_ticketsatisfactions` | Avaliações de satisfação |
| `glpi_slas` | Definições de SLA |
| `glpi_slms` | Service Level Management (agrupa SLAs) |
| `glpi_slalevels` | Níveis e escalonamentos de SLA |

### Campos de SLA em `glpi_tickets`
| Campo | Tipo | Descrição |
|---|---|---|
| `date` | datetime | Abertura do ticket |
| `time_to_resolve` | datetime | Prazo calculado de resolução (já com calendário aplicado) |
| `solvedate` | datetime | Data/hora efetiva de resolução |
| `time_to_own` | datetime | Prazo calculado para assumir o ticket |
| `takeintoaccount_delay_stat` | int | Segundos até ser assumido |
| `solve_delay_stat` | int | Segundos até resolução |
| `sla_id_tto` | int | FK para `glpi_slas` (tipo TTO) |
| `sla_id_ttr` | int | FK para `glpi_slas` (tipo TTR) |
| `ola_id_tto` | int | FK OLA TTO |
| `ola_id_ttr` | int | FK OLA TTR |

### Usuários
| Tabela | Descrição |
|---|---|
| `glpi_users` | Usuários do GLPI |
| `glpi_groups` | Grupos |
| `glpi_groups_users` | Relação grupo ↔ usuário |
| `glpi_profiles` | Perfis de acesso |
| `glpi_profiles_users` | Relação perfil ↔ usuário ↔ entidade |

### Entidades e assets
| Tabela | Descrição |
|---|---|
| `glpi_entities` | Entidades (empresas/departamentos) |
| `glpi_computers` | Computadores inventariados |
| `glpi_networkequipments` | Equipamentos de rede |
| `glpi_software` | Software inventariado |

### Planning / Apontamentos
| Tabela | Descrição |
|---|---|
| `glpi_tickettasks` | Tarefas com campo `actiontime` (segundos) |
| `glpi_planningexternalevents` | Eventos externos no planejamento |

---

## Convenções no cache ASV (`glpi_tickets_cache`)
- `assigned_to` = texto com nome completo → join via `glpi_users_cache.full_name`
- `ecosystem` = preenchido por trigger `glpi_tickets_cache_fill_ecosystem` (não pela sync)
- `actiontime` em segundos → dividir por 60 para comparar com `technician_goals.daily_goal_minutes`
- Usar `task_date` (não `begin_time`) para filtrar período em `glpi_planning_cache`

---

## Notas para expandir por domínio
- [ ] Tickets e SLA (campos completos, relacionamentos)
- [ ] Usuários e perfis
- [ ] Entidades e hierarquia
- [ ] Assets e inventário
- [ ] Contratos e fornecedores
- [ ] Notificações e fila
- [ ] Logs e auditoria
