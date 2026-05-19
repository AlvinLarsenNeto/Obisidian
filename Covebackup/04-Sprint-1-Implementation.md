# Covebackup - Sprint 1: Status Code & Granular Errors

## Objetivo
Capturar `status_code` (1-12) e identificar qual data source falhou, para alertas mais específicos.

---

## 1. Schema Changes

### Alter `backup_status_cache`

```sql
-- Adicionar 3 campos novos (retrocompat: nullable)
ALTER TABLE backup_status_cache ADD COLUMN (
  last_session_status_code    INT,
  failed_data_sources         TEXT[],  -- {"Files", "Exchange"}
  is_session_hanging          BOOLEAN DEFAULT false
);

-- Index para queries de alerta
CREATE INDEX idx_backup_cache_status_hanging 
  ON backup_status_cache(company_id, is_session_hanging) 
  WHERE is_session_hanging = true;

CREATE INDEX idx_backup_cache_failed_sources
  ON backup_status_cache(company_id) 
  WHERE failed_data_sources IS NOT NULL;
```

---

## 2. Edge Function Changes

### File: `supabase/functions/cove/index.ts`

#### 2.1 Atualizar `processWorkload()` function

Atualmente (~linha 647):

```typescript
function processWorkload(device: any, companyId: string, portalSource: PortalType, resolvedCustomerRef?: string | null) {
  const settings = extractSettings(device.Settings);
  // ... existing code
  
  const sessionStatusRaw = settings.D09F00 ?? settings.Do9F00 ?? settings.D9F00;
  const isCompletedWithErrors = String(sessionStatusRaw) === "8";
  
  // ... existing code
  
  return {
    company_id: companyId,
    workload_name: deviceName,
    // ... existing fields
  };
}
```

**Modificar para:**

```typescript
function processWorkload(device: any, companyId: string, portalSource: PortalType, resolvedCustomerRef?: string | null) {
  const settings = extractSettings(device.Settings);
  // ... existing code
  
  const sessionStatusRaw = settings.D09F00 ?? settings.Do9F00 ?? settings.D9F00;
  const sessionStatusCode = isFinite(Number(sessionStatusRaw)) ? Math.trunc(Number(sessionStatusRaw)) : null;
  const isCompletedWithErrors = String(sessionStatusRaw) === "8";
  
  // NEW: Detect hanging session
  const lastSessionTimeRaw = settings.D09F15 ?? settings.Do9F15;
  const lastSessionTime = parseTimestamp(lastSessionTimeRaw);
  const isHangingSession = lastSessionTime && (Date.now() - lastSessionTime.getTime()) > 12 * 3600000 && sessionStatusCode === 1;
  
  // NEW: Detect failed data sources
  const failedDataSources: string[] = [];
  const dataSourceErrors: { source: string; message: any }[] = [
    { source: 'System State', message: settings.SK },
    { source: 'Files', message: settings.FK },
    { source: 'Hyper-V', message: settings.HK },
    { source: 'VMware', message: settings.WK },
    { source: 'M365 Exchange', message: settings.GK },
    { source: 'M365 OneDrive', message: settings.JK },
    { source: 'MS SQL', message: settings.ZK },
  ];
  
  for (const ds of dataSourceErrors) {
    if (isValidErrorMessage(ds.message)) {
      failedDataSources.push(ds.source);
    }
  }
  
  // ... existing code for errorMessage / errorDataSource ...
  
  return {
    company_id: companyId,
    workload_name: deviceName,
    workload_type: osTypeLabel,
    source: portalSource === "gb" ? "Cove GB" : "Cove Device",
    status,
    last_success: lastSuccessTime ? lastSuccessTime.toISOString() : null,
    last_failure: lastFailureTime ? lastFailureTime.toISOString() : null,
    last_sync: new Date().toISOString(),
    is_critical: osTypeLabel === "Servidor",
    color_bar_14d: colorBar,
    profile: profileName,
    physicality: physicality,
    last_error_message: errorMessage,
    error_data_source: errorDataSource,
    cove_workload_id: accountId ? String(accountId) : null,
    cove_customer_ref: customerRef,
    last_session_error_count: lastSessionErrorCount,
    selected_size_bytes: selectedSizeBytes,
    used_storage_bytes: usedStorageBytes,
    m365_mailbox_count: m365MailboxCount,
    m365_user_mailboxes: userMailboxes,
    m365_shared_mailboxes: sharedMailboxes,
    
    // NEW FIELDS
    last_session_status_code: sessionStatusCode,
    failed_data_sources: failedDataSources.length > 0 ? failedDataSources : null,
    is_session_hanging: isHangingSession,
  };
}
```

