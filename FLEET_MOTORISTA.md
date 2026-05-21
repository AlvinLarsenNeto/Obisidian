---
title: FLEET — Portal Motorista (/m)
type: reference
status: active
created: 2026-05-21
last_updated: 2026-05-21
---

# Portal Motorista — /m (Frota ASV)

**URL:** `portalasv.com.br/m` (ou PWA instalado)  
**Acesso:** `is_fleet_driver = true` em auth profile  
**Framework:** React + TypeScript + Supabase

---

## 📍 Componentes Principais

| Componente | Arquivo | Função |
|-----------|---------|--------|
| **DriverTripHub** | `src/components/fleet/driver/DriverTripHub.tsx` | Hub central: viagem ativa ou iniciar |
| **DriverStartTripDialog** | `src/components/fleet/driver/DriverStartTripDialog.tsx` | Iniciar viagem (KM, fotos, combustível, CNH, checklist) |
| **DriverStopDialog** | `src/components/fleet/driver/DriverStopDialog.tsx` | Registrar parada (cliente, KM, local, motivo) |
| **DriverEndTripDialog** | `src/components/fleet/driver/DriverEndTripDialog.tsx` | Encerrar viagem (KM final, fotos, destino) |
| **InspectionChecklistInline** | `src/components/fleet/driver/InspectionChecklistInline.tsx` | Checklist inspeção (foto + observação) |
| **FleetDriverShortcutPage** | `src/pages/fleet/FleetDriverShortcutPage.tsx` | Página rota `/m` |

---

## 🔴 BUGS CONHECIDOS (2026-05-21)

| Bug | Localização | Status | Fix |
|-----|-----------|--------|-----|
| Checklist não abre foto/obs ao desmarcar | `InspectionChecklistInline.tsx:159` | 🔴 CRÍTICO | Toggle state não ativando renderização |
| PWA não instala no celular | `index.html` | 🔴 CRÍTICO | Falta Service Worker (`public/service-worker.js`) |
| Combustível VARCHAR(20) overflow | `DriverStartTripDialog.tsx:354` | 🔴 FIXADO? | Trocar por ENUM: vazio/1/4/1/2/3/4/cheio |
| Motorista faltando availability check em `/m` | `DriverStartTripDialog.tsx` | 🟡 PARCIAL | Tem `veiculoStatus` mas falta `motoristaStatus` |

---

## 🔧 FLUXOS IMPLEMENTADOS

### ✅ Iniciar Viagem
1. **Motorista** (auto, de `profile`)
2. **Veículo** (seleção, com reserva check)
3. **KM inicial** (foto IA + manual)
4. **Fotos 4 cantos** (frente, trás, esq, dir)
5. **Combustível** (ENUM: vazio/1/4/1/2/3/4/cheio)
6. **Checklist inspeção** (toca item → foto + obs) — ⚠️ BUG
7. **CNH** (bloqueador se incompleta)
8. **Salva trip** → gera lavação/manutenção automática

### ✅ Registrar Parada
1. **Cliente** (obrigatório, combobox)
2. **KM parada** (foto)
3. **Local** (auto-fill GPS)
4. **Motivo** (obrigatório)
5. **Observações** (opcional)

### ✅ Encerrar Viagem (2 passos obrigatórios)
1. **Passo 1:** KM final (foto, > KM inicial)
2. **Passo 2:** Fotos 4 cantos (chegada)
3. **Destino** (opcional)
4. **Salva trip** → gera lavação/manutenção automática

### 🟡 Histórico de Viagens
- ✅ Listagem (últimas viagens)
- ❌ Detalhes (abre `FleetTripDetailsDialog` — TODO)

### 🟡 Viagem Ativa em Time-Real
- 🔴 TODO: Card topo mostrando tempo decorrido + KM rodado

---

## 📊 GERAÇÃO AUTOMÁTICA: Lavações + Manutenções

**Função:** `src/lib/fleet/createChecklistMaintenance.ts` (lines 99-250)

### Sujeira Interna → **LAVAÇÃO**
Itens: bancos, painel, vidros internos, lixo, odor
- Cria: `frota_lavacoes_agendamentos` (status: pendente)
- Tipo: completa
- Prioridade: alta

