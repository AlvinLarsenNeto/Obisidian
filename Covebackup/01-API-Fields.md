# Covebackup API - Mapeamento Completo de Dados

## Resumo Executivo

A API Covebackup expõe **112+ campos de informação** por workload (dispositivo/servidor). Atualmente, o Portal ASV utiliza apenas **~20% desses dados**. Este documento mapeia tudo que está disponível e sugere oportunidades de melhoria.

---

## 1. Arquitetura da Integração

### Endpoints Utilizados

| Endpoint | Propósito | Frequência |
|---|---|---|
| `EnumerateAccountStatistics` | Fetch de ALL workloads com status | 5min (auto-refresh) + cron |
| `GetPartnerInfoById` | Resolução de IDs de cliente/partner | 1x por sync |
| `GetAccountInfoById` | Detalhes de erro de workload específico | On-demand |
| `EnumeratePartners` | Listagem de clientes (não usado) | —|

### Portais Dual

- **Device Portal**: Cove API padrão (servidores, workstations, NAS)
- **GB Portal**: Gestão de backup de longa retenção (redundância geográfica)
- Ambas sincronizam via `backup_status_cache` (tabela única)

---

## 2. Dados ATUALMENTE CAPTURADOS

### Tabela: `backup_status_cache`

Campos persistidos no banco após sync:

```sql
-- Identificação
company_id              -- Empresa (cliente)
workload_name           -- Nome do dispositivo/servidor
cove_workload_id        -- AccountId no Cove (para lookups futuros)
source                  -- "Cove Device" ou "Cove GB"

-- Status de Backup
status                  -- "ok", "warning", "critical", "unknown"
last_success            -- Último backup bem-sucedido (ISO)
last_failure            -- Último backup que falhou (ISO)
last_sync               -- Última sincronização com Cove

-- Características do Workload
workload_type           -- "Servidor", "Workstation", "M365", null
physicality             -- "Physical" ou "Virtual"
is_critical             -- boolean (true se Servidor)
profile                 -- Nome do perfil de backup (da config Cove)

-- Tamanho & Armazenamento
selected_size_bytes     -- Bytes selecionados para backup
used_storage_bytes      -- Bytes efetivamente utilizados no armazenamento
color_bar_14d           -- String 14-char: histórico visual (5=ok, 2=fail, etc)

-- M365 Específico
m365_mailbox_count      -- Contagem de mailboxes (USER ONLY, excluso shared)
m365_user_mailboxes     -- Mailboxes de usuário
m365_shared_mailboxes   -- Mailboxes compartilhadas

-- Erros & Alertas
last_error_message      -- Mensagem de erro do último backup (até 2000 chars)
error_data_source       -- Data source onde o erro ocorreu
last_session_error_count-- Contagem de erros na última sessão

-- Reconciliação
recurrence_30d          -- Contagem de incidentes nos últimos 30d (calculado)
cove_customer_ref       -- Customer name (após resolução de hierarchy)
```

---

## 3. Dados DISPONÍVEIS NA API MAS NÃO CAPTURADOS

### 3.1 Session Status Detalhado (D09F00 / SessionStatus)

**O que é?** Código numérico do status da última sessão.

**Valores:**
- `1` = "Em execução"
- `2` = "Falha"
- `3` = "Abortado"
- `5` = "Concluído"
- `6` = "Interrompido"
- `7` = "Não iniciado"
- `8` = "Concluído com erros"
- `9` = "Em execução (falhas)"
- `10` = "Over quota"
- `11` = "Sem seleção"
- `12` = "Reiniciado"

**Uso potencial:**
- Dashboard diferencia "warning" vs "critical" por tempo, não por status real
- Podia alertar sobre "Over quota" (10) ou "Sem seleção" (11) antes de virar failure
- Histórico de status por workload (rastrear padrões)

### 3.2 Timestamps Múltiplos

| Campo | Significado |
|---|---|
| `D09F09` | **Last Successful Session** (já capturado) |
| `D09F07` | **Last Failure Time** (já capturado) |
| `D09F15` | **Last Session Time** (ANY status, sucesso ou falha) |

**Uso potencial:**
- `D09F15` diz exatamente quando a última tentativa foi (independente de sucesso)
- Alerta: "Sem tentativa há 48h" é diferente de "Última tentativa falhou"
- Prognóstico: se `D09F15` < 12h mas `D09F09` > 7d, algo está quebrando

