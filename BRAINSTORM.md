# Brainstorm: Melhoria WhatsApp Backend + Frontend

> Status: pronto pra implementação · 2026-05-22

## Objetivo
Melhorar WhatsApp module (código limpo, performance, UX) SEM quebrar produção. Começar com **low-risk refactoring**, depois otimizações.

## Escopo

### ✅ Inclui — Etapa 1 (Low-Risk, HOJE)
- **Extraction constants**: `src/lib/whatsapp/constants.ts`
  - Throttle delays (500ms, 5000ms)
  - Event types strings
  - Error messages padrão
  - URL paths
- **Pure helpers**: `src/lib/whatsapp/utils.ts`
  - `calculateDuration(startTime)` — calcula minutos
  - `formatDuration(minutes)` — formata pra display
  - `compareConversations(prev, next)` — detecta mudanças
  - `buildConvQuerySelect()` — monta select string
- **Type improvements**: `src/lib/whatsapp/types.ts`
  - Replace `any` com tipos específicos
  - Improve WhatsAppMessage, WhatsAppCase interfaces
- **Refactor hooks**: usar constants + helpers
  - `useWhatsAppConversations.ts` — 1170 → ~800 linhas
  - `useWhatsAppMessages.ts` — 484 → ~350 linhas

### ❌ Não inclui (decidido fora de escopo)
- Realtime logic refactor (risco de dessincronização)
- Edge function redesign
- whatsappQueues.ts refactor (crítico em RLS)
- Component refactor (Etapa 3)
- Feature novas (Etapa 4)

## Dependências críticas (NÃO QUEBRAR)
| Função | Risco | Por quê |
|--------|-------|--------|
| `assignCase()` | CRÍTICO | Muda case_status → triage/waiting/in_progress |
| `closeCase()` | CRÍTICO | Escreve closed_at, closed_by |
| `transferCase()` | CRÍTICO | Muda queue_id, unassigns |
| `useWhatsAppConversations()` return | CRÍTICO | 12 componentes dependem |
| `whatsappQueues.ts` | CRÍTICO | RLS policies usam funções |
| Realtime subscriptions | CRÍTICO | Sync em tempo real |
| `supabase.functions.invoke('whatsapp-api')` | CRÍTICO | Envia msgs/mídia |

## Plano de implementação

### Fase 1a — Constants + Types (HOJE, parallelizável)
**Risco:** Nenhum. Só muda como código é organizado.

#### 1. Create `src/lib/whatsapp/constants.ts`
```typescript
// Realtime
export const THROTTLE_DELAY = 500;
export const POLLING_INTERVAL = 5000;
export const MAX_RECONNECT_ATTEMPTS = 10;
export const BASE_RECONNECT_DELAY = 1000;
export const STABILITY_TIMEOUT = 5000;

// Tab visibility
export const STALE_DATA_THRESHOLD_MS = 10000;

// Event types (from line 84)
export const EVENT_TYPES = {
  ASSIGN: 'operator_assign',
  UNASSIGN: 'operator_unassign',
  TRANSFER: 'operator_transfer',
  CLOSE: 'operator_close',
  CHANGE_QUEUE: 'operator_change_queue',
  ASSIGN_BY_ADMIN: 'operator_assign_by_admin',
  TRANSFER_DIRECT: 'operator_transfer_direct',
} as const;

// Messages
export const MESSAGES = {
  ASSIGNED: (name: string) => `${name} assumiu este atendimento`,
  RETURNED: (name: string) => `${name} devolveu para a fila`,
  TRANSFERRED: (name: string, from: string, to: string) => `${name} transferiu de ${from} para ${to}`,
  CLOSED: (name: string) => `${name} encerrou o atendimento`,
  TRANSFER_TO_OPERATOR: (from: string, to: string) => `${from} transferiu para ${to}`,
  ASSIGN_BY_ADMIN: (admin: string, op: string) => `${admin} atribuiu para ${op}`,
};

export const SYSTEM_MESSAGE_ACTION = {
  ASSIGN: 'assign',
  UNASSIGN: 'unassign',
  TRANSFER: 'transfer',
  CLOSE: 'close',
  TRANSFER_TO_OPERATOR: 'transfer_to_operator',
  ASSIGN_BY_ADMIN: 'assign_operator',
} as const;

// Toast messages
export const TOAST = {
  SUCCESS: {
    ASSIGNED: 'Atendimento atualizado',
    ASSIGNED_DESC: 'Atendimento assumido com sucesso',
    UNASSIGNED_DESC: 'Atendimento devolvido para a fila',
    TRANSFERRED: 'Atendimento transferido',
    TRANSFERRED_DESC: 'O atendimento foi transferido para outra fila',
    CLOSED: 'Atendimento encerrado',
    DELETED: 'Atendimento excluído',
    QUEUE_CHANGED: 'Fila alterada',
    QUEUE_CHANGED_DESC: 'A conversa foi movida para outra fila',
  },
  ERROR: {
    DEFAULT: 'Erro',
    ASSIGN: 'Não foi possível atribuir o atendimento',
    TRANSFER: 'Não foi possível transferir o atendimento',
    CLOSE: 'Não foi possível encerrar o atendimento',
    DELETE: 'Não foi possível excluir o atendimento',
    QUEUE: 'Não foi possível alterar a fila',
    CONV_START: 'Não foi possível iniciar a conversa',
  },
};
```

