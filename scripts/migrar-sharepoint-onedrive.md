---
title: Migração SharePoint → OneDrive (arquivo morto)
tags: [script, powershell, sharepoint, onedrive, pnp, migracao]
created: 2026-04-17
updated: 2026-04-17
status: active
type: permanent
---

# Migração SharePoint → OneDrive (Riffel)

Script PowerShell cross-tenant que copia pastas de uma biblioteca SharePoint para um OneDrive pessoal usado como "arquivo morto".

## Contexto

- **Origem:** `https://riffelmotopecas.sharepoint.com` → biblioteca `File Server` → pastas `MARKETING/2021` e `MARKETING/2022`
- **Destino:** `https://riffelmotopecas-my.sharepoint.com/personal/arquivo_morto_riffel_com_br` → `Documents/MARKETING/<ano>`
- **Arquivo local:** `C:\Claude\microsoft\migrar_sharepoint.ps1`
- **Logs:** `C:\Temp\SPMigration\migration_<timestamp>.log`
- **Dependência:** `registrar_acesso_pnp.ps1` (gera ClientId do app registrado no Entra ID e grava em `C:\Temp\SPMigration\pnp_clientid.txt`)

## Pré-requisitos

- PowerShell 5.1+
- Módulo `PnP.PowerShell` (script instala automático se ausente)
- Dois logins MFA interativos (um por conexão: origem + destino)
- ClientId válido de app registration (gerado via `registrar_acesso_pnp.ps1`)

## Execução

```powershell
cd C:\Claude\microsoft
.\migrar_sharepoint.ps1
```

Duração típica: ~45 min por rodada completa (origem + destino listados duas vezes: cópia + verificação final).

## Histórico de bugs

### 2026-04-17 — Falso-positivo de SKIP

**Sintoma:** Primeira execução reportou "Pulados (já existiam): 8442" mas a verificação final encontrou apenas 8 arquivos no destino (5 em 2021 + 3 em 2022). Log em `migration_20260417_083951.log`.

**Raiz:** verificação `Get-PnPFile -Url <dst> -ErrorAction SilentlyContinue` retornava objeto "falso" quando o arquivo não existia no destino — o script marcava SKIP em 8440+ arquivos que nunca foram copiados.

**Fix aplicado:** pré-listar o destino uma vez por pasta-raiz e montar um hashtable (`$dstIndex`) indexado por `ServerRelativeUrl` (lowercase) → tamanho. Comparação vira `ContainsKey` + igualdade de `Length`. Código novo em `migrar_sharepoint.ps1:218-230` e `migrar_sharepoint.ps1:260-268`.

**Trade-off:** listagem do destino antes da cópia custa ~3-4 min extra, mas evita perda silenciosa de dados e torna o re-run realmente idempotente.

### 2026-04-18 — Get-FilesRecursive quebrado em OneDrive pessoal

**Sintoma:** Segunda execução (9h de duração, log `migration_20260417_232330.log`) reportou "Copiados: 8436 / Falhas: 0" mas a VERIFICAÇÃO FINAL ainda dizia "Destino: 5 arquivos" e "DIVERGENCIA 3810 faltando". Visualmente no OneDrive as pastas estavam populadas (Jobs com 68 itens, Mês do Motociclista com 21 itens, etc — ver `imagens/errom1..3.png` capturadas pelo usuário).

**Raiz:** `Get-FilesRecursive` calculava o path site-relativo de subpastas assim:
```powershell
$subPath = $folder.ServerRelativeUrl.TrimStart("/")
```
Isso funciona no SharePoint raiz (`/File Server/...` → `File Server/...`) mas **quebra em OneDrive pessoal**, onde `ServerRelativeUrl` é `/personal/<user>/Documents/...`. O TrimStart gerava `personal/<user>/Documents/...`, que `Get-PnPFolderItem -FolderSiteRelativeUrl` não reconhece → retorna vazio silenciosamente → recursão não desce → só contam os arquivos da raiz.