### 3.3 Dados por Data Source (granular)

Cada workload pode ter **múltiplas data sources**. Cove retorna erro por source:

```
D09F19  -- Total session verification data (general error)
TK      -- Total - session verification details
SK      -- System State - session verification details
FK      -- Files and Folders - session verification details
HK      -- Hyper-V - session verification details
WK      -- VMware - session verification details
GK      -- M365 Exchange - session verification details
JK      -- M365 OneDrive - session verification details
ZK      -- VSS MS SQL - session verification details
```

**Exemplo:** Servidor com backup de "Files + SQL" pode ter Files ok mas SQL falhando.

**Uso potencial:**
- Alertas granulares: "M365 Exchange falhou, mas Files ok"
- Auditoria: quais data sources estão habilitadas por workload
- Otimização: encontrar data sources não usados

### 3.4 Tamanho por Data Source (D01F03, D02F03, D08F03, D14F03, D19F03, D20F03)

```
D01F03 -- Files and Folders - Selected Size
D02F03 -- System State - Selected Size
D08F03 -- VMware - Selected Size
D14F03 -- Hyper-V - Selected Size
D19F03 -- M365 Exchange - Selected Size
D20F03 -- M365 OneDrive - Selected Size
```

**Atualmente:** Usa primeiro valor não-nulo (simplificado).

**Uso potencial:**
- Breakdown: "Exchange usa 500GB, OneDrive 200GB, Files 300GB"
- Otimização: qual data source consome mais?
- Forecast: crescimento por source

### 3.5 Infos de Partner/Customer

```
PN  -- Partner Name (intermediate reseller or direct)
AR  -- Account Reference (pode ser customer name, departamento, etc)
```

**Normalmente usado para:** reconciliar workloads a empresas no portal.

**Captura atual:** Sim, mas apenas para matching (não armazenado).

**Uso potencial:**
- Armazenar `PN` original antes de normalização → auditoria
- Detectar mismatches: workload aponta pro Partner A mas Company B configurada

### 3.6 OS Type & Physicality Inferido

```
OT    -- OS Type (1=Workstation, 2=Servidor, null=M365)
I81   -- Physicality (1=Physical, 2=Virtual)
AP    -- Active Profile/Product codes (bitfield)
```

**Atualmente:**
- OT é capturado mas `I81` e `AP` apenas usados para inference

**Uso potencial:**
- Trend: % de virtual vs physical ao longo do tempo
- Compliance: garantir máquinas virtuais estão em hypervisors rastreados
- Cost: VMs podem ter SLA diferente de physical

### 3.7 Active Product Code (AP / Bitfield)

```
"D8"  -- VMware (indica máquina virtual)
"D14" -- Hyper-V (indica máquina virtual)
```

**Uso potencial:**
- Detecção automática: se AP contém "D8" ou "D14", é VM 100%
- Inventory: quais hypervisores estão sendo usados

---

## 4. Dados em Estruturas Aninhadas NÃO EXPLORADOS

### 4.1 Company Info (GetPartnerInfoById)

```json
{
  "Id": 123,
  "ParentId": 456,
  "Company": {
    "LegalCompanyName": "ASV Tecnologia LTDA",
    "Name": "ASV Tecnologia"
  },
  "Name": "ACME Corp"
}
```

**Captura atual:** Apenas para resolver customer name (Partner hierarchy).

**Uso potencial:**
- Armazenar metadados do cliente: razão social, CNPJ (via Company)
- Detectar resellers vs clientes diretos (via ParentId)
- Multi-tier reconciliation: Grupo > Reseller > Cliente

### 4.2 Session History (EnumerateAccountHistoryStatistics)

**Status:** Não implementado (requer elevated API permissions).

**O que seria:**
- Histórico de N últimas sessões por workload
- Backup duration, bytes transferred, etc.

**Bloqueador:** Credenciais Cove atuais não têm permissão.

---

## 5. Tabela de Campos COMPLETA da API

### Request Columns (enumerateAccountStatistics)

