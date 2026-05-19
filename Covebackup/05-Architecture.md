# Covebackup - Arquitetura de Dados

## 1. Fluxo de Dados (Current State)

```
┌─────────────────────────────────────────────────────────────┐
│                    COVE BACKUP PORTALS                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Device Portal                          GB Portal            │
│  (api.backup.management)                (api.backup.management)
│     ├─ Servidores                         ├─ Long retention  │
│     ├─ Workstations                       └─ Geo-redundancy  │
│     └─ NAS                                                    │
│                                                               │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           │ EnumerateAccountStatistics
                           │ (112+ campos, >5000 workloads)
                           │
                    ┌──────▼──────┐
                    │ Edge Function│
                    │   /cove      │
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    ┌─────────────┐  ┌──────────────┐  ┌──────────┐
    │ Parse + Map │  │ Resolve      │  │ Extract  │
    │ Data        │  │ Partner IDs  │  │ Errors   │
    │             │  │              │  │          │
    │ Fields: 20  │  │ PartnerId->  │  │ TK,SK,FK │
    │ captured    │  │ Customer     │  │ GK,JK,etc│
    │             │  │ Name         │  │          │
    └─────────────┘  └──────────────┘  └──────────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
                    ┌──────▼──────────────┐
                    │ INSERT/UPDATE       │
                    │ backup_status_cache │
                    │ (1 record = 1 WL)   │
                    └────────────┬────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Supabase PostgreSQL     │
                    │  ┌────────────────────┐  │
                    │  │ 1x row per         │  │
                    │  │ workload + portal  │  │
                    │  │                    │  │
                    │  │ ~40 columns stored │  │
                    │  │ 5000+ rows total   │  │
                    │  └────────────────────┘  │
                    └────────────────────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │ Dashboard + Pages        │
                    │                          │
                    │ • CoveBackupSection      │
                    │ • BackupsPage            │
                    │ • Status badges          │
                    │ • Filters (type/status)  │
                    │ • Color bars (14d)       │
                    └──────────────────────────┘
```

---

## 2. Dados da API vs Banco vs UI (Comparação)

### Estrutura de Dados

```
COVE API                          BACKUP_STATUS_CACHE         UI Component
─────────────────────             ───────────────────         ────────────

Workload ID (I0) ─────────────→ cove_workload_id ─────────→ [Rarely used]

Name (AN/MN) ─────────────────→ workload_name ───────────→ Display name

Customer (AR/PN) ────────────→ cove_customer_ref ─────────→ [Not shown]

OS Type (OT) ──────────────→ workload_type ────────────→ Badge: "Servidor"

Physicality (I81) ────────────→ physicality ──────────────→ Badge: "Virtual"

Status Code (D09F00) ──────→ [NOT captured yet] ────→ [Would show: "Running"]
                                                       [Or: "Over quota"]

Last Success (D09F09) ─────────→ last_success ────────────→ "dd/MM HH:mm"

Last Session (D09F15) ──────→ [NOT captured yet] ────→ [Would help detect]
                                                       [hanging sessions]

Error Messages (TK/SK/FK/GK/JK) → last_error_message ──→ Tooltip

Error by Source (SK,FK,etc) ──→ [NOT captured yet] ────→ [Would show badge]:
                                                       ["Files", "Exchange"]

Color Bar (D09F08) ────────────→ color_bar_14d ────────────→ Visual: ▌▌▌ etc

Profile (OP) ──────────────────→ profile ──────────────→ Gray text

Mail count (D19F20/21) ────────→ m365_mailbox_count ────────→ [Not shown]

Selected Size (D09F03) ────────→ selected_size_bytes ───────→ [Not shown]

Used Storage (I14) ────────────→ used_storage_bytes ───────→ [Not shown]

Error Count (D09F06) ──────────→ last_session_error_count → [Not shown]

Size per Source (D01F03...) ──→ [NOT captured] ─────────→ [Would show pie]
                                                       [chart breakdown]

Partner IDs + Hierarchy ───────→ [NOT stored] ──────────→ [Auto-map future]
```

---

## 3. EnumerateAccountStatistics - Campo por Campo

