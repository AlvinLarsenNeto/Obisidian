# ASV Portal — OPS quickref (ler PRIMEIRO, é barato)

> Operacional do portal frota. Repo clone: `ASV/source-git` (origin `gustavofey-asv/portalasv`, branch `main`).
> RLS/visibilidade de viagens: ver [[FLEET_TRIPS_RLS_MAP]].

## Workflow (mudou — não é mais via Lovable)
Eu (Claude) edito direto em `ASV/source-git` → `build` → `commit` → `git push origin main`.
Alvin deploya na VPS. Lovable ainda commita no mesmo repo às vezes (commits "Changes"/"Fast Visual Edit") → fazer `git fetch` + rebase antes de push.

- Build local p/ validar antes de push: `npx vite build` (precisa `npm install --legacy-peer-deps` 1x; **apagar node_modules depois** — polui o vault Obsidian).
- Deploy (Alvin roda na VPS `/opt/portalasv`):
  ```bash
  git fetch origin && git reset --hard origin/main && docker compose up -d --build
  ```
- Stack deploy: VPS própria, Docker (node build → nginx serve `dist/`). **Sem Vercel/Netlify, sem Cloudflare.** nginx responde direto.
- Banco: Supabase gerenciado pelo Lovable (projeto ref `vaklqjseqtoseqhaugpe`). MCP `portal-asv-ro` (SQL read-only) **voltou a funcionar** (2026-06-16, tool `mcp__portal-asv-ro__query`). Se cair de novo, rodar SQL no editor do Lovable.

## Desambiguação de repo (IMPORTANTE — não confundir)
O **WhatsApp do portal ASV** (fila Inteligência de Dados, grupos, menções, LID) fica em **`portalasv`** (este `source-git`) e usa provider **uazapi**. Arquitetura: tabela `whatsapp_contacts`, fn `whatsapp-api`, `whatsapp-group-participants`, `WhatsAppChatWindow.tsx`, hook `useGroupParticipants.ts`.
**NÃO é** `gustavofey-asv/whatscrmsac` (arquitetura diferente: `contacts`/`contact_phones`, `inbox/ChatPanel`, fn `whatsapp-send`), nem talkyo, nem alumetaf. Agente Lovable já confundiu dizendo "whatscrmsac" — falso.

## Gotcha clone desatualizado
`source-git` fica pra trás (Lovable commita direto no repo). Já peguei **56 commits atrás** de origin/main → parecia que arquivos "não existiam". **Sempre `git fetch origin` + comparar com `origin/main` antes de afirmar que algo não existe.** Sync: `git reset --hard origin/main`.

## Gotchas que já me morderam (sintoma → causa)
- **Msg nova não aparece na thread do WhatsApp (só na prévia lateral)** (2026-06-17) → **PostgREST corta SELECT em 1000 linhas por padrão**. `useWhatsAppMessages` fazia `.order('timestamp', asc).range(0,4999)` → em conversa com >1000 msgs trazia as 1000 **mais antigas** e descartava as novas. A prévia lateral atualiza porque vem de `whatsapp_conversations.last_message_preview` (tabela diferente), NÃO da query de mensagens. **CORRIGIDO**: `order desc + range(0,999) + .reverse()` (1000 mais recentes). Limitação: sem scroll-up p/ msgs antigas. Bônus: canal realtime da thread NÃO deve usar `filter:` server-side (`postgres_changes` filtrado descarta eventos) — assinar sem filtro + filtrar client-side + refetch on focus/visibility (a thread não tinha polling como a lista).
- **"Mudança não aparece no ar"** → quase sempre: (a) VPS não deployou, OU (b) **service worker PWA** (`/service-worker.js`, `/sw.js`) servindo app velho no browser. Testar em **aba anônima**; furar cache. NÃO é o git/Lovable.
- **Menu/dashboard cortado p/ gestor ASV** → era `useAccessProfile.ts` forçando `isFleetDriverProfile=true` p/ operador ASV (`…0001`). **CORRIGIDO** (só é driver quem tem flag `is_fleet_driver`).
- **Data/hora errada nas viagens** → `new Date(trip_date)` parseava UTC (−1 dia em BRT) + escrita usava fuso do device. **CORRIGIDO**: display usa `+'T00:00:00'`; escrita usa helper BRT (`America/Sao_Paulo`) em `useDriverTrip.ts`.
- **KM total/média inflados** → viagens "KM a conferir" (sem `km_arrival`) eram estimadas e somadas. **CORRIGIDO**: só conta com `km_arrival` real; resto conta 0 (fica no histórico).
- **Modal do motorista fecha ao clicar fora** → **CORRIGIDO**: `onInteractOutside`/`onEscapeKeyDown` preventDefault nos 3 dialogs (início/parada/fim).

## Bugs/features resolvidos (2026-06) — contexto rápido
- **Menção (@) em grupo WhatsApp** (2026-06-16) — 2 causas distintas, ambas resolvidas:
  1. **Nome/dropdown mostrava LID "+275444926517369"** → `whatsapp-group-participants/index.ts` (`normalizeParticipant`) ignorava o campo **`PhoneNumber`** (PascalCase) que a UaZapi `/group/info` manda com o fone real (`554...@s.whatsapp.net`). Fix: ler `PhoneNumber`/`LID`/`Lid`/`DisplayName`, `formatPhoneBR` endurecido (só +55 12/13 díg ou DDI curto válido), guard `isRealPhone` no `lidToPhone`, cache `v3:`→`v4:`. + guard no front (`WhatsAppChatWindow.tsx handleSend`): só envia menção se fone real + jid `@s.whatsapp.net`, senão deixa `@Nome` literal.
  2. **Enviar menção dava `UaZapi API error: 400 Invalid payload`** → `whatsapp-api/index.ts` (`sendTextViaUaZapi`) mandava `mentions` como **array de JIDs** + campo inexistente `mentioned`. OpenAPI oficial: **`mentions` = STRING de números crus separados por vírgula** (`"5511...,5511..."`, sem `@s.whatsapp.net`). Fix: `payload.mentions = mentions.map(j=>j.replace(/@.*$/,'').replace(/\D/g,'')).filter(Boolean).join(',')` e remover `mentioned`. Texto já contém `@<numero>` (casa com mentions).
  - **Gotcha UaZapi:** campos em **PascalCase** no `/group/info` (`PhoneNumber`/`JID`/`LID`/`DisplayName`); `/send/text` `mentions` é **string CSV de números**, não array nem JID. Spec: repo `jonesfernandess/uazapi-cli` → `uazapi-openapi.json`.
- Encerrar viagem exige KM final (UI + schema + hook + CHECK constraint).
- Histórico mostra todas empresas acessíveis (`.in(accessibleCompanies)`, sem depender de toggle).
- "Detalhes" do dashboard abre a viagem certa (`?viagem=<id>` → FleetTripsPage lê param).
- Parada exige cliente do dropdown, sem "Outro" (`FleetClientCombobox allowOther={false}`).

## Em andamento / TODO
- **Início e Fim de viagem exigirem cliente do dropdown** (obrigatório, sem "Outro"; texto = só observação opcional). FALTA: `frota_viagens` não tem coluna de cliente → precisa **migration** (add `entity_id` + `arrival_entity_id`) antes do front. Combobox já tem prop `allowOther`.
