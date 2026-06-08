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
- Banco: Supabase gerenciado pelo Lovable (projeto ref `vaklqjseqtoseqhaugpe`). **MCP `portal-asv-ro` está morto** (credencial `claude_external_ro` não resolve). Pra query no banco vivo: rodar SQL no editor do Lovable.

## Gotchas que já me morderam (sintoma → causa)
- **"Mudança não aparece no ar"** → quase sempre: (a) VPS não deployou, OU (b) **service worker PWA** (`/service-worker.js`, `/sw.js`) servindo app velho no browser. Testar em **aba anônima**; furar cache. NÃO é o git/Lovable.
- **Menu/dashboard cortado p/ gestor ASV** → era `useAccessProfile.ts` forçando `isFleetDriverProfile=true` p/ operador ASV (`…0001`). **CORRIGIDO** (só é driver quem tem flag `is_fleet_driver`).
- **Data/hora errada nas viagens** → `new Date(trip_date)` parseava UTC (−1 dia em BRT) + escrita usava fuso do device. **CORRIGIDO**: display usa `+'T00:00:00'`; escrita usa helper BRT (`America/Sao_Paulo`) em `useDriverTrip.ts`.
- **KM total/média inflados** → viagens "KM a conferir" (sem `km_arrival`) eram estimadas e somadas. **CORRIGIDO**: só conta com `km_arrival` real; resto conta 0 (fica no histórico).
- **Modal do motorista fecha ao clicar fora** → **CORRIGIDO**: `onInteractOutside`/`onEscapeKeyDown` preventDefault nos 3 dialogs (início/parada/fim).

## Bugs/features resolvidos (2026-06) — contexto rápido
- Encerrar viagem exige KM final (UI + schema + hook + CHECK constraint).
- Histórico mostra todas empresas acessíveis (`.in(accessibleCompanies)`, sem depender de toggle).
- "Detalhes" do dashboard abre a viagem certa (`?viagem=<id>` → FleetTripsPage lê param).
- Parada exige cliente do dropdown, sem "Outro" (`FleetClientCombobox allowOther={false}`).

## Em andamento / TODO
- **Início e Fim de viagem exigirem cliente do dropdown** (obrigatório, sem "Outro"; texto = só observação opcional). FALTA: `frota_viagens` não tem coluna de cliente → precisa **migration** (add `entity_id` + `arrival_entity_id`) antes do front. Combobox já tem prop `allowOther`.