```
Metadata do Workload:
┌─────────────────────────────────────────────────────────┐
│ AN  = Account Name (device name, e.g., "PROD-MAIL-01")  │
│ MN  = Machine Name (alternative, e.g., hostname)        │
│ OT  = OS Type (1=WS, 2=Srv, null=M365)                  │
│ I81 = Physicality (1=Phys, 2=Virt, null=unknown)        │
│ I14 = Used Storage (bytes)                              │
│ AP  = Active Products (bitfield: "D8"=VMware, etc)      │
│ OP  = Operating Profile (backup schedule name)          │
│ I0  = AccountId (for direct Cove link)                  │
└─────────────────────────────────────────────────────────┘

Customer/Partner Reference:
┌─────────────────────────────────────────────────────────┐
│ AR = Account Reference (often customer name/ID)         │
│ PN = Partner Name (intermediate reseller, if any)       │
└─────────────────────────────────────────────────────────┘

Session Status (Last Attempt):
┌─────────────────────────────────────────────────────────┐
│ D09F00 = Session Status Code (1-12)                     │
│   1=Running, 2=Failed, 5=OK, 8=WithErrors, 10=Quota... │
│ D09F15 = Last Session Timestamp (ANY outcome)          │
│ D09F08 = Color Bar (14-day history: "555542345555555") │
│ D09F06 = Error Count (in last session)                  │
└─────────────────────────────────────────────────────────┘

Session Success vs Failure:
┌─────────────────────────────────────────────────────────┐
│ D09F09 = Last Successful Session Timestamp             │
│ D09F07 = Last Failure Timestamp                        │
│ D09F19 = General Error Message                         │
└─────────────────────────────────────────────────────────┘

Data Source Breakdown - Errors:
┌─────────────────────────────────────────────────────────┐
│ SK = System State error                                │
│ FK = Files & Folders error                             │
│ HK = Hyper-V error                                     │
│ WK = VMware error                                      │
│ GK = M365 Exchange error                               │
│ JK = M365 OneDrive error                               │
│ ZK = MS SQL error                                      │
│ TK = Total/General error                               │
│ (Each contains error message or null)                  │
└─────────────────────────────────────────────────────────┘

Data Source Breakdown - Size:
┌─────────────────────────────────────────────────────────┐
│ D01F03 = Files & Folders - Selected Size (bytes)       │
│ D02F03 = System State - Selected Size (bytes)          │
│ D08F03 = VMware - Selected Size (bytes)                │
│ D14F03 = Hyper-V - Selected Size (bytes)               │
│ D19F03 = M365 Exchange - Selected Size (bytes)         │
│ D20F03 = M365 OneDrive - Selected Size (bytes)         │
│ (D09F03 = Total selected size - primary field)         │
└─────────────────────────────────────────────────────────┘

M365 Specifics:
┌─────────────────────────────────────────────────────────┐
│ D19F20 = M365 Exchange - User Mailboxes Count           │
│ D19F21 = M365 Exchange - Shared Mailboxes Count         │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Database Schema - Estado Atual vs Proposto

### Tabela: `backup_status_cache` (CURRENT)

```sql
CREATE TABLE backup_status_cache (
  id UUID PRIMARY KEY,
  company_id UUID,
  workload_name TEXT,
  workload_type TEXT,            -- "Servidor", "Workstation", "M365"
  source TEXT,                   -- "Cove Device" / "Cove GB"
  status TEXT,                   -- "ok", "warning", "critical"
  last_success TIMESTAMP,        -- D09F09
  last_failure TIMESTAMP,        -- D09F07
  last_sync TIMESTAMP,
  is_critical BOOLEAN,
  color_bar_14d TEXT,            -- D09F08 (14 chars)
  profile TEXT,                  -- OP
  physicality TEXT,              -- "Physical" / "Virtual"
  last_error_message TEXT,       -- TK/SK/etc truncado
  error_data_source TEXT,        -- qual source falhou (extraído)
  cove_workload_id TEXT,         -- I0
  cove_customer_ref TEXT,        -- AR (cleaned)
  last_session_error_count INT,  -- D09F06
  selected_size_bytes BIGINT,    -- D09F03
  used_storage_bytes BIGINT,     -- I14
  m365_mailbox_count INT,        -- D19F20 (user only)
  m365_user_mailboxes INT,       -- D19F20
  m365_shared_mailboxes INT,     -- D19F21
  recurrence_30d INT,            -- Calculado (incidentes últimos 30d)
  
  UNIQUE(company_id, workload_name, source),
  FOREIGN KEY(company_id) REFERENCES companies(id)
);
```

### Tabela: `backup_status_cache` (SPRINT 1 - PROPOSED)

```sql
ALTER TABLE backup_status_cache ADD COLUMN (
  last_session_status_code INT,          -- D09F00 (status code 1-12)
  failed_data_sources TEXT[],            -- ["Files", "Exchange"]
  is_session_hanging BOOLEAN,            -- true se status=1 AND D09F15 > 12h ago
  
  -- Índices para queries de alerta
  INDEX idx_hanging ON (company_id) WHERE is_session_hanging = true,
  INDEX idx_failed_sources ON (company_id) WHERE failed_data_sources IS NOT NULL
);
```

### Tabela: `backup_workload_details` (SPRINT 2 - PROPOSED)

```sql
CREATE TABLE backup_workload_details (
  id UUID PRIMARY KEY,
  company_id UUID,
  workload_name TEXT,
  cove_workload_id TEXT,
  source TEXT,                   -- "Cove Device" / "Cove GB"
  
  -- Data Source Specificity
  data_source TEXT,              -- "Files", "Exchange", "OneDrive", etc
  size_bytes BIGINT,             -- D01F03, D02F03, etc
  last_session_error TEXT,       -- FK, GK, JK, etc
  last_session_status INT,       -- Status code desse source
  last_session_time TIMESTAMP,   -- D09F15
  
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now(),
  
  PRIMARY KEY (company_id, workload_name, source, data_source),
  FOREIGN KEY(company_id) REFERENCES companies(id)
);
```

---

## 5. Fluxo de Sync Detalhado

```
┌─────────────────────────────────────────────────────────────────┐
│                    CRON JOB (diário/horário)                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
         ┌──────────────────────────────┐
         │ POST /cove/sync-all          │
         │ (headers: X-Cron-Secret)     │
         └──────────────┬───────────────┘
                        │
         ┌──────────────▼───────────────┐
         │ 1. Fetch Visa (auth token)   │
         │    - Device Portal           │
         │    - GB Portal               │
         │    - Cache 30 min            │
         └──────────────┬───────────────┘
                        │
         ┌──────────────▼───────────────┐
         │ 2. EnumerateAccountStatistics│
         │    (Device Portal)           │
         │    → 5000+ workloads         │
         │    → 112+ campos             │
         └──────────────┬───────────────┘
                        │
         ┌──────────────▼───────────────┐
         │ 3. EnumerateAccountStatistics│
         │    (GB Portal)               │
         │    → 5000+ workloads         │
         │    → 112+ campos             │
         └──────────────┬───────────────┘
                        │
         ┌──────────────▼───────────────────────┐
         │ 4. GetPartnerInfoById (para cada ID) │
         │    → Resolve PartnerId → Customer    │
         │    → Walk hierarchy                  │
         │    → Caching local                   │
         └──────────────┬───────────────────────┘
                        │
         ┌──────────────▼────────────────────────┐
         │ 5. Parse + Transform                 │
         │    - Extract D09F00, D09F09, TK...   │
         │    - Parse timestamps                │
         │    - Detect hanging sessions (NEW)   │
         │    - Identify failed sources (NEW)   │
         │    - Normalize customer names        │
         │    - Match to company_id             │
         └──────────────┬────────────────────────┘
                        │
         ┌──────────────▼────────────────────────┐
         │ 6. Reconcile Unmapped                │
         │    - Workloads sem company match     │
         │    - Insert com UNMAPPED_COMPANY_ID  │
         │    - Log pra auditoria               │
         └──────────────┬────────────────────────┘
                        │
         ┌──────────────▼────────────────────────┐
         │ 7. DELETE old cache                  │
         │    - DELETE * WHERE company_id IN (...) │
         │    - Substitui completamente         │
         └──────────────┬────────────────────────┘
                        │
         ┌──────────────▼────────────────────────┐
         │ 8. INSERT new backup_status_cache    │
         │    - ~40 cols (agora com 3 novos)    │
         │    - Bulk insert                     │
         └──────────────┬────────────────────────┘
                        │
         ┌──────────────▼────────────────────────┐
         │ 9. Calculate recurrence_30d          │
         │    - Query backup_workload_incidents │
         │    - Count per workload (últimos 30d)│
         │    - UPDATE backup_status_cache      │
         └──────────────┬────────────────────────┘
                        │
         ┌──────────────▼────────────────────────┐
         │ 10. Auto-close orphaned incidents    │
         │     - Workloads que sumiram da API   │
         │     - Mark as "resolved"             │
         │     - Auto-resolve date = now()      │
         └──────────────┬────────────────────────┘
                        │
         ┌──────────────▼────────────────────────┐
         │ 11. Log sync result                  │
         │     - INSERT backup_sync_logs        │
         │     - Duration, count, errors        │
         └──────────────┬────────────────────────┘
                        │
                        ▼
            ┌─────────────────────────┐
            │ Frontend auto-refresh    │
            │ (5 min interval)         │
            │ GET /cove/summary        │
            │ (re-render dashboard)    │
            └─────────────────────────┘