**Arquivos afetados:**
- `useWhatsAppConversations.ts:641-675` (assignCase toasts)
- `useWhatsAppConversations.ts:678-731` (closeCase, toasts, messages)
- `useWhatsAppConversations.ts:84-85` (event types)

#### 2. Create `src/lib/whatsapp/utils.ts`
```typescript
// Pure helpers (testáveis, sem side effects)

export function calculateDuration(startTime: string | null): number | null {
  if (!startTime) return null;
  const start = new Date(startTime).getTime();
  return Math.round((Date.now() - start) / 1000 / 60);
}

export function formatDuration(minutes: number | null): string {
  if (minutes === null || minutes < 1) return 'menos de 1 min';
  if (minutes < 60) return `${minutes} min`;
  const hours = Math.floor(minutes / 60);
  const mins = minutes % 60;
  return mins > 0 ? `${hours}h ${mins}min` : `${hours}h`;
}

// Check if conversation data changed (from line 210-223)
export function conversationHasChanges(
  prev: WhatsAppCase,
  next: WhatsAppCase,
): boolean {
  return (
    prev.id !== next.id ||
    prev.last_message_at !== next.last_message_at ||
    prev.last_message_preview !== next.last_message_preview ||
    prev.unread_count !== next.unread_count ||
    prev.case_status !== next.case_status ||
    prev.assigned_to !== next.assigned_to ||
    prev.queue_id !== next.queue_id ||
    prev.last_message_from_me !== next.last_message_from_me
  );
}

// Build Supabase select query (from line 165-170)
export const CONV_QUERY_SELECT = `
  *,
  contact:whatsapp_contacts(*, company:companies(id, name)),
  queue:whatsapp_queues(*)
`;
```

**Arquivos afetados:**
- `useWhatsAppConversations.ts:133-147` (calculateDuration, formatDuration)
- `useWhatsAppConversations.ts:210-223` (compareConversations lógica)
- `useWhatsAppConversations.ts:165-170` (select query)

#### 3. Improve `src/lib/whatsapp/types.ts`
Replace `any` com tipos específicos:

