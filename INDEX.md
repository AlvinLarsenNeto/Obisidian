---
title: INDEX — ASV Obsidian
tags: [index, home, obsidian]
created: 2026-05-19
status: active
type: permanent
---

# ASV — Base de Conhecimento

> **Leia [[QUICKREF]] primeiro** — respostas rápidas em uma página.

---

## 📌 Documentação de Referência

### API Cove (Backup em Nuvem)
- **Entrada:** [[Covebackup/INDEX]] — roadmap, campos, implementação
- **Para iniciantes:** [[Covebackup/00-QUICKREF]] (5 min)
- **Para implementar:** [[Covebackup/04-Sprint-1-Implementation]] (código pronto)
- **Arquitetura:** [[Covebackup/05-Architecture]]
- **Complementar:** [[modules/modulo-backup-cove-glpi]] — automação GLPI

### GLPI (Tickets, Entidades, SLA)
- **Entrada:** [[GLPI/00 - Índice]] — índice oficial
- **Começar aqui:** [[GLPI/04 - Módulos/04.03 - Assistance]] (tickets, SLA, OLA)
- **Banco de dados:** [[GLPI/10 - Banco de Dados]] (schema MySQL)
- **Glossário:** [[GLPI/11 - Glossário]] (GLPI ↔ Portal ASV)
- **Admin/Config:** [[GLPI/04 - Módulos/04.06 - Administration]] + [[GLPI/04 - Módulos/04.07 - Configuration]]

---

## 🏗️ Projeto Principal

### Portal ASV
- **Visão geral:** [[projects/portal-asv]] (stack, módulos, rotas, dados)
- **Frontend:** React 18 + TypeScript + shadcn/ui
- **Backend:** Supabase (Edge Functions + Postgres)
- **Auth:** Microsoft SSO (M365)

### Portal Motorista (/m)
- **Referência rápida:** [[FLEET_MOTORISTA]] ⭐ COMECE AQUI para /m
  - Componentes, bugs, fluxos, schema, próximas tarefas
- **Guia usuário:** [[FLEET_GUIA_USUARIO.md]] (PDF para motoristas)

---

## 📚 Módulos & Complementos

| Módulo | Arquivo | Descrição |
|--------|---------|-----------|
| **Backup Cove** | [[modules/modulo-backup-cove-glpi]] | Sincronização COVE → GLPI automática |

---

## 🛠️ Scripts Utilitários

| Script | Arquivo | Uso |
|--------|---------|-----|
| **SharePoint → OneDrive** | [[scripts/migrar-sharepoint-onedrive]] | Migração cross-tenant Riffel |

---

## 🎯 Buscar por Tópico

### GLPI — Campos, Schema, Entidades
→ [[GLPI/10 - Banco de Dados]] + [[GLPI/11 - Glossário]]

### SLA / OLA / Planejamento
→ [[GLPI/04 - Módulos/04.03 - Assistance]]

### API Cove — Campos Disponíveis
→ [[Covebackup/01-API-Fields]]

### Implementação Cove — Código Pronto
→ [[Covebackup/04-Sprint-1-Implementation]]

### Rotas e Páginas do Portal
→ [[projects/portal-asv#Rotas Principais]]

### Arquitetura Portal ASV
→ [[projects/portal-asv#Stack]] + [[projects/portal-asv#Fontes de Dados Operacionais]]

---

## 📂 Estrutura de Pastas

```
C:\Obsidian\ASV/
├── Covebackup/           ← API Cove (7 arquivos)
│   └── INDEX.md          ← COMECE AQUI para Cove
├── GLPI/                 ← GLPI (documentação oficial)
│   └── 00 - Índice.md    ← COMECE AQUI para GLPI
├── modules/
│   └── modulo-backup-cove-glpi.md
├── projects/
│   └── portal-asv.md
├── scripts/
│   └── migrar-sharepoint-onedrive.md
├── source/               ← Código-fonte local
└── source-git/           ← Código-fonte (git)
```

---

## ⚡ Checklist de Consulta Rápida

- [ ] Preciso de **info rápida**? → Leia [[QUICKREF]]
- [ ] Preciso entender **API Cove**? → Vá para [[Covebackup/INDEX]]
- [ ] Preciso entender **GLPI**? → Vá para [[GLPI/00 - Índice]]
- [ ] Preciso de **código pronto** Cove? → [[Covebackup/04-Sprint-1-Implementation]]
- [ ] Preciso de **schema GLPI**? → [[GLPI/10 - Banco de Dados]]
- [ ] Preciso de **rotas Portal**? → [[projects/portal-asv#Rotas Principais]]

---

**Última atualização:** 2026-05-19  
**Mantido por:** Claude
