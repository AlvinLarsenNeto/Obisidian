# 🔐 COVEBACKUP - MÓDULO BACKUP EM NUVEM

> **Central de referência para o módulo Backup-Verification (Cove)**  
> Última atualização: 2026-05-05

> ⚠️ **DESATUALIZADO (verificado 2026-05-29 contra GitHub `origin/main`).** Estes 6 docs
> refletem o estado de ~abril e o roadmap planejado. Já mudou na prática:
> - **Sprint 1 NÃO está mais "não implementada"** — `last_session_status_code` (D09F00) foi
>   capturado e a **matriz `should_open_incident`** decide abertura (atrás da flag
>   `use_d09f00_matrix`). Ver [[../modules/modulo-backup-cove-glpi]] para o estado real.
> - Existem **10 edge functions backup-*** (não 4) e **8 tabelas novas** (feature_flags, fila
>   de e-mail, relatório semanal, etc).
> - Cron de `backup-auto-tickets` é **diário 04:00 UTC**, não ~5min.
> Use [[../modules/modulo-backup-cove-glpi]] como fonte atual; estes docs ficam como histórico
> do mapeamento de campos da API (que continua válido).

---

## 📌 INÍCIO RÁPIDO

### Para Iniciantes
1. Leia: [[00-QUICKREF]] (5 min)
2. Veja: [[02-Current-Implementation]] (10 min)
3. Entenda: [[01-API-Fields]] (15 min)

### Para Implementação
1. Scope: [[03-Opportunities]] (Sprint 1/2/3)
2. Code: [[04-Sprint-1-Implementation]] (pronto para usar)
3. Deploy: [[05-Architecture]] (fluxos)

---

## 📚 DOCUMENTAÇÃO COMPLETA

### 📖 Documentos Principais

| Doc | Propósito | Tamanho | Leitura |
|-----|-----------|---------|---------|
| [[00-QUICKREF]] | Tabela comparativa campos | 2 KB | 3 min |
| [[01-API-Fields]] | 112+ campos da API mapeados | 15 KB | 15 min |
| [[02-Current-Implementation]] | O que o módulo usa hoje | 8 KB | 10 min |
| [[03-Opportunities]] | Oportunidades Sprint 1/2/3 | 20 KB | 20 min |
| [[04-Sprint-1-Implementation]] | Código SQL/TS/React pronto | 25 KB | 30 min |
| [[05-Architecture]] | Diagramas visuais e fluxos | 18 KB | 15 min |

---

## 🎯 BUSCAR POR TÓPICO