```
Identificação:
  AN    -- Account Name (device name)
  MN    -- Machine Name (altnative)
  AR    -- Account Reference (customer ID)
  OT    -- OS Type
  I14   -- Used Storage (bytes)
  I81   -- Physicality
  AP    -- Active Products (bitfield)
  OP    -- Operating Profile
  PN    -- Partner Name

Última Sessão (Main):
  D09F00 -- Session Status (1=running, 2=failed, 5=ok, 8=errors, etc)
  D09F03 -- Selected Size (primary)
  D09F06 -- Errors Count
  D09F07 -- Last Failure Time
  D09F08 -- Color Bar (14-day visual)
  D09F09 -- Last Success Time
  D09F15 -- Last Session Time (ANY)
  D09F19 -- General Error Message

Última Sessão (By Data Source):
  SK, FK, HK, WK, GK, JK, ZK, TK -- Error per data source
  D01F03, D02F03, D08F03, D14F03, D19F03, D20F03 -- Size per source

M365 Specific:
  D19F20 -- M365 Exchange User Mailboxes
  D19F21 -- M365 Exchange Shared Mailboxes

Portal Link:
  I0 -- AccountId (for direct API link)
```

---

## 6. Oportunidades de Melhoria para o Módulo Backup

### 6.1 SHORT-TERM (1-2 semanas)

#### A. Alertas Mais Específicos

**Problema:** "Critical" = "não fez backup em >48h", mas não diferencia:
- Servidor parou de reportar (acesso perdido)
- Backup rodando há dias sem terminar (hanging session)
- Data source específico falha mas outros ok

**Solução:**
```sql
-- Adicionar à backup_status_cache:
pending_session         -- boolean: if D09F00=1 (running) AND D09F15 recent
selected_vs_used_ratio  -- pct: selected_size/used_storage
data_sources_enabled    -- json: {"files": true, "exchange": false, ...}
last_error_source       -- "Files" vs "Exchange" vs "System State"
```

**UI:** Dashboard mostra:
- 🔵 **Running 3d** (hanging) vs 🟡 **Last ok 2d** (normal delay)
- 🔴 **Exchange failed** (but Files ok) vs 🔴 **All failed**

#### B. Granular Storage Breakdown

**Problema:** Só vê "100 GB usado", não sabe se é Files/Exchange/OneDrive.

**Solução:** Armazenar e exibir:

```sql
-- Adicionar à backup_status_cache:
size_files_bytes
size_system_state_bytes
size_vmware_bytes
size_hyperv_bytes
size_m365_exchange_bytes
size_m365_onedrive_bytes
size_sql_bytes
```

**UI:** Gráfico de pizza por workload:
```
Exchange: 500 GB (50%)
OneDrive: 300 GB (30%)
Files: 200 GB (20%)
```

#### C. Detectar Over-Quota & Sem-Seleção

**Status 10 & 11 raramente são monitados.**

**Solução:**
```sql
-- Alert rules
WHERE status_code = 10 THEN alert "Over quota - expand storage"
WHERE status_code = 11 THEN alert "Nothing selected - check backup settings"
WHERE status_code = 7 THEN alert "Never started - check network"
```

#### D. Trend: Backup Duração & Velocidade

**Problema:** Cove não expõe `session_duration` ou `bytes_per_second`.

**Workaround:** Comparar `D09F15 - D09F09` para detectar backups lentos.

**Solução:**
```js
// Se última sessão bem-sucedida > 24h atrás:
const lastSessionTime = parseTimestamp(D09F15);
const lastSuccessTime = parseTimestamp(D09F09);
const estimatedDuration = lastSessionTime - lastSuccessTime; // pode estar em sessão atual

if (estimatedDuration > 48 * 3600) {
  alert("Backup está levando anormalmente longo");
}
```

### 6.2 MID-TERM (2-4 semanas)

#### A. Histórico de Incidentes por Data Source

**Problema:** `backup_workload_incidents` não rastreia qual data source falhou.

**Solução:**
```sql
-- Expandir backup_workload_incidents:
ALTER TABLE backup_workload_incidents ADD COLUMN (
  failed_data_sources  TEXT,  -- "Files,Exchange" or "System State"
  status_code          INT,   -- 1-12 da Cove
  session_attempt_time TIMESTAMP
);
```

**Relatório:** "M365 Exchange falhou 5x em 30d, Files nunca falhou"

#### B. Partner Hierarchy Sync

**Problema:** Cove tem relação cliente > reseller > grupo, portal não explora.

**Solução:**
```sql
-- Novo table: cove_partner_hierarchy
CREATE TABLE cove_partner_hierarchy (
  cove_partner_id      INT PRIMARY KEY,
  parent_partner_id    INT,
  legal_name           TEXT,
  display_name         TEXT,
  cove_portal          TEXT, -- "device" ou "gb"
  synced_at            TIMESTAMP
);
```