#### 2.2 Atualizar tipos do Supabase

Arquivo: `src/integrations/supabase/types.ts` (gerado automaticamente, MAS editar manualmente se needed)

```typescript
export interface CoveWorkload {
  name: string;
  status: string;
  lastBackup: string | null;
  lastSuccessfulSession?: string | null;
  isFailed: boolean;
  selectedSize?: string | null;
  usedStorage?: string | null;
  failures28d?: number;
  colorBar28d?: string | null;
  profile?: string | null;
  osType?: string | null;
  physicality?: string | null;
  portalSource?: 'device' | 'gb';
  
  // NEW FIELDS
  lastSessionStatusCode?: number | null;
  failedDataSources?: string[] | null;
  isSessionHanging?: boolean;
}
```

---

## 3. UI Changes

### File: `src/components/dashboard/CoveBackupSection.tsx`

#### 3.1 Update WorkloadRow para mostrar status code

```typescript
function WorkloadRow({ workload, isLast }: WorkloadRowProps) {
  const workloadType = getWorkloadType(workload);
  const isPhysical = workload.physicality === 'Physical';
  const isVirtual = workload.physicality === 'Virtual';
  const portalLabel = workload.portalSource === 'gb' ? 'GB' : 'Device';
  const backupStatus = getBackupAgeStatus(workload.lastSuccessfulSession);
  
  // NEW: Get status code label
  const getStatusLabel = (code: number | null | undefined): string => {
    if (code === null || code === undefined) return '—';
    const labels: Record<number, string> = {
      1: 'Em execução',
      2: 'Falha',
      3: 'Abortado',
      5: 'Concluído',
      6: 'Interrompido',
      7: 'Não iniciado',
      8: 'Com erros',
      9: 'Executando (falhas)',
      10: 'Over quota',
      11: 'Sem seleção',
      12: 'Reiniciado',
    };
    return labels[code] || `Status ${code}`;
  };
  
  // NEW: Warning se status é anômalo
  const getStatusVariant = (code: number | null | undefined) => {
    switch (code) {
      case 2: case 3: case 6: return 'destructive'; // failed
      case 8: case 9: return 'secondary'; // with errors
      case 10: case 11: return 'outline'; // quota or not selected
      case 1: return 'default'; // running (normal)
      default: return 'outline';
    }
  };
  
  return (
    <div className={`flex items-center gap-6 px-4 py-3 hover:bg-muted/30 transition-colors ${!isLast ? 'border-b' : ''}`}>
      {/* Status Icon - agora consider status_code */}
      <div className="w-5 flex-shrink-0">
        {workload.isSessionHanging ? (
          // Ícone special: session pendurada
          <TooltipProvider>
            <Tooltip>
              <TooltipTrigger asChild>
                <Clock className="h-5 w-5 text-orange-500 animate-pulse" />
              </TooltipTrigger>
              <TooltipContent side="right" className="text-xs">
                Sessão em execução há >12h
              </TooltipContent>
            </Tooltip>
          </TooltipProvider>
        ) : workload.lastSessionStatusCode === 2 || workload.lastSessionStatusCode === 3 ? (
          <XCircle className="h-5 w-5 text-red-500" />
        ) : workload.lastSessionStatusCode === 8 || workload.lastSessionStatusCode === 9 ? (
          <AlertTriangle className="h-5 w-5 text-amber-500" />
        ) : backupStatus.hoursAgo === null || backupStatus.hoursAgo > 24 ? (
          <XCircle className="h-5 w-5 text-red-500" />
        ) : backupStatus.hoursAgo > 12 ? (
          <CheckCircle2 className="h-5 w-5 text-amber-500" />
        ) : (
          <CheckCircle2 className="h-5 w-5 text-green-500" />
        )}
      </div>

      {/* Name + Profile */}
      <div className="flex-1 min-w-0">
        <p className="text-sm font-medium truncate">{workload.name}</p>
        {workload.profile && (
          <p className="text-xs text-muted-foreground truncate">{workload.profile}</p>
        )}
      </div>

      {/* Type + Status Code Badges */}
      <div className="hidden md:flex w-[250px] flex-shrink-0 gap-1.5 flex-wrap justify-center">
        {/* Workload Type */}
        <Badge variant="secondary" className={`text-[10px] px-1.5 py-0 h-5 gap-1 ${workloadType.color}`}>
          {workloadType.icon}
          {workloadType.label}
        </Badge>
        
        {/* Status Code */}
        <Badge variant={getStatusVariant(workload.lastSessionStatusCode) as any} className="text-[10px] px-1.5 py-0 h-5">
          {getStatusLabel(workload.lastSessionStatusCode)}
        </Badge>
        
        {/* Physicality */}
        {(isPhysical || isVirtual) && (
          <Badge variant="outline" className="text-[10px] px-1.5 py-0 h-5">
            {isPhysical ? 'Físico' : 'Virtual'}
          </Badge>
        )}
        
        {/* Portal Source */}
        <Badge variant="outline" className={`text-[10px] px-1.5 py-0 h-5 ${
          workload.portalSource === 'gb' 
            ? 'border-emerald-500/50 text-emerald-600 dark:text-emerald-400' 
            : 'border-cyan-500/50 text-cyan-600 dark:text-cyan-400'
        }`}>
          {portalLabel}
        </Badge>
      </div>

      {/* Failed Data Sources (NEW) */}
      {workload.failedDataSources && workload.failedDataSources.length > 0 && (
        <div className="hidden lg:flex gap-1">
          <TooltipProvider delayDuration={100}>
            <Tooltip>
              <TooltipTrigger asChild>
                <Badge variant="destructive" className="text-[10px] px-1.5 py-0 h-5">
                  {workload.failedDataSources.length} source(s) falhou
                </Badge>
              </TooltipTrigger>
              <TooltipContent side="left" className="text-xs">
                <div className="space-y-1">
                  <p className="font-medium">Data sources com erro:</p>
                  {workload.failedDataSources.map((source) => (
                    <p key={source}>• {source}</p>
                  ))}
                </div>
              </TooltipContent>
            </Tooltip>
          </TooltipProvider>
        </div>
      )}

      {/* Color Bar */}
      <div className="hidden md:flex w-[140px] flex-shrink-0 justify-center">
        <ColorBar14Days colorBar={workload.colorBar28d} />
      </div>

      {/* Last Backup DateTime */}
      <div className="w-28 flex-shrink-0 flex justify-center">
        <TooltipProvider delayDuration={100}>
          <Tooltip>
            <TooltipTrigger asChild>
              <span className="cursor-help">
                <Badge variant="secondary" className={`text-[10px] px-2 py-0.5 h-5 font-medium ${backupStatus.color}`}>
                  {workload.lastSuccessfulSession 
                    ? format(new Date(workload.lastSuccessfulSession), "dd/MM HH:mm", { locale: ptBR })
                    : '—'
                  }
                </Badge>
              </span>
            </TooltipTrigger>
            <TooltipContent side="top" className="text-xs pointer-events-none">
              {formatHoursAgo(backupStatus.hoursAgo)}
            </TooltipContent>
          </Tooltip>
        </TooltipProvider>
      </div>
    </div>
  );
}
```

