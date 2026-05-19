# 🚀 OPORTUNIDADES DE MELHORIA

> **Roadmap: Sprint 1 → Sprint 2 → Sprint 3+ Backlog**  
> Referência: [[01-API-Fields]] | [[04-Sprint-1-Implementation]]

---

## 📌 VISÃO GERAL

Hoje o módulo captura ~20 dos 112+ campos da API.  
As oportunidades abaixo estão priorizadas por **impacto** + **esforço**.

**Meta Sprint 1:** 3 campos críticos → alertas melhores em 1-2 semanas

---

## 🔴 SPRINT 1 - CRÍTICO (1-2 semanas)

> **O que:** Capturar 3 campos + UI badges  
> **Por quê:** Alertas específicos sem esperar >48h  
> **Tempo:** 4-5 dias úteis  

### A. Capturar Status Code (D09F00)

**O que é:**
```
Código numérico 1-12 que diferencia a situação:
  1  = Running (em execução)
  2  = Failed (falhou)
  5  = OK (concluído bem)
  8  = Completed with Errors
  10 = Over Quota (STORAGE CHEIO!)
  11 = Not Selected (configuração quebrada)
  12 = Restarted (reiniciado)
```

**Impacto:**
- ✅ Alertas específicos: "Over Quota" sem esperar >48h
- ✅ Diferencia "rodando normalmente" de "rodando há dias (hanging)"
- ✅ Recomendações automáticas por status

**Esforço:** ⬜ Baixo (1 campo + 2 linhas código)

**Benefício/Tempo:** 100x (melhor acurácia com tempo mínimo)

---

### B. Capturar Last Session Time (D09F15)

**O que é:**
- Timestamp da **última tentativa** de backup (qualquer resultado)
- Diferente de D09F09 (último sucesso)

**Diferença crítica:**
```
D09F09 (Last Success) = "há 7 dias" (sucesso)
D09F15 (Last Attempt)  = "há 2 horas" (tentou agora, mas falhou)

Interpretação:
  Antes: "7 dias sem backup!" (alerta)
  Depois: "Tentou há 2h mas falhou" (problema agora, não desgosto)
```

**Impacto:**
- ✅ Detectar **hanging sessions** (rodando >12h = acende 🔴)
- ✅ Diferença: "sem sucesso" ≠ "sem tentativa"
- ✅ Triage: "problema recente" vs "nunca funcionou"

**Esforço:** ⬜ Baixo (1 campo + detecção)

**Benefício/Tempo:** 50x

---

### C. Extrair Erros por Data Source (SK/FK/GK/JK/ZK)

**O que é:**
- Hoje: captura PRIMEIRA mensagem genérica
- Sprint 1: extrair qual data source específico falhou
```
SK = System State error
FK = Files & Folders error
HK = Hyper-V error
WK = VMware error
GK = M365 Exchange error ← muitas falhas aqui
JK = M365 OneDrive error
ZK = MS SQL error
TK = Total error
```

**Impacto:**
- ✅ Saber: "M365 Exchange falhou, mas Files OK" (vs "tudo falhando")
- ✅ Diagnóstico: root cause muito mais rápido
- ✅ Recomendação: "Reiniciar Exchange connector" vs "Troubleshoot geral"

**Esforço:** ⬜⬜ Médio (loop + array + badge)

**Benefício/Tempo:** 200x (50% redução troubleshooting)

---

### Checklist Sprint 1

```sql
-- Schema
ALTER TABLE backup_status_cache ADD COLUMN (
  last_session_status_code INT,      -- D09F00
  failed_data_sources TEXT[],        -- ["Files", "Exchange"]
  is_session_hanging BOOLEAN         -- true se status=1 AND elapsed>12h
);

-- Index
CREATE INDEX idx_hanging ON backup_status_cache(company_id) 
  WHERE is_session_hanging = true;
```

**Veja implementação:** [[04-Sprint-1-Implementation]]

---

## 🟡 SPRINT 2 - IMPORTANTE (2-4 semanas)

> **O que:** Storage breakdown + Partner hierarchy  
> **Por quê:** Otimização + auto-mapping  
> **Tempo:** 10-14 dias úteis  

### A. Storage Breakdown (D01F03...D20F03)

**O que é:**
- Hoje: total de storage selecionado (genérico)
- Sprint 2: breakdown por data source
```
Files & Folders:     200 GB (D01F03)
System State:         50 GB (D02F03)
VMware:              300 GB (D08F03)
Hyper-V:             150 GB (D14F03)
M365 Exchange:       500 GB (D19F03)  ← maior consumidor
M365 OneDrive:       300 GB (D20F03)
MS SQL:              200 GB (D09F03 - não separado)
────────────────────────────────
TOTAL:             1,700 GB
```