```

---

## 6. Fluxo de Alertas (Proposto Sprint 1)

```
backup_status_cache (updated)
│
├─ last_session_status_code = 2 (FAILED)
│  └─→ [CRIT] Backup failed
│
├─ last_session_status_code = 10 (OVER QUOTA)
│  └─→ [CRIT] Over quota - expand storage
│
├─ last_session_status_code = 11 (NOT SELECTED)
│  └─→ [WARN] No data sources selected - check config
│
├─ is_session_hanging = true (status=1 + elapsed > 12h)
│  └─→ [WARN] Session running >12h (check for hang/deadlock)
│
├─ failed_data_sources = ["Exchange"]
│  └─→ [WARN] M365 Exchange backup failed, others OK
│       Action: Check Exchange connector
│
├─ failed_data_sources = ["Files", "System State"]
│  └─→ [CRIT] Multiple sources failed - escalate
│
└─ last_success IS NULL + is_critical = true
   └─→ [CRIT] Critical server never backed up - urgent
```

---

## 7. UI Component Hierarchy

```
DashboardPage
│
└─ CoveBackupSection (memo)
   ├─ Header (title, refresh, "Ver detalhes")
   │
   ├─ KPI Cards (NEW Sprint 2)
   │  ├─ Total Workloads
   │  ├─ Critical Failures
   │  ├─ Hanging Sessions (NEW)
   │  └─ Over Quota (NEW)
   │
   ├─ Workload List
   │  ├─ Header Row
   │  │  ├─ Status Icon
   │  │  ├─ Workload Name
   │  │  ├─ Type + Status Code Badge (NEW)
   │  │  ├─ Color Bar (14d)
   │  │  ├─ Last Success
   │  │  └─ Failed Sources (NEW)
   │  │
   │  └─ Workload Rows x N
   │     ├─ Status Icon (with tooltip)
   │     ├─ Name + Profile
   │     ├─ Type Badge
   │     ├─ Status Code Badge (NEW)
   │     │  └─ Color by status:
   │     │     🔴 Failed / Over Quota
   │     │     🟠 WithErrors / Running
   │     │     🟢 OK
   │     ├─ Physicality Badge
   │     ├─ Portal Badge
   │     ├─ Failed Sources Badge (NEW)
   │     │  └─ Tooltip: list sources
   │     ├─ Color Bar
   │     └─ Last Backup Date
   │
   └─ Pagination

