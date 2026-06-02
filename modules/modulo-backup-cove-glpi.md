---
title: Módulo Backup Cove + GLPI
tags: [backup, cove, glpi, edge-functions, automacao]
created: 2026-04-16
updated: 2026-05-29
status: active
type: permanent
---

# Módulo Backup Cove + GLPI

Monitora backups via Cove Backup API e abre/fecha chamados automaticamente no GLPI.
Cove e GLPI nunca se comunicam diretamente — tudo passa por Edge Functions.

## ⚡ TL;DR (ler só isto pra 90% das perguntas)

- **Fluxo:** `COVE API → Edge Functions → Supabase → GLPI`. 10 funções `backup-*` + `cove` (sync).
- **Motor de abertura:** matriz **D09F00** via RPC `should_open_incident` (atrás da flag `use_d09f00_matrix`). Code 2=Failed(crít), 10=OverQuota(crít), 8/6=erros, 11=sem seleção, **5=OK→skip**, 1/9/12=rodando→skip, 3=abortado→skip.
- **Cron `backup-auto-tickets`:** diário **04:00 UTC** (01:00 BRT).
- **Tabelas-chave:** `backup_status_cache` (snapshot), `backup_workload_incidents` (vínculo c/ ticket), `backup_feature_flags`.
- **`workload_key` = `{source}|{workload_name}`** (ex: `Cove Device|SERVER`).
- **Ponto cego:** code=5 sempre dá skip → não detecta "agente parou de reportar" (staleness). Tratado por override `stale_no_session` na edge function.
- **Não abre quando:** exceção wildcard, buffer 4h, empresa sem `glpi_entity_id` OU sem `parent_company_id`, incidente `open` já existe.
- Detalhe completo abaixo. Mapeamento dos 112 campos da API → [[../Covebackup/INDEX]].

> **Estado verificado em 2026-05-29** contra `github.com/gustavofey-asv/portalasv` `origin/main`
> (commit `bac7069a`). O subsistema D09F00 + fila de e-mail + relatório semanal + flags
> entrou entre 25-29/05. Valores de runtime (flags ligadas/desligadas, cron ativo) **só o
> banco confirma** — GitHub mostra o seed/migration, não o estado vivo.

## Arquitetura

```
COVE API → Edge Functions → Supabase → GLPI API
```

## Edge Functions (10 backup-* + cove)

| Função | Frequência | Responsabilidade |
|--------|-----------|-----------------|
| `cove` | pg_cron / sob demanda | Sync cache — busca todos os workloads |
| `backup-auto-tickets` | **diário 04:00 UTC** (`0 4 * * *`) | Abre / followup / fecha chamados |
| `backup-resolve-normalized` | 30min | Fecha chamados normalizados |
| `backup-sync-glpi-status` | 30min | Detecta fechamentos manuais no GLPI |
| `backup-close-weekend-tickets` | fim de semana / segunda | Fecha tickets de perfil SEG-SEXTA c/ último sucesso na sexta |
| `backup-close-exception-tickets` | — | Fecha tickets de workloads colocados em exceção |
| `backup-exceptions-expire` | — | Expira exceções temporárias |
| `backup-ticket-lookup` | sob demanda | Consulta de ticket por workload |
| `backup-ticket-reconcile` | — | Reconcilia incidente ↔ ticket GLPI |
| `backup-ticket-verify` | — | Verifica estado real do ticket no GLPI |
| `backup-weekly-report` | semanal | Relatório agregado por empresa |

> ⚠️ Histórico do cron `backup-auto-tickets`: era 15min → mudou p/ diário 10:05 UTC
> (migration 20260114) → **religado p/ `0 4 * * *` (04:00 UTC = 01:00 BRT)** na migration
> 20260528170113. A doc antiga dizia "~5min" — **incorreto**.

## Tabelas Principais

- `backup_status_cache` — snapshot do estado de cada workload
- `backup_workload_incidents` — vínculo workload ↔ chamado GLPI
- `backup_sync_logs` — auditoria de cada ciclo
- `backup_workload_exceptions` — workloads ignorados (suporta wildcard `*`)
- `glpi_session_pool` — pool de sessões com circuit breaker
- `glpi_circuit_breaker` — 3 falhas → abre por 5min → half_open

### Tabelas novas (subsistema D09F00 / e-mail / relatório — mai/2026)

