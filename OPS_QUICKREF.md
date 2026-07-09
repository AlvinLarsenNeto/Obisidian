# ASV Portal — OPS quickref (ler PRIMEIRO, é barato)

> Operacional portal frota. Repo clone: `ASV/source-git` (origin `gustavofey-asv/portalasv`, branch `main`).
> RLS/viagens: ver [[FLEET_TRIPS_RLS_MAP]].

## Workflow (mudou — não é mais via Lovable)
Claude edita direto em `ASV/source-git` → `build` → `commit` → `git push origin main`.
Alvin deploya VPS. Lovable ainda commita (commits "Changes"/"Fast Visual Edit") → `git fetch` + rebase antes push.

- Build local p/ validar antes push: `npx vite build` (precisa `npm install --legacy-peer-deps` 1x; **apagar node_modules depois** — polui Obsidian).
- Deploy (roda `/opt/portalasv`):
  ```bash
  git fetch origin && git reset --hard origin/main && docker compose up -d --build
  ```
- Stack: VPS própria, Docker (node build → nginx serve `dist/`). **Sem Vercel/Netlify/Cloudflare.** Nginx responde direto.
- Banco: Supabase Lovable (projeto `vaklqjseqtoseqhaugpe`). MCP `portal-asv-ro` (SQL read-only) **voltou funcionar** (2026-06-16, tool `mcp__portal-asv-ro__query`). Se cair, rodar SQL editor Lovable.

## Desambiguação de repo (IMPORTANTE — não confundir)
**WhatsApp portal ASV** (fila Inteligência de Dados, grupos, menções, LID) em **`portalasv`** (`source-git`), provider **uazapi**. Arquitetura: tabela `whatsapp_contacts`, fn `whatsapp-api`, `whatsapp-group-participants`, `WhatsAppChatWindow.tsx`, hook `useGroupParticipants.ts`.
**NÃO é** `gustavofey-asv/whatscrmsac` (arquitetura: `contacts`/`contact_phones`, `inbox/ChatPanel`, fn `whatsapp-send`), talkyo, alumetaf. Lovable já confundiu ("whatscrmsac") — falso.

## Gotcha clone desatualizado
`source-git` atrás (Lovable commita direto). Já peguei **56 commits atrás** origin/main → arquivos "não existiam". **Sempre `git fetch origin` + comparar `origin/main` antes afirmar algo não existe.** Sync: `git reset --hard origin/main`.

## Gotchas que já me morderam (sintoma → causa)
- **Msg nova não aparece thread WhatsApp (só prévia lateral)** (2026-06-17) → **PostgREST corta SELECT 1000 linhas default**. `useWhatsAppMessages`: `.order('timestamp', asc).range(0,4999)` → >1000 msgs traz 1000 **antigas**, descarta novas. Prévia lateral atualiza de `whatsapp_conversations.last_message_preview` (tabela diferente), NÃO query mensagens. **CORRIGIDO**: `order desc + range(0,999) + .reverse()` (1000 recentes). Limitação: sem scroll-up p/ msgs antigas. Bônus: canal realtime NÃO `filter:` server-side (`postgres_changes` descarta) — assinar sem filtro + filtrar client + refetch focus/visibility (sem polling vs lista).

- **"Mudança não aparece"** → (a) VPS não deployou, (b) **service worker PWA** (`/service-worker.js`, `/sw.js`) serve app velho. Teste **aba anônima**; furar cache. NÃO git/Lovable.

- **Menu/dashboard cortado gestor ASV** → era `useAccessProfile.ts` força `isFleetDriverProfile=true` operador ASV (`…0001`). **CORRIGIDO** (só driver = flag `is_fleet_driver`).

- **Data/hora errada viagens** → `new Date(trip_date)` parseava UTC (−1 dia BRT) + escrita fuso device. **CORRIGIDO**: display `+'T00:00:00'`; escrita helper BRT (`America/Sao_Paulo`) `useDriverTrip.ts`.

- **KM total/média inflados** → viagens "KM conferir" (sem `km_arrival`) estimadas e somadas. **CORRIGIDO**: só conta `km_arrival` real; resto 0 (histórico).

- **Modal motorista fecha clicar fora** → **CORRIGIDO**: `onInteractOutside`/`onEscapeKeyDown` preventDefault 3 dialogs (início/parada/fim).

## Bugs/features resolvidos — contexto rápido
- **Update UX** (2026-07-09) — trocado popup bloqueante por botão discreto. Antes: `UpdateAvailableDialog.tsx` (Dialog centralizado não-fechável, montado em `App.tsx`). Agora: `UpdateSystemButton.tsx` no `Header.tsx` ao lado do marcador `v{BUILD_LABEL}` — laranja `bg-orange-500` "Atualizar sistema", `return null` até detectar deploy. Mesma detecção (poll 60s `/version.json?t=` vs `__BUILD_TIME__`) + `handleUpdate` (unregister SW + `caches.delete` + reload `?_v=<ts>`). Sempre pega ÚLTIMA versão (bundle vem do nginx). `UpdateAvailableDialog.tsx` deletado.