BackupsPage (full page)
│
├─ Filters
│  ├─ Search (name)
│  ├─ Type (Server/Workstation/M365)
│  ├─ Status (All/OK/Alert/Failed/Hanging/OverQuota) (NEW)
│  └─ Portal (All/Device/GB)
│
├─ Sorting (Name, Status, Last Success)
│
├─ Results Table
│  ├─ Workload Detail Row x N
│  │  └─ Expandable row (NEW Sprint 2)
│  │     ├─ Storage Breakdown (pie chart)
│  │     ├─ Data Sources Health
│  │     └─ Error History (timeline)
│  │
│  └─ Pagination
│
└─ Detail Modal (click on row)
   ├─ Overview
   │  ├─ Name, Type, Physicality
   │  ├─ Status Code + Label
   │  └─ Last Success/Failure
   │
   ├─ Storage (NEW)
   │  ├─ Selected: XXX GB
   │  ├─ Used: YYY GB
   │  └─ Breakdown by source
   │
   ├─ Error Details (NEW)
   │  ├─ Failed sources list
   │  ├─ Error messages per source
   │  └─ Recommendations
   │
   └─ History (NEW Sprint 2)
      ├─ Timeline of status changes
      └─ Incidents with dates
```

---

## 8. Data Availability Timeline

```
TODAY (Sprint 0):
✅ workload_name, status, last_success, color_bar_14d
✅ profile, physicality, osType, portalSource
✅ selected_size_bytes, used_storage_bytes
✅ m365_mailbox_count
⚠️  error_message (genérico, não por source)