### Status Code (D09F00)
- O que é: [[01-API-Fields#Status Code]]
- Implementar: [[04-Sprint-1-Implementation#Capturar Status Code]]
- Benefícios: [[03-Opportunities#A. Capturar Status Code]]

### Erros por Data Source (SK/FK/GK/JK)
- O que é: [[01-API-Fields#Erros por Data Source]]
- Implementar: [[04-Sprint-1-Implementation#Extrair Erros por Source]]
- Benefícios: [[03-Opportunities#C. Extrair Erros por Data Source]]

### Last Session Time (D09F15)
- O que é: [[01-API-Fields#Last Session Time]]
- Implementar: [[04-Sprint-1-Implementation#Capturar Last Session Time]]
- Detectar hanging: [[04-Sprint-1-Implementation#Detectar Hanging Sessions]]

### Storage Breakdown (D01F03...D20F03)
- O que é: [[01-API-Fields#Tamanho por Data Source]]
- Implementar: [[03-Opportunities#Sprint 2 - Storage Breakdown]]
- Esquema: [[05-Architecture#Tabela backup workload details]]

### Partner Hierarchy
- O que é: [[01-API-Fields#Partner Name]]
- Implementar: [[03-Opportunities#Sprint 2 - Partner Hierarchy]]
- Resolução: [[04-Sprint-1-Implementation#Partner Resolution]]

---

## 🚀 ROADMAP

### ✅ SPRINT 1 (1-2 semanas)
```
Capturar:
  ☐ Status Code (D09F00)
  ☐ Last Session Time (D09F15)
  ☐ Erros por Data Source (SK/FK/GK/JK)
  
UI:
  ☐ Badges com status code
  ☐ Badge com failed sources
  ☐ Filtro "Hanging", "Over Quota"
```

Veja: [[03-Opportunities#Sprint 1]] | [[04-Sprint-1-Implementation]]

### ⏳ SPRINT 2 (2-4 semanas)
```
Capturar:
  ☐ Tamanho por Data Source (D01F03...D20F03)
  ☐ Partner Hierarchy (PN, ParentId)
  
UI:
  ☐ Storage breakdown (pie chart)
  ☐ Activity timeline
  ☐ Data source health matrix
```

Veja: [[03-Opportunities#Sprint 2]]

### 🎁 BACKLOG
```
  ☐ Session history (requer permissão Cove)
  ☐ SLA Dashboard
  ☐ Forecasting & capacity planning
  ☐ Real-time webhooks
```

---

## 📊 TABELAS RÁPIDAS

### Campos Capturados vs Não Capturados
Veja: [[00-QUICKREF#Comparação Rápida]]

### Schema Atual vs Proposto
Veja: [[05-Architecture#Database Schema]]

### Dados da API vs UI
Veja: [[05-Architecture#Dados da API vs Banco vs UI]]

---

## 🔧 IMPLEMENTAÇÃO

### SQL
```sql
-- Ver: [[04-Sprint-1-Implementation#Schema Changes]]
ALTER TABLE backup_status_cache ADD COLUMN (
  last_session_status_code INT,
  failed_data_sources TEXT[],
  is_session_hanging BOOLEAN
);
```

### Edge Function (Deno/TypeScript)
```typescript
// Ver: [[04-Sprint-1-Implementation#Edge Function Changes]]
// File: supabase/functions/cove/index.ts
function processWorkload(...) {
  const sessionStatusCode = ...;
  const failedDataSources = ...;
}
```

### React/TypeScript (UI)
```typescript
// Ver: [[04-Sprint-1-Implementation#UI Changes]]
// File: src/components/dashboard/CoveBackupSection.tsx
<Badge variant={getStatusVariant(statusCode)}>
  {getStatusLabel(statusCode)}
</Badge>
```

---

## ✅ CHECKLIST DE REFERÊNCIA

### Antes de Implementar
- [ ] Li [[00-QUICKREF]] e entendo os campos
- [ ] Identifiquei qual Sprint vou implementar
- [ ] Revisei código de exemplo em [[04-Sprint-1-Implementation]]

### Durante Implementação
- [ ] Schema SQL criado e testado
- [ ] Edge function parseando novos campos
- [ ] UI renderizando badges novos
- [ ] Queries de alerta funcionando

### Depois de Deploy
- [ ] Sync de produção rodando sem erros
- [ ] Dados novos aparecendo no banco
- [ ] UI mostrando badges/filtros
- [ ] Alertas disparando em <5min

---

## 📞 REFERÊNCIAS EXTERNAS

### Código-Fonte
- **Edge Function:** `supabase/functions/cove/index.ts`
- **Hook:** `src/hooks/useCoveBackup.ts`
- **Components:** `src/components/dashboard/CoveBackupSection.tsx`
- **Page:** `src/pages/BackupsPage.tsx`

### Banco de Dados
- **Tabela principal:** `backup_status_cache`
- **Tabela de logs:** `backup_sync_logs`
- **Tabela de incidentes:** `backup_workload_incidents`
- **Tabela proposta:** `backup_workload_details` (Sprint 2)

### APIs
- **Cove API:** `https://api.backup.management/jsonrpcv1`
- **Portais:** Device Portal + GB Portal (dual)

---

## 🎓 GLOSSÁRIO

| Termo | Significado | Referência |
|-------|-------------|-----------|
| **Workload** | Dispositivo/servidor sendo backupado | [[01-API-Fields#Identificação]] |
| **Data Source** | Tipo de backup (Files, Exchange, SQL, etc) | [[01-API-Fields#Erros por Data Source]] |
| **Status Code** | Código 1-12 descrevendo situação | [[01-API-Fields#Status Code]] |
| **Over Quota** | Status 10 = storage cheio | [[03-Opportunities#Over Quota]] |
| **Hanging** | Sessão rodando >12h | [[04-Sprint-1-Implementation#Detectar Hanging]] |
| **Portal** | Device ou GB (dual portal) | [[02-Current-Implementation#Portais]] |
| **Partner ID** | ID do cliente/reseller no Cove | [[01-API-Fields#Partner Name]] |

---

## 💾 VERSÃO & HISTÓRICO

| Versão | Data | Mudanças |
|--------|------|----------|
| 1.0 | 2026-05-05 | Criação inicial: 6 docs, roadmap Sprint 1-3 |

---

## 📝 NOTAS

- **Estrutura:** Documentos numerados (00-05) para leitura sequencial
- **Links:** Use `[[filename#section]]` para navegar
- **Tamanho:** Cada doc ~5-25 KB, leitura <30 min
- **Atualização:** Revisar a cada sprint novo

---

## 🔍 PROCURAR

Use Ctrl+Shift+F (busca global Obsidian):

```
Status Code       → find: D09F00
Hanging           → find: is_session_hanging
Error Source      → find: failed_data_sources
Storage           → find: selected_size_bytes
Partner           → find: PN, ParentId
```

---

**Última consultoria:** 2026-05-05 pelo Claude  
**Próxima revisão:** Sprint 1 completo
