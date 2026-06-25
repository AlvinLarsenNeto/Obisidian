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