**Consequência:** A cópia funcionou (arquivos foram realmente uploadados), mas tanto a **pré-listagem do destino** (anti-SKIP-falso-positivo) quanto a **verificação final** estavam cegas a tudo que não fosse raiz.

**Fix aplicado:**
1. Adicionado helper `Get-WebSrp` que obtém `Web.ServerRelativeUrl` via `Get-PnPWeb`, cacheado por conexão.
2. `Get-FilesRecursive` agora faz strip do prefixo do Web antes de recursar, funcionando em qualquer tipo de site.
3. VERIFICAÇÃO FINAL reescrita para comparar **por caminho relativo** (não só por nome) usando hashtable de `relPath → size`, reportando `missing` e `sizeMismatch` separadamente.

### Script de verificação independente

Criado `verificar_sharepoint.ps1` — roda só a verificação (sem cópia), leva ~15 min, exporta CSV com faltantes/divergentes. Usar **antes** de deletar a origem para garantir integridade.

Fluxo recomendado:
1. Rodar `migrar_sharepoint.ps1` (já idempotente — re-executa com segurança copia só o que falta).
2. Rodar `verificar_sharepoint.ps1` — confere tudo por caminho relativo + tamanho.
3. Se resultado for `MIGRACAO 100% VERIFICADA`, é seguro excluir a origem.

## Verificação pós-execução

O próprio script roda um bloco **VERIFICAÇÃO FINAL** que conta arquivos/tamanho em origem e destino e emite `STATUS: VERIFICADO OK` ou `DIVERGENCIA DETECTADA` com lista dos faltantes. Sempre conferir esse bloco no log antes de considerar a migração concluída.

## Código (snapshot 2026-04-17 pós-fix)

