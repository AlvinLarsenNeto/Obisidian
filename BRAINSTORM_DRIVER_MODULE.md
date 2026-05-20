# Brainstorm: Refatoração Módulo de Motorista (/m)

> Status: rascunho · gerado em 2026-05-20

## Problema

Diálogos de viagem (DriverStartTripDialog 673 linhas, DriverEndTripDialog 616 linhas) são monolitos com:
- Múltiplos states desacoplados (veiculoId, origin, fotos, odômetro, limpeza, inspeção, CNH)
- Lógica duplicada (upload de fotos, validações, queries de reserva)
- Responsabilidades misturadas (UI, validação, API, storage)
- Difícil testar, manter, reutilizar

Resultado: ~1289 linhas de código repetido + difícil de evoluir quando regras mudam (ex: novo campo de validação = alterar em 2 lugares).

## Scope — O que refatorar

**Inclui:**
- Decompor DriverStartTripDialog em componentes menores (seleção veículo, CNH, limpeza, odômetro, inspeção)
- Decompor DriverEndTripDialog (odômetro fim, destino, inspeção, agendamentos)
- Centralizar schemas Zod para validação de viagem (início/fim)
- Centralizar constantes de trip (status, tipos de inspeção, campos config)
- Extrair hook useDriverTrip para lógica compartilhada (mutations, queries)
- Extrair hook usePhotoUpload para reutilizar upload em ambos diálogos
- Centralizar types de Trip, TripPhoto, InspectionItem

**Não inclui (fora de escopo):**
- Refatoração de InspectionChecklistInline, CleaningChecklist, OdometerPhotoCapture, TripPhotoCapture (componentes filhos — já funcionam)
- Mudança de schema BD (frota_viagens, etc.)
- Integração com portal.asv.com.br (apenas refatoração interna)
- Novo fluxo de motorista (apenas reorganização de código existente)

## Decisões Arquiteturais

| Decisão | Escolha | Por quê |
|---|---|---|
| **Estratégia decomposição** | Baseada em responsabilidade (seleção, validação, fotos, checkout) | Cada componente faz UM coisas, mais testável |
| **Hooks vs. Context** | Hooks customizados (useDriverTrip, usePhotoUpload) | Menos prop-drilling que Context, mais simples que reducer |
| **Tipos centralizados** | src/types/trip.ts | Évita `any`, facilita type-safety |
| **Schemas Zod** | src/schemas/trip.ts | Uma fonte de verdade para validação |
| **Constantes** | src/constants/trip.ts | Sem hardcoded strings em componentes |
| **Ordem de execução** | 6 prompts sequenciais (um de cada vez para Lovable) | Evita overload, cada mudança isolada |

## Plano de Implementação

### Fase 1: Preparar infra (types, schemas, constantes)

**PROMPT A1:** Centralizar types de viagem
- Criar: src/types/trip.ts
- Exportar: Trip, TripPhoto, InspectionItem, CleaningChecklist
- Remover any types de DriverStartTripDialog e DriverEndTripDialog
- (Ref: DriverStartTripDialog:27-31 tipos inline, DriverEndTripDialog:28-34)

**PROMPT A2:** Centralizar schemas Zod
- Criar: src/schemas/trip.ts
- Schemas: tripStartSchema, tripEndSchema, inspectionItemSchema, cleaningChecklistSchema
- Usar tipos de A1
- (Ref: validações espalhadas em DriverStartTripDialog:194-221)

**PROMPT A3:** Centralizar constantes
- Criar: src/constants/trip.ts
- TRIP_STATUS, PHOTO_TYPES, INSPECTION_CATEGORIES, CONFIG_FLAGS
- (Ref: hardcoded "em_andamento", "lido_ia", "pendente_conferencia" em DriverStartTripDialog:241-243)

### Fase 2: Extrair lógica reutilizável (hooks)

**PROMPT B1:** Extrair hook usePhotoUpload
- Criar: src/hooks/usePhotoUpload.ts
- Lógica de uploadPhoto (já duplicada entre Start/End)
- Input: file, path | Output: url ou null
- (Ref: DriverStartTripDialog:43-50, DriverEndTripDialog:37-43)