```typescript
// Add to existing types.ts or create dedicated file
export type EventType = 
  | 'operator_assign'
  | 'operator_unassign'
  | 'operator_transfer'
  | 'operator_close'
  | 'operator_change_queue'
  | 'operator_assign_by_admin'
  | 'operator_transfer_direct';

export interface WhatsAppMessage {
  id: string;
  conversation_id: string;
  waha_message_id: string;
  from_me: boolean;
  content: string | null;
  message_type: 'text' | 'image' | 'video' | 'audio' | 'document' | 'location' | 'contact' | 'system' | 'sticker' | string;
  media_url: string | null;
  media_mimetype: string | null;
  media_filename: string | null;
  status: 'pending' | 'sent' | 'delivered' | 'read' | 'failed' | string;
  timestamp: string;
  metadata: Record<string, any> | null;
  sent_by_user_id: string | null;
  quoted_message_id?: string | null;
  sender_name?: string;
}

export interface SystemMessagePayload {
  action: string;
  actor_id?: string;
  actor_name?: string;
  [key: string]: any;
}
```

**Arquivo:** Adicionar a `src/hooks/useWhatsAppConversations.ts` e/ou `src/lib/whatsapp/types.ts`

---

### Fase 1b — Refactor hooks (use constants + helpers)
**Risco:** Baixo. Mantém assinatura pública igual.

#### 4. Refactor `useWhatsAppConversations.ts`
- Replace magic strings com `THROTTLE_DELAY`, `EVENT_TYPES.ASSIGN`, etc
- Use `calculateDuration()`, `formatDuration()` em vez de copiar lógica
- Use `conversationHasChanges()` em vez de inline comparison
- Use `CONV_QUERY_SELECT` em vez de hardcoded select string
- Replace `{ ...msg, ...next }` patterns com helper pra evitar dupes

**Linhas a mexer:**
- L133-147 → import helpers
- L84-95 → use EVENT_TYPES const
- L321 → `THROTTLE_DELAY`
- L356, L415, L430 → `POLLING_INTERVAL`
- L641-675 → use TOAST const
- L690-718 → use MESSAGES, TOAST const
- L210-223 → use `conversationHasChanges()`
- L165-170, L246-250 → use `CONV_QUERY_SELECT`

**Tests que PRECISAM passar:**
- `assignCase()` ainda retorna void, muda state igual
- `closeCase()` ainda manda closed_at, closed_by pra DB
- `transferCase()` ainda muda queue_id, unassigns
- Realtime ainda dispara `throttledUpdateConversation`

#### 5. Refactor `useWhatsAppMessages.ts`
- Similar: replace magic strings, use helpers
- Linhas: L28, L37-59, L147-155

---

### Fase 2 — Deduplication (após Fase 1)
**Risco:** Médio. Precisa testes.

Muitas funções (assignCase, assignToOperator, transferToOperator) fazem:
1. Fetch conversation
2. Build update object
3. Update DB
4. Insert system message
5. Log operator action
6. Show toast

Pode extrair `_updateCaseAndLog()` helper. MAS só após Phase 1 rodar e ter testes cobrindo.

---

### Fase 3 — Performance (após testes em Prod)
- Batch profile fetches
- Memoize selectors
- Lazy-load conversations list
- Remove realtime quando tab hidden

---

### Fase 4 — UX Fixes (paralelo)
Fix modal issues found in `tmp/`:
- `whatsapp-contact-modal-issue.png`
- `whatsapp-spellcheck.png`

---

## Decisões
| Decisão | Escolha | Por quê |
|---------|---------|--------|
| Framework | Lovable/Cursor | Geração automática + human review |
| Fase 1a | Constants+Types | Zero risk, máx ROI no código limpo |
| Fase 1b | Refactor hooks | Usa Phase 1a helpers, não quebra assinatura |
| Tests | Antes de Fase 2 | Deduplication requer cobertura alta |
| Ordem | 1a → 1b → 2 → 3 → 4 | Low→Medium risk, validar cada uma |

## Métricas de sucesso
- ✅ Fase 1: `useWhatsAppConversations` < 900 linhas (era 1170)
- ✅ Fase 1: 0 funcionalidades quebradas (testes passam)
- ✅ Fase 1: 0 mudança na assinatura pública (12 consumidores continuam funcionando)
- ✅ Produção: sem erro novo em whatsapp_ai_logs