**Impacto:**
- ✅ Pie chart no dashboard: "Qual tipo consome mais?"
- ✅ Trend: "Exchange cresceu 100GB em 3 meses"
- ✅ Forecasting: "Vai exceder quota em 2 meses, aumentar agora"

**Esforço:** ⬜⬜ Médio (nova tabela + UI chart + queries)

**Novo Schema:**
```sql
CREATE TABLE backup_workload_details (
  id UUID PRIMARY KEY,
  company_id UUID,
  workload_name TEXT,
  data_source TEXT,  -- "Files", "Exchange", etc
  size_bytes BIGINT,
  last_error TEXT,
  ...
  UNIQUE(company_id, workload_name, data_source)
);
```

---

### B. Partner Hierarchy Sync (PN, ParentId)

**O que é:**
- Hoje: customer name via AR (string, pode mudar)
- Sprint 2: usar PartnerId (número, nunca muda)

**Impacto:**
- ✅ Auto-mapping mesmo se customer name muda
- ✅ Detectar: resellers vs clientes diretos vs subunidades
- ✅ Auditoria: histórico de mapeamentos

**Esforço:** ⬜⬜ Médio (recursão + cache)

---

### C. Activity Timeline Visual

**O que é:**
- Timeline histórico dos últimos 30 dias
```
[✅] [✅] [❌] [✅] [❌] [❌] [⏳]
 1   2    3    4    5    6    7  (dias)

Hover: "Jan 15: Completed with Errors"
```

**Impacto:**
- ✅ Ver padrão: "Falha toda 2ª feira? Network maintenance?"
- ✅ Contexto: "Começou a falhar dia 15, antes era ok"

**Esforço:** ⬜⬜ Médio (UI + historicização)

---

## 🟢 BACKLOG - NICE-TO-HAVE

### Session History (TBD - requer permissão Cove)

**O que seria:** Histórico de últimas 10 sessões (duração, bytes, status)

**Bloqueador:** Credenciais Cove sem permissão  
**Timeline:** TBD

---

### SLA Dashboard

**O que seria:**
```
Backup SLA Target:
  - Max 24h sem sucesso
  - Max 48h sem tentativa
  - Max 3 falhas consecutivas

Métricas:
  - Uptime: 99.2% (últimos 30d)
  - MTTR: 2.3h (mean time to recover)
  - Cost per GB: R$ 0.50
```

**Esforço:** ⬜⬜⬜ Alto  
**Timeline:** 4-5 semanas pós-Sprint 2

---

### Forecasting & Capacity Planning

**O que seria:**
- Storage growth rate: "10GB/mês"
- Projeção: "Vai exceder quota em 3 meses"
- Ação: "Aumentar quota agora (lead time 2 semanas)"

**Depende de:** Sprint 2 Storage Breakdown

**Timeline:** 3 semanas pós-Sprint 2

---

### Real-Time Webhooks

**O que seria:**
- Hoje: poll a cada 5 min (overhead)
- Ideal: Cove envia webhook quando status muda (0s latency)

**Bloqueador:** API Cove talvez não suporte  
**Timeline:** TBD (investigação)

---

## 📊 MATRIZ PRIORIZAÇÃO

| Oportunidade | Complexidade | Impacto | Timeline | Sprint | Benefício |
|---|---|---|---|---|---|
| **Status Code** | ⬜ | ⬜⬜⬜⬜ | 1d | 1 | 100x |
| **Last Session** | ⬜ | ⬜⬜⬜ | 1d | 1 | 50x |
| **Error by Source** | ⬜⬜ | ⬜⬜⬜⬜ | 2d | 1 | 200x |
| **Size Breakdown** | ⬜⬜ | ⬜⬜⬜ | 3w | 2 | 30x |
| **Partner Hier.** | ⬜⬜ | ⬜⬜ | 3w | 2 | 10x |
| **Activity Timeline** | ⬜⬜ | ⬜⬜ | 2w | 2 | 20x |
| **Session History** | ⬜⬜⬜ | ⬜⬜ | TBD | 3+ | 15x |
| **SLA Dashboard** | ⬜⬜⬜ | ⬜⬜⬜ | 4-5w | 3+ | 40x |

---

## 🎯 RECOMENDAÇÃO

**Comece por Sprint 1** (Status Code + Last Session + Error Sources)

→ 4 dias úteis  
→ Alertas 50-200x melhores  
→ Foundation para Sprint 2

**Depois Sprint 2** (quando Sprint 1 estável)

→ Storage insights + Forecasting  
→ Compliance + Capacity planning

---

## 📚 PRÓXIMAS LEITURAS

- **Como implementar:** [[04-Sprint-1-Implementation]]
- **Campos detalhados:** [[01-API-Fields]]
- **Arquitetura:** [[05-Architecture]]
- **Quick ref:** [[00-QUICKREF]]

---

**Última atualização:** 2026-05-05  
**Status:** Pronto para Sprint 1
