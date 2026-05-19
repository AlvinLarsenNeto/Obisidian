---
title: Módulo Backup Cove + GLPI
tags: [backup, cove, glpi, edge-functions, automacao]
created: 2026-04-16
updated: 2026-04-16
status: active
type: permanent
---

# Módulo Backup Cove + GLPI

Monitora backups via Cove Backup API e abre/fecha chamados automaticamente no GLPI.
Cove e GLPI nunca se comunicam diretamente — tudo passa por 4 Edge Functions.

## Arquitetura

```
COVE API → Edge Functions → Supabase → GLPI API
```

## Edge Functions

| Função | Frequência | Responsabilidade |
|--------|-----------|-----------------|
| `cove` | pg_cron / sob demanda | Sync cache — busca todos os workloads |
| `backup-auto-tickets` | ~5min | Abre / followup / fecha chamados |
| `backup-resolve-normalized` | 30min | Fecha chamados normalizados |
| `backup-sync-glpi-status` | 30min | Detecta fechamentos manuais no GLPI |

## Tabelas Principais

- `backup_status_cache` — snapshot do estado de cada workload
- `backup_workload_incidents` — vínculo workload ↔ chamado GLPI
- `backup_sync_logs` — auditoria de cada ciclo
- `backup_workload_exceptions` — workloads ignorados (suporta wildcard `*`)
- `glpi_session_pool` — pool de sessões com circuit breaker
- `glpi_circuit_breaker` — 3 falhas → abre por 5min → half_open

## Autenticação Cove

- Protocolo JSON-RPC
- Endpoint: `https://api.backup.management/jsonrpcv1`
- Token "visa" temporário (TTL 30min), auto-refresh
- 2 portais: **Device** (`COVE_*`) e **GB/M365** (`COVE_GB_*`)
- Método: `EnumerateAccountStatistics` — até 5000 workloads por chamada

## Autenticação GLPI

- Headers: `App-Token` + `user_token` → retorna `session_token`
- Circuit breaker: 3 falhas → circuito aberto 5min

## Workload Key

Identificador único do vínculo workload → incidente:
```
{source}|{workload_name}
ex: "Cove Device|VSRV-DC01"
ex: "Cove GB|empresa.onmicrosoft.com"
```

## Regras de Abertura de Chamado

| Tipo | Condição | Urgência GLPI |
|------|----------|--------------|
| `CRITICO_24H` | sem sucesso ≥ 24h | 4 (Alta) |
| `ALERTA_ERROS` | `error_count > 0` | 3 (Média) |

**Não abre quando:**
- Workload em exceção (wildcard `*`)
- Dentro do buffer de 4h após horário agendado
- Empresa sem `glpi_entity_id`
- Já existe incidente `open` para o mesmo `workload_key + company_id`

**Duplicatas:** adiciona followup diário (max 1/dia via `last_followup_sent_on`)

## Regras de Fechamento

- **Uma execução bem-sucedida** é suficiente para fechar
- CRITICO_24H: fecha se `last_success < 24h` E `error_count == 0`
- ALERTA_ERROS: fecha se `error_count == 0`
- Fechamento manual detectado por `backup-sync-glpi-status`
- Reincidência: novo chamado com referência ao ticket anterior

## Campos Cove → GLPI

| Cove | Cache | Chamado |
|------|-------|---------|
| `AN/MN` | `workload_name` | Título |
| `D09F09` | `last_success` | "Último backup com sucesso" |
| `D09F06` | `last_session_error_count` | "Backup com N erro(s)" |
| `OT` | `workload_type` | Define `is_critical` |

## Workloads sem Empresa

Mapeados para sentinela `00000000-0000-0000-0000-000000000099`.
Visíveis na tela de reconciliação, não geram chamados.

## Referências
- [[projects/portal-asv]]