### Avarias/Segurança → **MANUTENÇÃO**
Itens: lataria, vidro externo, pneus, espelhos, segurança (estepe, triângulo, extintor, docs)
- Cria: `frota_manutencoes` (status: pendente)
- Gera: `frota_alertas` (tipo: avaria_nova, veiculo_divergente, km_divergente)

---

## 🗄️ SCHEMA (Tabelas Principais)

```
frota_viagens
├── id, company_id, veiculo_id, motorista_id
├── trip_date, trip_time, trip_status (em_andamento/concluida)
├── km_recorded, km_arrival, km_status (pendente/lido_ia/conferido)
├── origin, destination, origin_lat, origin_lng, origin_endereco
├── departure_photos[], arrival_photos[], vehicle_photos[]
├── checklist_data { limpeza_interna, limpeza_externa, interna, externa, seguranca, combustivel, inspection_start }
└── gps_status (sem_rastreio/parcial/completo)

frota_paradas
├── id, viagem_id, numero_parada
├── km_parada, motivo_parada, notas
├── latitude, longitude, endereco
├── entity_id (cliente), other_client_name
└── odometer_photo_url

frota_lavacoes_agendamentos
├── id, veiculo_id, data_prevista, status (pendente/concluida)
├── tipo_sugerido (completa), prioridade (alta)
└── observacoes

frota_manutencoes
├── id, veiculo_id, status (pendente/concluida/cancelada)
├── description, scheduled_date, maintenance_type
└── supplier, notes

frota_alertas
├── id, viagem_id, veiculo_id, motorista_id
├── tipo (avaria_nova/veiculo_divergente/km_divergente/foto_distante)
├── severidade (critical/warning)
├── titulo, descricao, payload
└── created_at
```

---

## 🎯 MANIFEST & PWA

**Arquivo:** `public/manifest.webmanifest`

```json
{
  "name": "Portal ASV — Frota",
  "short_name": "Frota ASV",
  "start_url": "/fleet",  // ❌ ERRADO: deveria ser "/m"
  "scope": "/",
  "display": "standalone",
  "theme_color": "#1e3a5f"
}
```

**Missing:** `public/service-worker.js` (causa PWA não instalar)

---

## 🔗 ROTAS (via React Router)

| Rota | Componente | Tipo |
|------|-----------|------|
| `/m` | `FleetDriverShortcutPage` | Driver-only |
| `/fleet` | `FleetDashboardPage` | Admin |
| `/fleet/trips` | `FleetTripsPage` | Admin (com detalhes via `FleetTripDetailsDialog`) |
| `/fleet/drivers` | `FleetDriversPage` | Admin |

---

## 📋 CHECKLIST INSPEÇÃO (Categorias)

**Interna (sujeira):**
- bancos, painel, vidros_internos, sem_lixo, sem_odor

**Externa (avaria/sujeira):**
- lataria, vidros_externos, pneus_calotas, placas_visiveis, espelhos

**Segurança (faltando):**
- estepe, triangulo, macaco_chave, extintor, documentos

Código: `src/lib/fleet/inspectionConfig.ts`

---

## 🚀 PRÓXIMAS TAREFAS (Prioridade)

### 🔴 CRÍTICO
1. Fix: Checklist inspeção não abre foto/obs
2. Fix: Adicionar Service Worker para PWA instalar
3. Fix: Corrigir start_url manifest (`/m` em vez de `/fleet`)
4. Fix: Motorista availability check antes de iniciar viagem

### 🟡 IMPORTANTE
1. Feature: Histórico de viagens + detalhes (reusar `FleetTripDetailsDialog`)
2. Feature: Card viagem ativa (tempo decorrido + KM rodado em tempo real)
3. Feature: Encerramento em passos obrigatórios (já está code, validar fluxo)
4. Feature: Auto-fill local com GPS ao fotografar KM na parada

### 🟢 NICE-TO-HAVE
1. Auto-open parada dialog se viagem ativa ao abrir PWA
2. Teste geração automática de lavações/manutenções (fluxo real)

---

## 📚 DOCUMENTAÇÃO ASSOCIADA

- [[projects/portal-asv]] — Visão geral stack
- [[FLEET_GUIA_USUARIO.md]] — Guia motorista (PDF)
- `src/lib/fleet/createChecklistMaintenance.ts` — Geração automática

---

**Última atualização:** 2026-05-21 (Claude)
