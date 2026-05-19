---
title: GLPI - Glossário
source: https://help.glpi-project.org/documentation/glossary
updated: 2026-04-20
status: oficial
area: glpi
tags: [glpi, glossario, termos]
---

# GLPI — Glossário

## Fonte oficial
- https://help.glpi-project.org/documentation/glossary

## Termos padronizados

| Termo GLPI | Tradução / uso interno ASV | Observação |
|---|---|---|
| Ticket | Chamado | Tipo: Incident ou Request |
| Incident | Incidente | Algo parou de funcionar |
| Request | Requisição / Solicitação | Pedido de novo serviço ou item |
| Problem | Problema | Agrupa incidentes com mesma causa raiz |
| Change | Mudança | Alteração formal na infraestrutura |
| Entity | Entidade | Empresa/departamento no GLPI |
| Profile | Perfil | Define o que o usuário pode fazer |
| Rule | Regra | Automação de atribuição/categorização |
| Receiver | Receptor | Coleta e-mails para abertura de tickets |
| Automatic action | Ação automática | Cron job interno do GLPI |
| Asset | Ativo | Hardware, software, rede, etc. |
| Requester | Solicitante | Quem abriu o chamado |
| Watcher | Observador | Recebe notificações sem agir |
| Technician | Técnico | Usuário do GLPI com perfil de suporte |
| SLA | Acordo de Nível de Serviço (externo) | Prazo com o cliente |
| OLA | Acordo de Nível Operacional (interno) | Prazo entre equipes internas |
| TTO | Time to Own | Tempo até ser assumido por técnico |
| TTR | Time to Resolve | Tempo até resolução do chamado |
| Escalation | Escalonamento | Ação automática quando SLA está próximo de vencer |
| Knowledge base | Base de conhecimento | Artigos de solução para consulta |
| Planning | Apontamento / Planejamento | Registro de tempo trabalhado num ticket |
| Actiontime | Tempo de ação | Duração de um apontamento, em **segundos** no banco |
| Dropdown | Lista suspensa | Campos de seleção configuráveis no GLPI |
| Dictionary | Dicionário | Regras de normalização de dados importados |

## Notas
- No Portal ASV, `assigned_to` é texto (nome completo) — diferente do GLPI onde é FK para `glpi_users`
- Confirmar sempre a versão do GLPI em uso para validar campos e comportamentos