**PROMPT B2:** Extrair hook useDriverTrip
- Criar: src/hooks/useDriverTrip.ts
- Encapsular: startTripMutation, endTripMutation, queries (vehicles, reservations, etc.)
- Input: motorista, veiculoId, tipo (start|end) | Output: mutation, isLoading, error
- (Ref: DriverStartTripDialog:223-392, DriverEndTripDialog mutations)

### Fase 3: Decompor DriverStartTripDialog

**PROMPT C1:** Extrair DriverVehicleSelector
- Novo: src/components/fleet/driver/steps/DriverVehicleSelector.tsx
- Props: motorista, onSelect, vehicles, errors
- Lógica: seleção veículo, check de reserva, bloqueios
- (Ref: DriverStartTripDialog:89-221 queries + validações de veículo/reserva)

**PROMPT C2:** Extrair DriverTripChecklist (consolidar limpeza + inspeção)
- Novo: src/components/fleet/driver/steps/DriverTripChecklist.tsx
- Props: cleaning, inspection, portalConfig, onChange
- Renderiza: CleaningChecklist + InspectionChecklistInline
- (Ref: DriverStartTripDialog:64-66, 349-357)

**PROMPT C3:** Refatorar DriverStartTripDialog (orquestração)
- Remover: states embutidos, lógica de queries, lógica de upload
- Novo fluxo: Usar DriverVehicleSelector → DriverCNHCheck → DriverOdometerCapture → DriverTripChecklist → submit
- Usar: usePhotoUpload, useDriverTrip, schemas/constantes
- (Ref: atual 673 linhas → ~200 linhas só orquestração)

### Fase 4: Decompor DriverEndTripDialog

**PROMPT D1:** Extrair DriverTripCompletion
- Novo: src/components/fleet/driver/steps/DriverTripCompletion.tsx
- Props: trip, portalConfig, pendingWash, pendingMaintenance, onChange
- Renderiza: OdometerPhotoCapture + InspectionChecklistInline + agendamentos pendentes
- (Ref: DriverEndTripDialog:52-86)

**PROMPT D2:** Refatorar DriverEndTripDialog (orquestração)
- Remover: states embutidos, lógica de queries, lógica de upload
- Novo fluxo: Carrega dados trip → DriverTripCompletion → validações → submit
- Usar: usePhotoUpload, useDriverTrip, schemas/constantes
- (Ref: atual 616 linhas → ~150 linhas só orquestração)

## Resultado Esperado

- **6 prompts** para Lovable (A1, A2, A3, B1, B2, C1, C2, C3, D1, D2)
- **Infraestrutura centralizada:**
  - src/types/trip.ts
  - src/schemas/trip.ts
  - src/constants/trip.ts
- **Lógica reutilizável:**
  - src/hooks/usePhotoUpload.ts
  - src/hooks/useDriverTrip.ts
- **Componentes decompostos:**
  - DriverStartTripDialog: 673 → ~250 linhas
  - DriverEndTripDialog: 616 → ~200 linhas
  - 4 novos step-components (~100 linhas cada)
- **Tokens economy:**
  - ~400-500 linhas de duplicação eliminadas
  - ~30-40% redução de código
  - Lógica de mutation/query reutilizada
- **Testabilidade:**
  - Cada step component isolado
  - Hooks testáveis sem Dialog
  - Schemas/constantes independentes

## Riscos / Pontos Abertos

- InspectionChecklistInline precisa ser testado em contexto de ambos Start/End (variações de estado)
- Photo upload pode falhar parcialmente (alguns arquivos sim, outros não) — tratamento de erro já existe?
- Reservas estão em query refetchInterval 30s — certeza que isso vai continuar funcionando?
- Config portalMotoristaConfig vem de query — será que precisa ser centralizado também?

## Próximo Passo

Usuário aprova escopo, decide se faz tudo 10 prompts ou fase por fase, então começa PROMPT A1 (types).