- **Loop de piscar/recarregar no iOS/Safari** (2026-07-09) → `main.tsx` handler `onupdatefound` do SW fazia `window.location.reload()` **incondicional sem guard**. Com SW `skipWaiting()`+`clients.claim()` (`public/service-worker.js`), iOS re-detectava SW a cada load → reload → loop. SW é **pass-through sem cache** → reload nunca serviu p/ frescor. **CORRIGIDO**: removido auto-reload, só registra o SW; update de versão é manual via `UpdateSystemButton`. Mecanismo build-id (`main.tsx:34-68`) mantido (guard `sessionStorage`, grava id antes do reload = reload único).

- **Relatório executivo — card Monitoramento traz servidores de OUTRAS empresas** (2026-07-09) — 2 níveis. **Causa raiz**: `companies.zabbix_hostgroup_id` da empresa = **NULL** (ex.: STAR `b87b7b36-…d303`). Em `supabase/functions/monitoring/index.ts`, `getHosts(hostgroupName)`/`getProblems` faziam **fail-open**: sem hostgroup → `host.get` SEM `groupids` → Zabbix retorna TODOS os hosts visíveis ao token (MAINHARDT, ASV, LIOTTO, BRUSTEC…). Override do companyId (body) já funcionava. **Fix código (fail-closed)**: guard interno em `getHosts` (`!hostgroupName`→`[]`) + nome informado mas inexistente→`[]`; call-site guards nos endpoints `hosts`/`problems`/`summary`/`sync-availability` (hostgroupId null→vazio). **NOC** (`all-problems`/`current-problems`) mantido — passa `null` de propósito. **Fix dado**: setar `zabbix_hostgroup_id` = nome do **grupo folha** no Zabbix (STAR = grupo `STAR`, 4 hosts). ⚠️ **Deploy é edge function** (`supabase functions deploy monitoring`), NÃO VPS. Gotcha classificação `classifyHost` (`useReports.ts:1384`): regra `FW` roda ANTES de `SRV` → host `SRV-FW…` vira firewall (não conta em Servidores Físicos).

- **Índice de Saúde (health_score) — edição manual sobrescrita ao regenerar** (2026-07-09) → `health_score` em `editableSections` (`useReports.ts:748`) mas `hasUserContent` (767-776) não checava campos dele (`pillars`/`overallScore`/`breakdown`) → sempre `false` → `upsert` sobrescrevia. **CORRIGIDO (padrão flag `manuallyEdited`)**: `onSave` (`ReportViewPage.tsx:1238`) injeta `manuallyEdited:true`+`editedAt`; `hasUserContent` respeita `existing?.manuallyEdited===true` (1ª condição, freeze genérico p/ qualquer seção editável) → cai no `else{continue}`. Escape hatch: botão "Recalcular automaticamente" (`ReportHealthScore.tsx`) salva sem flag. `updateSection` grava `payload_json` inteiro (sem merge), sem migration (JSONB livre).

## Bugs/features resolvidos (2026-06) — contexto rápido
- **Menção (@) grupo WhatsApp** (2026-06-16) — 2 causas, ambas resolvidas:
  1. **Dropdown mostrava LID "+275444926517369"** → `whatsapp-group-participants/index.ts` (`normalizeParticipant`) ignorava **`PhoneNumber`** (PascalCase) de UaZapi `/group/info` (`554...@s.whatsapp.net`). Fix: ler `PhoneNumber`/`LID`/`Lid`/`DisplayName`, `formatPhoneBR` endurecido (+55 12/13 díg), guard `isRealPhone`, cache `v3:`→`v4:`. + guard front (`WhatsAppChatWindow.tsx handleSend`): envia menção se fone real + jid `@s.whatsapp.net`, senão `@Nome`.
  2. **Enviar menção dava `UaZapi API error: 400 Invalid payload`** → `whatsapp-api/index.ts` (`sendTextViaUaZapi`) mandava `mentions` **array JIDs** + campo `mentioned`. OpenAPI: **`mentions` = STRING números CSV** (`"5511...,5511..."`, sem `@s.whatsapp.net`). Fix: `payload.mentions = mentions.map(j=>j.replace(/@.*$/,'').replace(/\D/g,'')).filter(Boolean).join(',')`, remover `mentioned`. Texto contém `@<numero>` (casa mentions).
  - **Gotcha UaZapi:** `/group/info` **PascalCase** (`PhoneNumber`/`JID`/`LID`/`DisplayName`); `/send/text` `mentions` **string CSV**, não array/JID. Spec: repo `jonesfernandess/uazapi-cli` → `uazapi-openapi.json`.

- Encerrar viagem exige KM final (UI + schema + hook + CHECK constraint).
- Histórico mostra empresas acessíveis (`.in(accessibleCompanies)`, sem toggle).
- "Detalhes" dashboard abre viagem certa (`?viagem=<id>` → FleetTripsPage lê param).
- Parada exige cliente dropdown, sem "Outro" (`FleetClientCombobox allowOther={false}`).

## Em andamento / TODO
- **Início/Fim viagem exigir cliente dropdown** (obrigatório, sem "Outro"; texto = observação). FALTA: `frota_viagens` sem coluna cliente → **migration** (`entity_id` + `arrival_entity_id`) antes front. Combobox já tem prop `allowOther`.