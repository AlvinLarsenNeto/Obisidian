# QUICKREF - Resumo de Campos (30 segundos)

> **Mapa mental rápido: O que é, está faltando, como fix**

---

## 🎯 EM UMA FRASE

A API Covebackup retorna **112+ campos**. O módulo captura **20 (~20%)**.  
**Faltam:** Status code numérico, erros por source, last session time, storage breakdown.

---

## 📊 TABELA COMPARATIVA

### Status & Timestamps (5 campos)

| Campo | Capturado | Uso | Sprint |
|-------|-----------|-----|--------|
| **D09F00** | ❌ | Diferencia Failed(2), OverQuota(10), NotSelected(11) | 1 |
| **D09F15** | ❌ | Última tentativa (detecta hanging >12h) | 1 |
| **D09F09** | ✅ | Último backup OK | — |
| **D09F07** | ✅ | Última falha | — |
| **D09F08** | ✅ | Color bar 14 dias | — |

### Erros (9 campos)

| Campo | Capturado | Tipo | Sprint |
|-------|-----------|------|--------|
| **D09F19** | ✅ | General error (genérico) | — |
| **SK** | ❌ | System State (individual) | 1 |
| **FK** | ❌ | Files (individual) | 1 |
| **HK** | ❌ | Hyper-V (individual) | 1 |
| **WK** | ❌ | VMware (individual) | 1 |
| **GK** | ❌ | M365 Exchange (individual) | 1 |
| **JK** | ❌ | M365 OneDrive (individual) | 1 |
| **ZK** | ❌ | MS SQL (individual) | 1 |
| **TK** | ❌ | Total (individual) | 1 |

### Tamanho (9 campos)

| Campo | Capturado | Uso | Sprint |
|-------|-----------|-----|--------|
| **D09F03** | ✅ | Total selected | — |
| **D01F03** | ❌ | Files & Folders | 2 |
| **D02F03** | ❌ | System State | 2 |
| **D08F03** | ❌ | VMware | 2 |
| **D14F03** | ❌ | Hyper-V | 2 |
| **D19F03** | ❌ | M365 Exchange | 2 |
| **D20F03** | ❌ | M365 OneDrive | 2 |
| **I14** | ✅ | Used storage | — |

### Outros

| Campo | Capturado | Uso | Sprint |
|-------|-----------|-----|--------|
| **PN** | ❌ | Partner name (hierarchy) | 2 |
| **AP** | ❌ | Active products (VM detection) | 2 |
| **OT** | ✅ | OS Type | — |
| **I81** | ✅ | Physicality | — |
| **OP** | ✅ | Profile | — |
| **D19F20/21** | ✅ | M365 Mailboxes | — |

---

## 🎯 TOP 3 IMPLEMENTAR (Sprint 1)

### 1️⃣ Status Code (D09F00)
```
Esforço:  ⬜ Baixo
Impacto:  ⬜⬜⬜⬜ Alto
Tempo:    1 dia
```
**Por quê:** Diferencia "Over Quota" de "Não selecionado" sem esperar >48h

### 2️⃣ Last Session Time (D09F15)
```
Esforço:  ⬜ Baixo
Impacto:  ⬜⬜⬜ Médio
Tempo:    1 dia
```
**Por quê:** Detecta sessões hanging (rodando >12h)

### 3️⃣ Erros por Source (SK/FK/GK/JK)
```
Esforço:  ⬜⬜ Médio
Impacto:  ⬜⬜⬜⬜ Alto
Tempo:    2 dias
```
**Por quê:** "Exchange falhou mas Files OK" → diagnóstico rápido

---

## 💡 EXEMPLOS

### ANTES (sem Sprint 1)
```
Status: 🔴 CRITICAL
Last: 7 days ago
Error: "Backup failed"
→ Técnico: "De novo??? Vou abrir ticket"
```

### DEPOIS (com Sprint 1)
```
Status Code: 10 (Over Quota)
Last attempt: 2 hours ago
Last success: 7 days ago

Failed sources:
  ❌ M365 Exchange (quota exceeded 800/750 GB)
  ⏳ M365 OneDrive (running 3h)
  ✅ Files (OK)
  ✅ System State (OK)

→ Técnico: "Entendi. Aumentar quota Exchange, PRONTO em 30min"
```

---

## 📋 SCHEMA CHANGES (Sprint 1)

```sql
ALTER TABLE backup_status_cache ADD COLUMN (
  last_session_status_code INT,      -- D09F00
  failed_data_sources TEXT[],        -- ["Files", "Exchange"]
  is_session_hanging BOOLEAN         -- true se status=1 AND elapsed>12h
);
```

---

## 🚀 ROADMAP

| Sprint | O quê | Semanas |
|--------|-------|---------|
| **1** | Status code, Last session, Error sources | 1-2 |
| **2** | Storage breakdown, Partner hierarchy | 2-4 |
| **3+** | Session history, SLA, Forecasting | TBD |

---

## 🔗 PRÓXIMAS LEITURAS

- **Campos detalhados:** [[01-API-Fields]]
- **O que módulo usa:** [[02-Current-Implementation]]
- **Como implementar:** [[04-Sprint-1-Implementation]]
- **Visão completa:** [[03-Opportunities]]

---

**Tempo de leitura:** 3 min  
**Última atualização:** 2026-05-05