- `backup_feature_flags` — flags de rollout (ver abaixo)
- `pending_backup_emails` — fila de e-mails de incidente
- `email_dry_run_log` — log de e-mails que *seriam* enviados (dry-run)
- `backup_report_settings` / `backup_report_audit` / `backup_report_dispatch_log` — relatório semanal
- `backup_pre_fix_baseline` — baseline de métricas pré-correção
- `backup_backfill_snapshot` — snapshot antes de backfill de incidentes

### Colunas novas

- `backup_status_cache`: `last_session_status_code`, `last_session_status_at`, `last_session_status_raw`
- `backup_workload_incidents`: `opened_status_code`, `is_recurring_suppressed`, `suppressed_since`, `suppression_reason`, `close_reason`

## Feature Flags (`backup_feature_flags`)

Seed (migration 20260525185854) — todas `false` exceto `email_dry_run`:

| key | seed | papel |
|-----|------|-------|
| `use_d09f00_matrix` | false* | usa matriz D09F00 p/ decidir abertura (vs regra antiga) |
| `use_24h_idempotency` | false | janela 24h anti-duplicata |
| `use_email_queue` | false | usa `pending_backup_emails` |
| `email_dry_run` | **true** | não envia e-mail real, só loga |
| `use_aggregated_weekly_report` | false | relatório semanal agregado |
| `block_personal_email_domains` | false | bloqueia gmail/hotmail etc |

> *Seed é `false`, mas em runtime (29/05) o banco estava com `use_d09f00_matrix = true` —
> ligado manualmente. **Só o banco sabe o estado atual de cada flag.**

## Motor de Abertura — Matriz D09F00 (`should_open_incident`)

Quando `use_d09f00_matrix=true`, a edge function chama a RPC
`should_open_incident(p_company_id uuid, p_workload_key text, p_new_code smallint) → jsonb`.

Mapa de severidade por status code D09F00:

| code | significado | severidade | ação |
|------|-------------|-----------|------|
| 1 / 9 / 12 | InProcess / InProgressWithFaults / Restarted | — | `skip` running_or_restarted |
| 5 | Completed (OK) | — | `skip` **ok_status** |
| 3 | Aborted | — | `skip` backup_aborted (+ fecha incidente aberto) |
| 11 | NoSelection | 1 | abre (alerta sem e-mail) |
| 8 / 6 | CompletedWithErrors / Interrupted | 2 | abre (alerta sem e-mail) |
| 2 | Failed | 3 | abre (crítico + e-mail) |
| 10 | OverQuota | 4 | abre (crítico + e-mail) |
| outro | desconhecido | — | `skip` unknown_status_code |

Regras extras da matriz:
- **Supressão:** ≥3 incidentes resolvidos em <1h nos últimos 3 dias → `suppress` (`recurring_short_resolve_3in3d`).
- **Idempotência 24h:** se há incidente aberto nas últimas 24h → `escalate` se severidade subiu, senão `skip within_24h_no_escalation`.
- **Caso base:** sem incidente recente → `open first_occurrence_or_24h_window`.

### ⚠️ Ponto cego conhecido: staleness (agente parou de reportar)

`code 5` retorna **sempre** `skip / ok_status` — a matriz só olha o **último code recebido**,
não detecta **ausência de novas sessões**. Se o agente para de reportar após um sucesso, o
wallboard marca crítico (conta horas), mas a matriz fica em silêncio. 

Tratamento: **override de staleness na edge function** (`backup-auto-tickets`, commit `bac7069a`
"Corrigiu staleness no backfill"): quando a matriz devolve `skip/ok_status` mas `last_success`
está velho além da janela do perfil → promove para `open` com reason `stale_no_session`. Respeita
`parseBackupProfile` / `isBackupWithinSchedule` (não dispara fim de semana p/ perfil SEG-SEXTA).