**Uso:** Auto-map workloads mesmo se customer name muda (usa PartnerId).

#### C. Activity Timeline

**Problema:** Não há visibilidade de "quando falhou pela 1ª vez" vs "recorrente".

**Solução:** Expandir Dashboard com:
```
Backup Timeline:
[OK]     [OK]     [FAIL]   [OK]     [FAIL]   [FAIL]
Jan 1    Jan 8    Jan 15   Jan 22   Jan 29   Feb 5
```

Automático via `recurrence_30d`.

### 6.3 LONG-TERM (1-2 meses)

#### A. Forecasting & Capacity Planning

**Dados necessários:**
- `size_growth_per_month`: track tamanho ao longo do tempo
- `last_full_backup_time`: quando foi o último full (vs incremental)
- `retention_days`: quanto tempo de retenção está configurado

**Relatório:** "Exchange 10GB/mês, vai exceder quota em 3 meses"

#### B. SLA Dashboard para Backup

```
Workload SLA:
- Max time without successful backup: 24h
- Max time without ANY attempt: 48h
- Max consecutive failures: 3

Métricas:
- Uptime: % tempo com backup ok (últimos 30d)
- MTTR: tempo médio pra resolver falha
- Cost per GB: usado para chargeback
```

#### C. Integração com Tickets

**Problema:** Backup crítico falha, ninguém sabe = RTO/RPO violation.

**Solução:**
```
backup-auto-tickets/  (já existe!)
├── trigger: se "critical" + status=failed + duration>24h
├── create ticket
├── escalate se >48h sem resolver
└── auto-resolve quando backup fica ok
```

**Melhoria:** Integrar dados granulares de data source:
```
Ticket description:
"Servidor PROD-SQL-01: M365 Exchange backup falha há 48h.
 Erro: OneDrive não foi selecionado.
 Files+System State: OK"
```

#### D. Webhook para Status Real-Time

**Status quo:** Poll a cada 5 min (overhead).

**Idealmente:** Cove enviaria webhook quando workload muda de status.

**Workaround:** Usar cache agressivo + change detection.

---

## 7. Estrutura de Tabelas Proposta

### Nova Tabela: `backup_workload_details`

Detalhe granular por data source:

```sql
CREATE TABLE backup_workload_details (
  id UUID PRIMARY KEY,
  company_id UUID,
  workload_name TEXT,
  cove_workload_id TEXT,
  source TEXT, -- "Cove Device" or "Cove GB"
  
  -- Data Source Breakdown
  data_source TEXT, -- "Files", "Exchange", "OneDrive", "System State", etc
  size_bytes BIGINT,
  last_session_error TEXT,
  last_session_status INT,
  last_session_time TIMESTAMP,
  
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  
  UNIQUE(company_id, workload_name, source, data_source)
);
```

### Expandir: `backup_status_cache`

Campos novos:

```sql
-- Status granular
last_session_status_code  INT,    -- 1-12
has_pending_session       BOOLEAN,
session_in_progress_since TIMESTAMP,

-- Data source audit
enabled_data_sources      TEXT[], -- {"Files", "Exchange"}
failed_data_sources       TEXT[], -- {"SQL"}

-- Storage breakdown (total)
size_breakdown            JSONB,  -- {files: 200GB, exchange: 500GB}

-- Comparativos
selected_vs_used_pct      DECIMAL,
growth_pct_7d             DECIMAL,
growth_pct_30d            DECIMAL,
```

---

## 8. Exemplo: Transformar Alert "Critical" em Decisão

### Cenário Atual

Dashboard mostra:
```
PROD-MAIL-01: 🔴 CRITICAL
Last Success: 7 days ago
```

Gestor: "Que diabos, vou abrir ticket de urgência!"

Técnico recebe ticket genérico: "Backup falhou".

### Cenário Melhorado (com dados granulares)

Dashboard mostra:
```
PROD-MAIL-01: 🔴 CRITICAL
Status Code: 8 (Completed with Errors)
Last Success: 7 days ago
Last Attempt: 2 hours ago

Data Sources:
  Files & Folders: ✅ OK (200 GB)
  System State:    ✅ OK (50 GB)
  M365 Exchange:   ❌ FAIL (quota exceeded - 800/750 GB)
  M365 OneDrive:   ⏳ RUNNING (started 3h ago)

Error Message: "OneDrive quota exceeded. Contact admin."
```