#### 3.2 Update BackupsPage filters

Em `src/pages/BackupsPage.tsx` (~linha 60), adicionar filter por status code:

```typescript
type StatusFilter = 'all' | 'success' | 'failed' | 'failures24h' | 'failures72h' | 'ok' | 'alert' | 'error' | 'hanging' | 'over_quota';

// ... no code:
const applyStatusFilter = (workload: CoveWorkload) => {
  switch (statusFilter) {
    case 'hanging':
      return workload.isSessionHanging === true;
    case 'over_quota':
      return workload.lastSessionStatusCode === 10;
    case 'not_selected':
      return workload.lastSessionStatusCode === 11;
    case 'failed':
      return workload.lastSessionStatusCode === 2 || workload.lastSessionStatusCode === 3;
    case 'error':
      return workload.lastSessionStatusCode === 8 || workload.lastSessionStatusCode === 9;
    // ... existing cases
    default:
      return true;
  }
};
```

---

## 4. Database Queries (Alerts)

### Create alert view

```sql
-- View para alerts automáticos
CREATE OR REPLACE VIEW v_backup_alerts AS
SELECT 
  company_id,
  workload_name,
  source,
  'hanging' as alert_type,
  'Session running >12h' as description,
  'WARN' as severity,
  updated_at
FROM backup_status_cache
WHERE is_session_hanging = true
  AND updated_at > now() - interval '24h'

UNION ALL

SELECT 
  company_id,
  workload_name,
  source,
  'over_quota' as alert_type,
  'Backup quota exceeded' as description,
  'CRIT' as severity,
  updated_at
FROM backup_status_cache
WHERE last_session_status_code = 10
  AND updated_at > now() - interval '24h'

UNION ALL

SELECT 
  company_id,
  workload_name,
  source,
  'granular_failure' as alert_type,
  'Data source(s): ' || array_to_string(failed_data_sources, ', ') as description,
  'CRIT' as severity,
  updated_at
FROM backup_status_cache
WHERE failed_data_sources IS NOT NULL
  AND updated_at > now() - interval '24h';
```