```powershell
#Requires -Version 5.1
<#
.SYNOPSIS
    Migração cross-tenant: SharePoint → OneDrive (arquivo morto)

    ORIGEM:  https://riffelmotopecas.sharepoint.com
             Biblioteca: "File Server"
             Pastas:     MARKETING/2021  e  MARKETING/2022

    DESTINO: https://riffelmotopecas-my.sharepoint.com/personal/arquivo_morto_riffel_com_br
             Biblioteca: Documents
             Pasta base: MARKETING

.NOTAS
    - Requer PnP.PowerShell  (instalado automaticamente se ausente)
    - Dois logins interativos com MFA serão solicitados
    - Arquivos já existentes no destino (mesmo nome + tamanho) são pulados
    - Log completo salvo em C:\Temp\SPMigration\
#>

Set-StrictMode -Version Latest
$ErrorActionPreference = "Continue"   # Continue para não abortar em erros de arquivo único

# ─────────────────────────────────────────────────────────────────────────────
# CONFIGURAÇÃO
# ─────────────────────────────────────────────────────────────────────────────
$SRC_URL     = "https://riffelmotopecas.sharepoint.com"
$SRC_LIB     = "File Server"
$SRC_FOLDERS = @("MARKETING/2021", "MARKETING/2022")

$DST_URL     = "https://riffelmotopecas-my.sharepoint.com/personal/arquivo_morto_riffel_com_br"
$DST_LIB     = "Documents"
$DST_BASE    = "MARKETING"           # pasta raiz dentro de Documents

$TEMP_DIR    = "C:\Temp\SPMigration"
$LOG_FILE    = "$TEMP_DIR\migration_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
$MAX_RETRIES = 3
$PNP_TIMEOUT = 600   # 10 minutos (default PnP = 100s, insuficiente para videos grandes)

# ─────────────────────────────────────────────────────────────────────────────
# SETUP
# ─────────────────────────────────────────────────────────────────────────────
if (-not (Test-Path $TEMP_DIR)) {
    New-Item -ItemType Directory -Path $TEMP_DIR -Force | Out-Null
}

function Write-Log {
    param([string]$Msg, [string]$Level = "INFO")
    $ts    = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $entry = "[$ts][$Level] $Msg"
    $color = switch ($Level) {
        "ERROR" { "Red"     }
        "WARN"  { "Yellow"  }
        "OK"    { "Green"   }
        default { "Cyan"    }
    }
    Write-Host $entry -ForegroundColor $color
    Add-Content -Path $LOG_FILE -Value $entry -Encoding UTF8
}

# ─────────────────────────────────────────────────────────────────────────────
# MÓDULO
# ─────────────────────────────────────────────────────────────────────────────
if (-not (Get-Module -ListAvailable -Name PnP.PowerShell)) {
    Write-Log "PnP.PowerShell não encontrado. Instalando..." "WARN"
    Install-Module PnP.PowerShell -Force -AllowClobber -Scope CurrentUser
    Write-Log "PnP.PowerShell instalado." "OK"
}
Import-Module PnP.PowerShell -DisableNameChecking -ErrorAction Stop

# ─────────────────────────────────────────────────────────────────────────────
# FUNÇÃO: listar arquivos recursivamente
# ─────────────────────────────────────────────────────────────────────────────
function Get-FilesRecursive {
    param(
        [string]$SiteRelPath,
        $Conn
    )

    $result = [System.Collections.Generic.List[PSObject]]::new()

    # Arquivos nesta pasta
    try {
        $files = Get-PnPFolderItem -FolderSiteRelativeUrl $SiteRelPath `
                                   -ItemType File -Connection $Conn -ErrorAction Stop
        foreach ($f in $files) { $result.Add($f) }
    }
    catch {
        Write-Log "Aviso: não foi possível listar arquivos em '$SiteRelPath': $_" "WARN"
    }

    # Subpastas → recursão
    try {
        $folders = Get-PnPFolderItem -FolderSiteRelativeUrl $SiteRelPath `
                                     -ItemType Folder -Connection $Conn -ErrorAction Stop
        foreach ($folder in $folders) {
            # ServerRelativeUrl começa com "/" – remove para virar site-relative
            $subPath = $folder.ServerRelativeUrl.TrimStart("/")
            $subItems = Get-FilesRecursive -SiteRelPath $subPath -Conn $Conn
            foreach ($item in $subItems) { $result.Add($item) }
        }
    }
    catch {
        Write-Log "Aviso: não foi possível listar subpastas em '$SiteRelPath': $_" "WARN"
    }

    return $result
}

# ─────────────────────────────────────────────────────────────────────────────
# FUNÇÃO: copiar arquivo com retry
# ─────────────────────────────────────────────────────────────────────────────
function Copy-SPFile {
    param(
        [string]$SrcServerRelUrl,   # ex: /File Server/MARKETING/2021/foto.jpg
        [string]$FileName,
        [string]$DstSiteRelFolder,  # ex: Documents/MARKETING/2021/subfolder
        $SrcConn,
        $DstConn
    )

    $localFile = Join-Path $TEMP_DIR $FileName

    for ($try = 1; $try -le $MAX_RETRIES; $try++) {
        try {
            # ── Download ──────────────────────────────────────────────────
            Get-PnPFile -Url $SrcServerRelUrl `
                        -Path $TEMP_DIR `
                        -Filename $FileName `
                        -AsFile -Force `
                        -Connection $SrcConn | Out-Null

            # ── Upload ────────────────────────────────────────────────────
            Add-PnPFile -Path $localFile `
                        -Folder $DstSiteRelFolder `
                        -Connection $DstConn | Out-Null

            # ── Limpar temp ───────────────────────────────────────────────
            if (Test-Path $localFile) { Remove-Item $localFile -Force -ErrorAction SilentlyContinue }

            return $true
        }
        catch {
            Write-Log "  [tentativa $try/$MAX_RETRIES] Erro em '$FileName': $_" "WARN"
            if (Test-Path $localFile) { Remove-Item $localFile -Force -ErrorAction SilentlyContinue }
            if ($try -lt $MAX_RETRIES) { Start-Sleep -Seconds (15 * $try) }
        }
    }

    Write-Log "  FALHOU após $MAX_RETRIES tentativas: $FileName" "ERROR"
    return $false
}

# ─────────────────────────────────────────────────────────────────────────────
# CONEXÕES
# ─────────────────────────────────────────────────────────────────────────────
Write-Log "============================================================="
Write-Log "  MIGRACAO SHAREPOINT -> ONDRIVE (arquivo morto)"
Write-Log "============================================================="

# Ler ClientId gerado pelo registrar_acesso_pnp.ps1
$clientIdFile = "C:\Temp\SPMigration\pnp_clientid.txt"
if (-not (Test-Path $clientIdFile)) {
    Write-Log "ClientId nao encontrado. Execute '.\registrar_acesso_pnp.ps1' primeiro." "ERROR"
    exit 1
}
$PNP_APP_ID = (Get-Content $clientIdFile -Raw -Encoding UTF8).Trim()
if ([string]::IsNullOrEmpty($PNP_APP_ID)) {
    Write-Log "ClientId em $clientIdFile esta vazio. Execute '.\registrar_acesso_pnp.ps1' novamente." "ERROR"
    exit 1
}
Write-Log "Usando ClientId: $PNP_APP_ID"

Write-Host "`n[LOGIN 1/2]  ORIGEM - faca login com a conta admin do SharePoint Riffel`n" -ForegroundColor Green
$srcConn = Connect-PnPOnline -Url $SRC_URL -Interactive -ClientId $PNP_APP_ID -ReturnConnection
$srcConn.HttpClient.Timeout = [TimeSpan]::FromSeconds($PNP_TIMEOUT)
if (-not $srcConn) { Write-Log "Falha ao conectar na ORIGEM. Abortando." "ERROR"; exit 1 }
Write-Log "Conectado a ORIGEM: $SRC_URL" "OK"

Write-Host "`n[LOGIN 2/2]  DESTINO - faca login com a conta do OneDrive arquivo_morto`n" -ForegroundColor Green
$dstConn = Connect-PnPOnline -Url $DST_URL -Interactive -ClientId $PNP_APP_ID -ReturnConnection
$dstConn.HttpClient.Timeout = [TimeSpan]::FromSeconds($PNP_TIMEOUT)
if (-not $dstConn) { Write-Log "Falha ao conectar no DESTINO. Abortando." "ERROR"; exit 1 }
Write-Log "Conectado ao DESTINO: $DST_URL" "OK"

# ─────────────────────────────────────────────────────────────────────────────
# CÓPIA
# ─────────────────────────────────────────────────────────────────────────────
$totalCopied  = 0
$totalSkipped = 0
$totalFailed  = 0
$globalStart  = Get-Date

foreach ($srcFolder in $SRC_FOLDERS) {

    $yearFolder  = Split-Path $srcFolder -Leaf          # "2021" ou "2022"
    $srcRelPath  = "$SRC_LIB/$srcFolder"                # "File Server/MARKETING/2021"
    $dstRelBase  = "$DST_LIB/$DST_BASE/$yearFolder"     # "Documents/MARKETING/2021"
    # Base do servidor-relativo da origem (para calcular caminhos relativos)
    $srcSrvBase  = "/$SRC_LIB/$srcFolder"               # "/File Server/MARKETING/2021"

    Write-Log ""
    Write-Log "--- Pasta: $srcRelPath ---"

    # Listar todos os arquivos recursivamente
    Write-Log "  Listando arquivos na origem (pode demorar)..."
    $allFiles = Get-FilesRecursive -SiteRelPath $srcRelPath -Conn $srcConn

    if (-not $allFiles -or $allFiles.Count -eq 0) {
        Write-Log "  Nenhum arquivo encontrado. Pulando." "WARN"
        continue
    }

    $totalInFolder = $allFiles.Count
    $sizeInFolder  = [math]::Round(($allFiles | Measure-Object -Property Length -Sum).Sum / 1GB, 2)
    Write-Log "  Encontrados $totalInFolder arquivos ($sizeInFolder GB)"

    # ── Pre-listar destino para comparacao em memoria (evita falso-positivo de SKIP) ──
    Write-Log "  Listando destino para comparacao (pode demorar)..."
    $dstIndex = @{}
    try {
        $dstExisting = Get-FilesRecursive -SiteRelPath $dstRelBase -Conn $dstConn
        foreach ($df in $dstExisting) {
            $dstIndex[$df.ServerRelativeUrl.ToLowerInvariant()] = [int64]$df.Length
        }
        Write-Log "  Destino ja possui $($dstIndex.Count) arquivos."
    }
    catch {
        Write-Log "  Destino vazio ou inacessivel (tratando como novo): $_" "WARN"
    }

    $idx = 0
    foreach ($file in $allFiles) {

        $idx++
        $fileName     = $file.Name
        $fileSrvUrl   = $file.ServerRelativeUrl        # ex: /File Server/MARKETING/2021/sub/arq.pdf
        $fileLength   = $file.Length

        # Caminho relativo dentro da pasta-base (ex: "sub/arq.pdf" ou "arq.pdf")
        $relPath      = $fileSrvUrl.Substring($srcSrvBase.Length).TrimStart("/")
        $relFolder    = Split-Path $relPath -Parent     # "sub" ou ""

        # Pasta de destino para este arquivo
        if ([string]::IsNullOrEmpty($relFolder)) {
            $dstFolder = $dstRelBase
        }
        else {
            $dstFolder = "$dstRelBase/$relFolder" -replace "\\", "/"
        }

        # URL servidor-relativo no destino
        $dstSitePersonal = "/personal/arquivo_morto_riffel_com_br"
        $dstFileSrvUrl   = "$dstSitePersonal/$dstFolder/$fileName" -replace "//", "/"

        Write-Progress -Activity "Copiando MARKETING/$yearFolder" `
                       -Status "[$idx/$totalInFolder] $relPath" `
                       -PercentComplete ([math]::Round(($idx / $totalInFolder) * 100))

        # ── Verificar se já existe (pular se mesmo tamanho) ──────────────
        # Usa indice em memoria (ver pre-listagem acima) em vez de Get-PnPFile,
        # que retornava falsos-positivos e marcava SKIP em arquivos inexistentes.
        $dstKey = $dstFileSrvUrl.ToLowerInvariant()
        if ($dstIndex.ContainsKey($dstKey) -and $dstIndex[$dstKey] -eq $fileLength) {
            Write-Log "  SKIP (já existe): $relPath"
            $totalSkipped++
            continue
        }

        # ── Garantir estrutura de pastas no destino ───────────────────────
        try {
            Resolve-PnPFolder -SiteRelativePath $dstFolder -Connection $dstConn | Out-Null
        }
        catch {
            Write-Log "  Erro ao criar pasta '$dstFolder': $_" "ERROR"
            $totalFailed++
            continue
        }

        # ── Copiar ────────────────────────────────────────────────────────
        Write-Log "  [$idx/$totalInFolder] Copiando: $relPath  ($([math]::Round($fileLength/1MB,1)) MB)"
        $ok = Copy-SPFile -SrcServerRelUrl $fileSrvUrl `
                          -FileName $fileName `
                          -DstSiteRelFolder $dstFolder `
                          -SrcConn $srcConn `
                          -DstConn $dstConn

        if ($ok) { $totalCopied++  }
        else     { $totalFailed++  }
    }

    Write-Progress -Activity "Copiando MARKETING/$yearFolder" -Completed
}

# ─────────────────────────────────────────────────────────────────────────────
# VERIFICAÇÃO FINAL
# ─────────────────────────────────────────────────────────────────────────────
Write-Log ""
Write-Log "============================================================="
Write-Log "  VERIFICAÇÃO FINAL"
Write-Log "============================================================="

$verifyOk = $true

foreach ($srcFolder in $SRC_FOLDERS) {

    $yearFolder = Split-Path $srcFolder -Leaf
    $srcRelPath = "$SRC_LIB/$srcFolder"
    $dstRelPath = "$DST_LIB/$DST_BASE/$yearFolder"

    Write-Log ""
    Write-Log "  Contando arquivos na origem  : $srcRelPath"
    $srcFiles   = Get-FilesRecursive -SiteRelPath $srcRelPath -Conn $srcConn
    $srcCount   = ($srcFiles | Measure-Object).Count
    $srcSizeRaw = ($srcFiles | Measure-Object -Property Length -Sum).Sum
    $srcSizeGB  = if ($srcSizeRaw) { [math]::Round($srcSizeRaw / 1GB, 3) } else { 0 }

    Write-Log "  Contando arquivos no destino : $dstRelPath"
    $dstFiles   = Get-FilesRecursive -SiteRelPath $dstRelPath -Conn $dstConn
    $dstCount   = ($dstFiles | Measure-Object).Count
    $dstSizeRaw = ($dstFiles | Measure-Object -Property Length -Sum).Sum
    $dstSizeGB  = if ($dstSizeRaw) { [math]::Round($dstSizeRaw / 1GB, 3) } else { 0 }

    Write-Log "  +------------------------------------------"
    Write-Log "  | Pasta : MARKETING/$yearFolder"
    Write-Log "  | Origem:  $srcCount arquivos | $srcSizeGB GB"
    Write-Log "  | Destino: $dstCount arquivos | $dstSizeGB GB"

    if ($srcCount -eq $dstCount -and $srcSizeGB -eq $dstSizeGB) {
        Write-Log "  | STATUS: VERIFICADO OK - contagem e tamanho conferem" "OK"
    }
    else {
        $verifyOk = $false
        Write-Log "  | STATUS: DIVERGENCIA DETECTADA" "ERROR"
        if ($srcCount -ne $dstCount) {
            Write-Log "  |   Arquivos faltando: $($srcCount - $dstCount)" "ERROR"
        }
        if ($srcSizeGB -ne $dstSizeGB) {
            $diffMB = [math]::Round(($srcSizeGB - $dstSizeGB) * 1024, 1)
            Write-Log "  |   Diferenca de tamanho: $diffMB MB" "ERROR"
        }
        # Listar arquivos faltantes
        $dstNames = $dstFiles | Select-Object -ExpandProperty Name
        $missing  = $srcFiles | Where-Object { $_.Name -notin $dstNames }
        if ($missing) {
            Write-Log "  |   Arquivos faltantes no destino:" "ERROR"
            foreach ($m in $missing) {
                Write-Log "  |     - $($m.ServerRelativeUrl)" "ERROR"
            }
        }
    }
    Write-Log "  +------------------------------------------"
}

# ─────────────────────────────────────────────────────────────────────────────
# RESUMO
# ─────────────────────────────────────────────────────────────────────────────
$elapsed = (Get-Date) - $globalStart

Write-Log ""
Write-Log "============================================================="
Write-Log "  RESUMO DA MIGRAÇÃO"
Write-Log "============================================================="
Write-Log "  Copiados com sucesso : $totalCopied"
Write-Log "  Pulados (já existiam): $totalSkipped"
Write-Log "  Com falha            : $totalFailed"
Write-Log "  Tempo total          : $([math]::Round($elapsed.TotalMinutes, 1)) minutos"
Write-Log "  Log salvo em         : $LOG_FILE"

if ($totalFailed -gt 0) {
    Write-Log "  ATENÇÃO: $totalFailed arquivo(s) falharam. Verifique o log e reexecute o script." "WARN"
    Write-Log "  O script pula automaticamente arquivos já copiados (idempotente)." "WARN"
}

if ($verifyOk -and $totalFailed -eq 0) {
    Write-Log "  RESULTADO FINAL: MIGRACAO CONCLUIDA COM SUCESSO" "OK"
}
else {
    Write-Log "  RESULTADO FINAL: MIGRACAO INCOMPLETA - verifique os erros acima" "ERROR"
}

# Desconectar
try { Disconnect-PnPOnline -Connection $srcConn } catch {}
try { Disconnect-PnPOnline -Connection $dstConn } catch {}
```