## Riscos / pontos abertos
- **Realtime dessincronização** — se mexer em `setupChannel()`, pode ficar fora de sync. MAS não estamos mexendo. ✅
- **Profile fetches** — estão inline em assignCase, closeCase, etc. Deixar pra Fase 2. ✅
- **whatsappQueues RLS** — NÃO mexer. Tá fora de escopo. ✅
- **Edge function contract** — assinatura de `whatsapp-api` invoke não muda. ✅

## Próximo passo
**Implementar Fase 1a com Lovable**: criar constants.ts + utils.ts + melhorar types.ts. Depois human review da lógica de refactor, depois Fase 1b.

---

# Plano Detalhado pra Lovable

## Instrução pra Lovable

**TODO**: Create refactored WhatsApp module files WITHOUT breaking production:

### Task 1 — Extract Constants
**File**: `src/lib/whatsapp/constants.ts`
**Source**: Extrai de `useWhatsAppConversations.ts` linhas 84, 133-147, 321, 356, 415, 430, 641-675, 690-718
**What**: Constants p/ delays, event types, messages, toasts
**Test**: Import em useWhatsAppConversations, verificar que const values == current hardcoded values
**Risk**: Nenhum, só reorganiza

### Task 2 — Extract Pure Helpers
**File**: `src/lib/whatsapp/utils.ts`
**Source**: Extrai de `useWhatsAppConversations.ts` linhas 133-147, 210-223, 165-170
**What**: calculateDuration, formatDuration, conversationHasChanges, CONV_QUERY_SELECT
**Test**: Unit tests p/ cada função (calcDuration edge cases, formatDuration pra 0/1/60/120 mins)
**Risk**: Nenhum, funções puras sem side effects

### Task 3 — Improve Types
**File**: `src/lib/whatsapp/types.ts`
**Source**: Extrai de `useWhatsAppConversations.ts` lines 6-70, `useWhatsAppMessages.ts` lines 6-22
**What**: Remove `any`, use union types (EventType, message_type, status, etc)
**Test**: TypeScript strict mode, verificar imports em 12 consumidores
**Risk**: Baixo se tipos são backwards-compatible (podem ser mais specific, não mais loose)

### Task 4 — Refactor useWhatsAppConversations.ts
**Action**: Replace:
- Magic numbers/strings → import constants
- Inline duration calcs → import helpers
- Inline comparisons → import conversationHasChanges
- Hardcoded select → import CONV_QUERY_SELECT
**Tests to validate**:
- assignCase still returns void ✓
- markAsRead still sets unread_count to 0 ✓
- closeCase still inserts system message ✓
- transferCase still unassigns ✓
- All 12 consumidores still import { useWhatsAppConversations } and work ✓
**Lines to touch**: 84, 133-147, 165-170, 210-223, 321, 356, 415, 430, 641-675, 690-718
**Risk**: Médio. But safe se signature pública não muda.

### Task 5 — Refactor useWhatsAppMessages.ts
**Action**: Similar ao Task 4, replacer constantes/helpers
**Lines**: 28, 37-59, 147-155
**Tests**: sendMessage still works, realtime still picks up new messages
**Risk**: Baixo, menos lógica que useWhatsAppConversations

---

## Validação Final (pra fazer DEPOIS do Lovable gerar)
1. Run `npm run typecheck` — sem erros novos
2. Run tests (se houver) — tudo passa
3. Verificar `git diff` — só constants/types/utils, sem mudança em logica crítica
4. Deploy em dev environment — sync WhatsApp ainda funciona?
5. Monitor `whatsapp_ai_logs` próximas 2h — erro novo?

---

## Timeline
- **Hoje (2026-05-22)**: Lovable gera Tasks 1-5, human review
- **Amanhã**: Deploy Tasks 1-3 em dev (sem risco)
- **Próximos 2 dias**: Deploy Tasks 4-5, validar produção
- **Semana que vem**: Se tudo OK, Fase 2 (dedup)

