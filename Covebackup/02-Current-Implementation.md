# Covebackup API - Resumo de Campos para Comparação

> **Uso:** Compare esta lista com `backup_status_cache` para identificar campos não utilizados.

---

## 📋 CAMPOS RETORNADOS PELA API (112+)

### 🔵 IDENTIFICAÇÃO (8 campos)
```
AN      = Account Name (nome do dispositivo/servidor)
MN      = Machine Name (nome alternativo)
AR      = Account Reference (ID/nome do cliente)
PN      = Partner Name (reseller intermediário, se houver)
I0      = AccountId (ID único no Cove, para links diretos)
OT      = OS Type (1=Workstation, 2=Servidor, null=M365)
I81     = Physicality (1=Físico, 2=Virtual)
AP      = Active Products (bitfield: "D8"=VMware, "D14"=Hyper-V)
```

### 🟢 STATUS DA ÚLTIMA SESSÃO (5 campos)
```
D09F00  = Session Status Code (1=Running, 2=Failed, 5=OK, 8=WithErrors, 10=Quota, 11=NotSelected, etc)
D09F15  = Last Session Time (timestamp de QUALQUER tentativa)
D09F09  = Last Successful Session (timestamp do último backup bem-sucedido)
D09F07  = Last Failure Time (timestamp da última falha)
D09F08  = Color Bar (string 14 chars: "555554233555", histórico visual)
```

### 📊 ERROS E CONTAGEM (9 campos)
```
D09F06  = Error Count (erros na última sessão)
D09F19  = General Error Message (mensagem geral)
TK      = Total - Session Error Details
SK      = System State - Session Error Details
FK      = Files & Folders - Session Error Details
HK      = Hyper-V - Session Error Details
WK      = VMware - Session Error Details
GK      = M365 Exchange - Session Error Details
JK      = M365 OneDrive - Session Error Details
ZK      = MS SQL - Session Error Details
```

### 💾 TAMANHO (9 campos)
```
I14     = Used Storage (bytes utilizados no armazenamento)
D09F03  = Selected Size - Total (bytes selecionados para backup)
D01F03  = Selected Size - Files & Folders
D02F03  = Selected Size - System State
D08F03  = Selected Size - VMware
D14F03  = Selected Size - Hyper-V
D19F03  = Selected Size - M365 Exchange
D20F03  = Selected Size - M365 OneDrive
```

### 📧 M365 ESPECÍFICO (2 campos)
```
D19F20  = M365 Exchange - User Mailboxes Count
D19F21  = M365 Exchange - Shared Mailboxes Count
```

### ⚙️ CONFIGURAÇÃO (2 campos)
```
OP      = Operating Profile (nome do perfil/schedule de backup)
```

---

## ✅ CAMPOS ATUALMENTE CAPTURADOS (no backup_status_cache)

```
✅ AN → workload_name
✅ OT → workload_type
✅ I81 → physicality
✅ AR → cove_customer_ref
✅ I0 → cove_workload_id
✅ D09F09 → last_success
✅ D09F07 → last_failure
✅ D09F08 → color_bar_14d
✅ D09F06 → last_session_error_count
✅ D09F19/TK/SK/FK/GK/JK/ZK → last_error_message (genérico, não granular)
✅ I14 → used_storage_bytes
✅ D09F03 → selected_size_bytes
✅ D19F20 → m365_mailbox_count
✅ D19F21 → m365_user_mailboxes / m365_shared_mailboxes
✅ OP → profile
✅ [Portal type] → source ("Cove Device" / "Cove GB")
```

---

## ❌ CAMPOS DISPONÍVEIS MAS NÃO CAPTURADOS

```
❌ D09F00    = Status Code numérico (1-12) — CRÍTICO
❌ D09F15    = Last Session Time (ANY) — para detectar hanging
❌ MN        = Machine Name alternativo
❌ PN        = Partner Name (antes de normalizar)
❌ AP        = Active Products bitfield — para detectar VM automático

❌ SK        = System State error (individual)
❌ FK        = Files error (individual)
❌ HK        = Hyper-V error (individual)
❌ WK        = VMware error (individual)
❌ GK        = M365 Exchange error (individual)
❌ JK        = M365 OneDrive error (individual)
❌ ZK        = MS SQL error (individual)
❌ TK        = Total error (individual)
    → Atualmente: pega apenas PRIMEIRA mensagem de erro, não discrimina por source

❌ D01F03    = Files & Folders size individual
❌ D02F03    = System State size individual
❌ D08F03    = VMware size individual
❌ D14F03    = Hyper-V size individual
❌ D19F03    = M365 Exchange size individual
❌ D20F03    = M365 OneDrive size individual
    → Atualmente: pega primeiro valor não-nulo, não o breakdown
```

---

## 🎯 CAMPOS POR IMPACTO (se implementar)

### 🔴 CRÍTICO (Sprint 1)
```
D09F00  → Status code: "Over quota" (10), "Not selected" (11), etc
D09F15  → Detectar hanging sessions (rodando >12h)
SK/FK/GK/JK/ZK → Qual data source específico falhou
```

### 🟡 IMPORTANTE (Sprint 2)
```
D01F03...D20F03 → Storage breakdown (qual source consome mais)
PN → Partner name original (auditoria)
```

### 🟢 NICE-TO-HAVE (Backlog)
```
MN        → Nome alternativo do dispositivo
AP        → Auto-detectar VM sem depender de I81
Session history → requer permissão elevada na API Cove
```

---

## 📈 COMPARAÇÃO RÁPIDA

| Campo | Nome | Atual | Proposição |
|-------|------|-------|-----------|
| **D09F00** | Status Code | ❌ | ✅ Sprint 1 |
| **D09F15** | Last Session Time | ❌ | ✅ Sprint 1 |
| **D09F09** | Last Success | ✅ | — |
| **D09F07** | Last Failure | ✅ | — |
| **D09F08** | Color Bar | ✅ | — |
| **SK/FK/GK/JK/ZK** | Error by Source | ⚠️ (genérico) | ✅ Sprint 1 |
| **D01F03...D20F03** | Size by Source | ❌ | ✅ Sprint 2 |
| **I14** | Used Storage | ✅ | — |
| **D09F03** | Selected Size | ✅ | — |
| **D19F20/21** | M365 Mailboxes | ✅ | — |
| **OT** | OS Type | ✅ | — |
| **I81** | Physicality | ✅ | — |
| **OP** | Profile | ✅ | — |
| **AR** | Account Ref | ✅ | — |
| **PN** | Partner Name | ❌ | Sprint 2 |
| **AP** | Active Products | ❌ | Sprint 2 |
| **MN** | Machine Name | ❌ | Backlog |

---

## 🚀 PROMPT PARA USAR

> **"A API Covebackup retorna 112+ campos por workload. Atualmente, o módulo captura ~20 campos (status, timestamps, nomes, tamanho, M365). Os campos NÃO usados mas disponíveis são: (1) Status code numérico (diferencia over-quota, not-selected, etc); (2) Last session time para detectar hanging; (3) Erros por data source (Files, Exchange, SQL separadamente); (4) Tamanho por data source. Comparar storage breakdown poderia revelar qual source consome mais; capturar status code permitiria alertas específicos sem esperar >48h. Qual campo te interessa mais implementar primeiro?"**