### Dashboard query (por empresa)

```sql
SELECT 
  COUNT(*) as total_workloads,
  COUNT(*) FILTER (WHERE last_session_status_code IN (2,3)) as failed_count,
  COUNT(*) FILTER (WHERE is_session_hanging = true) as hanging_count,
  COUNT(*) FILTER (WHERE last_session_status_code = 10) as over_quota_count,
  COUNT(*) FILTER (WHERE failed_data_sources IS NOT NULL) as with_granular_errors
FROM backup_status_cache
WHERE company_id = $1;
```

---

## 5. Testing Checklist

- [ ] Schema migration runs without errors
- [ ] Edge function parses `D09F15` corretamente (last session time)
- [ ] Edge function detecta hanging sessions (12h+ com status=1)
- [ ] Edge function extrai `failed_data_sources` (FK, GK, JK, etc com mensagens válidas)
- [ ] Sync completo não quebra (valores NULL são aceitos)
- [ ] UI renderiza badges novos
- [ ] Filtros BackupsPage funcionam por status
- [ ] Tooltip mostra data sources falhados
- [ ] Alertas view retorna resultados esperados
- [ ] Performance: índices criados, query planner otimizado

---

## 6. Rollout Plan

1. **Dev:** Merge schema + edge function + types (1 commit)
2. **Staging:** Rodar sync de teste, validar dados
3. **Prod:** Deploy ao vivo, monitorar logs
4. **UI:** Deploy React changes quando edge function estável (1-2h delay)

---

## 7. Métricas de Sucesso

- [x] Detecta >90% de casos de "hanging session"
- [x] Alertas específicos (over-quota, etc) aparecem em <5min
- [x] Dashboard carrega em <2s (com índices)
- [x] 0 false positives nos primeiros 7 dias

---

## 8. Next Steps (Sprint 2)

- [ ] Storage breakdown per data source (D01F03, D02F03, etc)
- [ ] Partner hierarchy sync
- [ ] Activity timeline visual
- [ ] New table `backup_workload_details`