> 🔴 **Incidente 30/05–01/06 — flood de 127 falsos positivos.** O `staleness_guard`
> (`index.ts:1335-1353`, `STALE_THRESHOLD_HOURS=24`, **hardcoded, sem circuit-breaker global**)
> disparou em massa porque o **`cove_sync` parou em 29/05 19:33** → `backup_status_cache.last_sync`
> congelou em 715 workloads → todos ficaram `last_success > 24h` → cron 04:00 promoveu
> `skip/ok_status`(code 5) → `open` sev 3. Volume: 30/05=16, **31/05=105**, 01/06=6. Tickets GLPI
> #2026052970–#2026060130, todos `opened_status_code=5`. Opener = conta de serviço `GLPI_USER_TOKEN`.
> Idempotência/matriz/GLPI OK — causa única = sync morto + guard sem freio.
> **Causa raiz FINAL (confirmada 01/06):** mismatch de **`X-Cron-Secret`**. O comando do
> `cron.job` jobid=29 (`cove-backup-sync`, `0,15,30,45 * * * *`) mandava `f7b3c8d2-...`, mas o env
> `CRON_SECRET` sempre valeu `@$V.sup0208` (mesmo dos crons 61–65). A edge `cove` (linhas 1671-1682)
> compara e devolve **403 "Invalid cron secret"** → `cove/sync-all` nunca executa → cache congela.
> NÃO foi rotação nem credencial Cove externa — foi comando do cron escrito com valor errado em
> 29/05. O cron marca `succeeded` em `cron.job_run_details` porque só mede o `net.http_post`
> enfileirado, não o 403 da resposta. **Fix (01/06):** alinhou jobid 29+31 ao valor real do env
> (`@$V.sup0208`); cove voltou a sincronizar (716 workloads). `update_secret` p/ rotacionar não
> persistiu em prod → **dívida: rotação completa dos 7 jobs agendada** (`mem/security/cron-secret-rotation-debt.md`).
> **Bug secundário:** `backup-auto-tickets` deixou `backup_sync_logs` em `status='running'` eterno
> (31/05, 01/06) — exceção engolida no loop de criação, sem try/finally que feche o registro (corrigido em P2).
> **Lição estrutural:** `cron.job_run_details = succeeded` é falso-positivo de saúde — só confirma o
> POST enfileirado, não o status HTTP da edge. Monitorar `net._http_response` p/ 4xx/5xx, não o cron.
>
> **Limpeza dos 127 (01/06 ~16:56–16:59 UTC) — auto-fechamento não-planejado.** Antes de revalidar
> por idade-de-sessão, o cron **jobid 53 `backup-ticket-reconcile-15min`** (`*/15 * * * *`) disparou
> internamente `auto_resolve` (`backup-resolve-normalized`) e resolveu **125** incidentes (code 5).
> ⚠️ Cron escondido: nome não tem "resolve" mas fecha tickets. A função `auto_resolve` **tem
> discriminação própria** — protegeu 3/4 legítimos (code 2/8: BOCA GRANDE, MAINHARDT, BRUSTEC
> ficaram `open`) mas **furou no LIOTTO** (code 1 in-progress, fechado indevido). Critério dela =
> presença de **erro na última sessão**, NÃO **idade da sessão** → não trata staleness real (fica
> coberto pela rede de segurança: staleness_guard re-abre no 04:00 seguinte se o agente está
> mesmo morto). `close_reason` ficou NULL (gap de rastreabilidade). **Inconsistência DB↔GLPI:**
> 13 tickets `resolved` no DB mas `open` no GLPI — `solveTicket` falhou 13× `ERROR_GLPI_ADD 400
> "Você não tem permissão"` + 1× `ERROR_SESSION_TOKEN_INVALID 401` (sessão morreu no meio do batch;
> run estourou IDLE_TIMEOUT 150s → 504). Cron 53 pausado (`cron.alter_job(53, active:=false)`)
> durante a reconciliação. **Lição:** mapear TODOS os crons que fecham ticket (53 reconcile + 64
> auto-tickets), não só os com "resolve"/"close" no nome.
>
> **2º bug estrutural — tickets órfãos sem `bwi` (descoberto 01/06).** Reconciliação 3-camadas
> (GLPI MySQL vivo ↔ `backup_workload_incidents` ↔ `glpi_tickets_cache`/`backup_ticket_verifications`)
> no range do flood: **145 open / 125 closed** no GLPI vivo. Tela de conferência **NÃO estava stale**
> — cache/btv/MySQL alinhados (cadeia `glpi-mysql-sync` jobid 35 `*/10` + `glpi-bridge-sync` jobid 42
> `*/2` → `glpi_tickets_cache` → `backup_ticket_verifications` via `backup-ticket-reconcile` jobid 53).
> **Achado-chave:** **141 dos 145 open NÃO existem em `bwi`** — órfãos. `auto_resolve`/`backup-resolve-normalized`
> operam sobre `bwi` → cegos a eles → nunca auto-fecham. Origem provável: `backup-auto-tickets` **criou
> o ticket no GLPI mas morreu antes de gravar `bwi`** (mesma exceção engolida do log `running` eterno).
> **Correção estrutural:** inserir `bwi` **logo após cada `createTicket`** (atômico por ticket), não em
> batch no fim do loop. **Limpeza:** os 141 precisam de fechamento fora do fluxo `bwi` (match
> company+workload pelo título → revalida code no cove → fecha falso-positivo com read-back MySQL).
> **Quem abriu (final):** `portal.grupoasv` (id 1324, `GLPI_USER_TOKEN`) — token compartilhado, fechamento
> manual via REST não deixa rastro (`glpi_logs.linked_action=0`); dívida de rotação junto do CRON_SECRET.
> **Lições:** (1) guard de staleness precisa de circuit-breaker: se `max(last_sync) > 12h`, abortar
> ciclo (`status=aborted`, reason `cove_sync_stale`) e **não** chamar `createTicket`; (2) emitir 1
> alerta no abort (senão troca flood por silêncio); (3) flag `enable_staleness_guard` p/ kill-switch
> sem deploy; (4) fechamento de falsos positivos só **após** religar cove_sync e revalidar contra
> cache fresco — nunca fechar cego (cache congelado esconde falha real pós-29/05).
>
> **Resolução (01/06).** Fechados **135** órfãos falso-positivo (code=5) em batches de 25 com read-back
> no MySQL vivo, `close_reason='false_positive_orphan_staleness'`. Mantidos abertos 3 LEGITIMO (BOCA
> GRANDE #2026060065, BRUSTEC #2026060082, MAINHARDT #2026060072 — falha real code 2/8). Fase 2
> deployada: (a) **fix bwi** — enum `backup_incident_status += pending_glpi`, helper `reserveAndCreateBwiTicket`
> (pré-reserva `pending_glpi` → createTicket → promove `open`; rollback no fail) + sweeper
> `sweepStalePendingGlpi` (deleta `pending_glpi` órfão >5min); (b) **fix Cove D09F15** —
> `last_session_status_at = parseTimestamp(D09F15)` (antes `new Date()`), agora diverge de `last_sync`
> → **staleness real fica visível** (ex: ASV-APP01 parado desde 01:52, 968min). **3º bug — mapeamento
> entidade↔customer:** ticket aberto contra entidade GLPI `ASV CONSULTORIA` enquanto o workload no Cove
> pertence a `ASV SISTEMAS` → SEM_MATCH no cruzamento. Falso positivo por desalinhamento de entidade
> (dívida, não urgente). Campo Cove **`D09F15` = timestamp real da última sessão** (lastSessionTimeRaw).

## Regra antiga (quando `use_d09f00_matrix=false`)

| Tipo | Condição | Urgência GLPI |
|------|----------|--------------|
| `CRITICO_24H` | sem sucesso ≥ 24h | 4 (Alta) |
| `ALERTA_ERROS` | `error_count > 0` | 3 (Média) |

**Não abre quando:** workload em exceção (wildcard `*`); dentro do buffer de 4h após horário
agendado; empresa sem `glpi_entity_id` **OU sem `parent_company_id`** (ambos filtrados na query);
já existe incidente `open` para `workload_key + company_id`.

> Nota: a query de `backup-auto-tickets` exige `glpi_entity_id IS NOT NULL` **E**
> `parent_company_id IS NOT NULL`. Sem qualquer um, o workload é descartado silenciosamente.

## Autenticação Cove

- Protocolo JSON-RPC. Endpoint: `https://api.backup.management/jsonrpcv1`
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

> ⚠️ Armadilha: o **nome do workload pode mudar no Cove** (ex: `SERVER_xpwah` → `SERVER`),
> gerando um `workload_key` novo. Incidentes antigos ficam órfãos sob o key antigo e o novo
> nunca casa com eles. Visto no caso ETIKJU (mai/2026).

## Regras de Fechamento

- **Uma execução bem-sucedida** é suficiente para fechar
- CRITICO_24H: fecha se `last_success < 24h` E `error_count == 0`
- ALERTA_ERROS: fecha se `error_count == 0`
- Fechamento manual detectado por `backup-sync-glpi-status`
- Reincidência: novo chamado com referência ao ticket anterior

## Workloads sem Empresa

Mapeados para sentinela `00000000-0000-0000-0000-000000000099`.
Visíveis na tela de reconciliação, não geram chamados.

## Referências
- [[projects/portal-asv]]
- Código: `supabase/functions/backup-*` e `supabase/migrations/2026052*` (GitHub `gustavofey-asv/portalasv`)