SPRINT 1 (1-2 weeks):
✅ last_session_status_code (1-12)
✅ failed_data_sources (["Files", "Exchange"])
✅ is_session_hanging (boolean)
✅ Alerts: hanging, over-quota, not-selected
✅ UI: status code badge, failed sources badge

SPRINT 2 (2-4 weeks):
✅ backup_workload_details (size per source)
✅ Storage breakdown (pie chart)
✅ Data source health matrix
✅ Activity timeline visual
⏳ Partner hierarchy auto-sync

SPRINT 3+ (backlog):
⏳ Session history (requer permissão Cove)
⏳ Forecasting & capacity planning
⏳ SLA dashboard
⏳ Webhook integration (real-time)
```

---

## 9. Recommendations by Data

```
Data Available → Recommendation

Session Status = "Over Quota" (10)
  └─→ Immediate: "Expand quota for Exchange (800/750 GB)"
  └─→ Timeline: resolve in <24h

Session Status = "Not Selected" (11)
  └─→ Verify: backup config changed?
  └─→ Check: data source disabled accidentally

Failed Data Sources = ["Exchange", "OneDrive"]
  └─→ Pattern: all M365 sources down
  └─→ Root cause: Exchange connector issue

Failed Data Sources = ["Files"]
  └─→ Isolated: only file backup failed
  └─→ Impact: partial backup (DB/system OK)

Is Session Hanging = true
  └─→ Alert: running 12h+ (normal: 2-4h)
  └─→ Check: network, agent status, locks

Color Bar = "5555552222555"
  └─→ Pattern: fails every Mon (2,2,2)
  └─→ Theory: Monday network maintenance?
```

---

## 10. Performance Metrics

### Current

```
Sync Duration: ~30-60s (5000+ workloads)
API Calls: 2 (Device + GB EnumerateAccountStatistics)
Refresh Interval: 5 min (dashboard)
Storage: ~200MB for backup_status_cache (5000 rows)
Query: /cove/summary < 100ms (indexed)
```

### With Sprint 1 Changes

```
Sync Duration: ~35-70s (+5-10s for hanging detection)
API Calls: Still 2 (new logic is client-side)
Refresh Interval: 5 min (unchanged)
Storage: ~220MB (+20MB for new TEXT[] fields)
Query: v_backup_alerts < 200ms (new indexes)
```

### Potential Bottlenecks

```
GetPartnerInfoById calls: O(N) where N = unique PartnerId count
  → Can be cached, batched
  → Currently: ~50-100 calls per sync
  
Partner hierarchy resolution: recursive
  → Can time out se muitos níveis
  → Currently: max 3-4 níveis (manageable)
  
failed_data_sources array construction: O(M) where M = data sources
  → Currently: fixed 8 sources (TK, SK, FK, etc)
  → No performance impact
```

---

## 11. Migration Path

```
V1 (Today):          V2 (Sprint 1):        V3 (Sprint 2):
────────────         ───────────────        ───────────────

backup_status_cache  + last_session_status_code
(40 cols)            + failed_data_sources   + backup_workload_details
                     + is_session_hanging      (separate table)

                                            + size_breakdown_jsonb
                                            + enabled_data_sources[]

UI: Simple list      UI: Badges for         UI: Breakdown charts
    status code, sources           activity timeline

No alerts           Auto-alerts            SLA tracking
                   (hanging, quota)         forecasting
```