Técnico recebe ticket específico:
```
PROD-MAIL-01: M365 Exchange quota exceeded
- Size: 800 GB, Limit: 750 GB, Over: 50 GB
- Action: Increase quota or clean retention
- Impact: Exchange data NOT backed up, Files/System OK
- ETA to resolve: 1h if quota increased
```

---

## 9. Priorização para Dev

| Item | Complexidade | Impacto | Timeline |
|---|---|---|---|
| **Capturar status_code + data_source errors** | Baixa | Alto | Semana 1 |
| **Novo schema backup_workload_details** | Média | Médio | Semana 2-3 |
| **UI: Storage breakdown + error source** | Média | Alto | Semana 2-3 |
| **Partner hierarchy sync** | Média | Médio | Semana 3-4 |
| **Activity timeline visual** | Baixa | Médio | Semana 2 |
| **Alertas específicos (over-quota, etc)** | Baixa | Alto | Semana 1-2 |
| **SLA Dashboard** | Alta | Médio | Semana 5-6 |
| **Session history API unlock** | Média | Médio | TBD (requer permissão Cove) |

---

## 10. Campos por Prioridade de Captura

### 🔴 Crítico (implementar já)

```
D09F00 -- Status code
D09F19, TK, SK, FK, HK, WK, GK, JK, ZK -- Errors por data source
D09F15 -- Last session time (ANY)
AP -- Active products (para detectar VM)
```

### 🟡 Importante (próximas 2 sprints)

```
D01F03, D02F03, D08F03, D14F03, D19F03, D20F03 -- Size per data source
PN -- Partner name (antes de normalize)
OT -- OS Type (já temos, melhorar storage)
I81 -- Physicality (já temos, melhorar storage)
```

### 🟢 Nice-to-have (backlog)

```
Session history (requer permissão)
Company hierarchy details (ParentId, LegalName)
Session duration inference
Bytes-per-second trends
```

---

## 11. Queries SQL Úteis (após implementação)

### Alert: Over Quota

```sql
SELECT workload_name, last_session_status_code, size_breakdown->>'exchange_bytes'
FROM backup_status_cache
WHERE company_id = ?
  AND last_session_status_code = 10
  AND updated_at > now() - interval '24h';
```

### Alert: Hanging Session

```sql
SELECT workload_name, session_in_progress_since,
       EXTRACT(HOUR FROM now() - session_in_progress_since) as hours_running
FROM backup_status_cache
WHERE company_id = ?
  AND has_pending_session = true
  AND session_in_progress_since < now() - interval '12h';
```

### Health: By Data Source

```sql
SELECT 
  data_source,
  COUNT(*) as total_workloads,
  COUNT(*) FILTER (WHERE last_session_error IS NULL) as healthy,
  ROUND(100.0 * COUNT(*) FILTER (WHERE last_session_error IS NULL) / COUNT(*), 2) as health_pct
FROM backup_workload_details
WHERE company_id = ? AND updated_at > now() - interval '30d'
GROUP BY data_source
ORDER BY health_pct;
```

### Storage Trend

```sql
SELECT 
  DATE(updated_at) as date,
  SUM(size_bytes) as total_backed_up
FROM backup_workload_details
WHERE company_id = ?
GROUP BY DATE(updated_at)
ORDER BY date DESC
LIMIT 90;
```

---

## 12. Documentação Cove API (Referência)

**Link:** Não público, mas credenciais no `.env`:
- `COVE_API_URL` = `https://api.backup.management/jsonrpcv1`
- Métodos: `Login`, `EnumerateAccountStatistics`, `GetPartnerInfoById`, `GetAccountInfoById`

**Nota:** Swagger/OpenAPI não disponível; API documentada via exemplos em `supabase/functions/cove/index.ts`.

---

## 13. Conclusão

O módulo Backup tem potencial pra evolução significativa. A API Cove expõe riqueza de dados que hoje não é explorada:

✅ **Hoje:** Status simples (ok/warning/critical) + lista de workloads  
❌ **Perdendo:** Granularidade de erro, breakdown de storage, trends, forecasting  

**Próximos passos:** Começar capturando `status_code` e `failed_data_sources` (semana 1), depois expandir schema e UI.
