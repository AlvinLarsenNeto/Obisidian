# FLEET Trips — Mapa rápido (visibilidade /fleet/trips)

> TL;DR leitura barata. Repo: `ASV/source-git` (origin `gustavofey-asv/portalasv`, branch main).
> Atualizado contra commit de 2026-06-05. Anchors = file:line pra pular direto.

## Cadeia escrita → leitura → RLS

### 1. ESCRITA (motorista /m)
`src/hooks/useDriverTrip.ts:110` (insert `frota_viagens`)
- `company_id = companyIdOverride || motorista.company_id`  ← cai na empresa do motorista
- `trip_status = IN_PROGRESS`, `trip_date = hoje`
- Motoristas Deivid/Adriano têm `company_id = 0000…0001` (ASV TECNOLOGIA) → corridas nascem em `…0001`.

### 2. LEITURA (gestor /fleet/trips)
`src/pages/fleet/FleetTripsPage.tsx:84-97`
- `role` vem de `useAuth()`; `isAdminOrOwner = role==='admin'||'owner'` (L91)
- `accessibleCompanies` vem de `useCompany()` (derivado do RLS `get_accessible_companies`)
- **Branch chave (L94):**
  - SE `(viewAllCompanies || isAdminOrOwner) && accessibleCompanyIds.length>0` → `.in('company_id', accessibleCompanyIds)` ✅ vê todas acessíveis
  - SENÃO → `.eq('company_id', effectiveCompanyId)` ← **só a empresa ATIVA** (se ativa = cliente, corridas de `…0001` somem)
- `effectiveCompanyId` = empresa ativa (de `useEffectiveCompany`), NÃO a empresa home.

### 3. RLS (banco)
`get_accessible_companies(user)` — def: `supabase/migrations/20260202142338_*.sql`
Retorna: empresa própria + `user_company_access` + (se ASV operator) filhos + self + filhos de irmãs ASV.
- `get_user_company_id(u)` = **`profiles.company_id`** (NÃO active_company_id) — migr `20251216180039_*`
- `is_asv_operator(u)` = true se `profiles.company_id` tem parent NULL ou `…0000` (GRUPO ASV) — migr `20260106151801_*`

## Modelo de empresas
- `…0000` = GRUPO ASV (raiz/master)
- `…0001` = ASV TECNOLOGIA (parent `…0000`) ← motoristas internos gravam aqui; é "self" dos operadores ASV
- Clientes = empresas sob GRUPO ASV; gestor ASV troca "empresa ativa" entre clientes.

## ⚠️ Insight (contradiz relatório Lovable)
Função **JÁ inclui self**: `c.id = get_user_company_id(_user_id)` (2x na def). Logo, se `profiles.company_id = …0001` e `is_asv_operator=true`, `get_accessible_companies` **DEVE** retornar `…0001`.
→ Se na prática não retorna: **função LIVE ≠ repo** (migration não aplicada em prod) OU consulta do Lovable usou user errado.

## Pra gestor (Alvin) VER as corridas de `…0001`, TODAS precisam valer:
1. `get_accessible_companies(alvin)` contém `…0001` (deveria, por self+asv_operator)
2. E **role ∈ {admin,owner}** OU toggle **"Ver Todas"** ligado → senão UI filtra `.eq(effectiveCompanyId=cliente)` e esconde
3. accessibleCompanies populado no client

## Antes de mexer na função, checar (no banco LIVE):
- `select company_id, active_company_id from profiles where user_id = <alvin>`
- `select role from user_roles where user_id = <alvin>` (precisa admin/owner OU usar Ver Todas)
- `select get_accessible_companies('<alvin>')` ← contém `…0001`?
- `select pg_get_functiondef('get_accessible_companies'::regproc)` ← bate com repo? (testar live vs migration)

Guilherme = MOTORISTA (não gestor) — comportamento dele (só vê próprias corridas) está correto.
