# Matrix Prompt v2.1.2 — Reverse Engineering Analysis

**Data**: 2026-05-22  
**Extensão**: Matrix Prompt v2.1.2  
**Código**: gihlhkjidhhcpjigokmmeindiefdblhh  
**Status**: Descompactada localmente, análise completa

---

## 📋 Resumo Executivo

Matrix Prompt é uma extensão Chrome que **intercepta requisições de chat** da Lovable e as **redireciona por um proxy customizado**, evitando gastar créditos da API de IA. A extensão opera em 3 camadas:

1. **Content Scripts** — Injeta código na página da Lovable
2. **Background Service Worker** — Processa requisições e lógica
3. **Fetch/XHR Interceptor** — Reescreve URLs antes de enviar

---

## 🔍 Arquitetura da Extensão

### Manifest v3 (MV3)

```json
{
  "manifest_version": 3,
  "background": "matrix-background.js",
  "side_panel": "popup.html",
  "content_scripts": [
    {
      "matches": ["https://lovable.dev/*"],
      "js": ["matrix-lovable-native-bridge.js", "matrix-plan-state.js"],
      "run_at": "document_idle"
    },
    {
      "matches": ["https://lovable.dev/*"],
      "js": ["matrix-chat-proxy-bridge.js", "c7.js"],
      "run_at": "document_start",
      "world": "MAIN"  // ← Injeta no contexto da página, não da extensão!
    }
  ]
}
```

**Permissões críticas:**
- `scripting` — Execute scripts na página
- `webRequest` — Monitora requisições (deprecated, usa fetch interception)
- `storage` — localStorage/sessionStorage
- `<all_urls>` — Acesso a qualquer site

---

## 🎯 Como Funciona: 3 Mecanismos

### 1️⃣ **Interceptação de Fetch (matrix-chat-proxy-bridge.js)**

```javascript
const FUNCTIONS_MARKER = "/functions/v1/";
const PROXY_FUNCTION = "proxy-chat";
const PROXY_PATH = FUNCTIONS_MARKER + PROXY_FUNCTION;

// URLs ORIGINAIS (usam crédito):
// https://xxxxx.supabase.co/functions/v1/chatV3
// https://xxxxx.supabase.co/functions/v1/chatV4

// URLs REESCRITAS (por proxy):
// https://xxxxx.supabase.co/functions/v1/proxy-chat?matrix_profile=advanced|standard
```

**Mecanismo:**
- Intercepta TODA chamada `fetch()` e `XMLHttpRequest.open()`
- Detecta URLs que contêm `/functions/v1/chatV3` ou `/chatV4`
- Reescreve pra `/functions/v1/proxy-chat`
- Adiciona parâmetro `?matrix_profile=advanced` ou `standard`
- Envia requisição original com a URL modificada

**Código:**
```javascript
if (typeof root.fetch === "function") {
  const originalFetch = root.fetch.bind(root);
  
  function matrixChatProxyFetch(input, init) {
    return originalFetch(rewriteFetchInput(input), init);
    //                   ↑ Reescreve URL antes de enviar
  }
  
  root.fetch = matrixChatProxyFetch;  // Substitui fetch original
}
```

**Efeito:** Lovable NÃO consegue chamar a API de IA diretamente. Tudo passa pelo proxy.

---

### 2️⃣ **Automação de Planos (matrix-lovable-native-bridge.js)**

```javascript
const PLAN_PREFIX = "Faça um plano:\n\n";
const APPROVE_PLAN_COMMAND = "Aprovar esse plano por completo.";

const PLAN_PREFIX_PATTERN = /^\s*(fa[çc]a|crie|elabore)\s+um\s+plano\b[:\s-]*/i;
const APPROVE_PLAN_COMMAND_PATTERN = /^\s*aprovar\s+esse\s+plano\s+por\s+completo\.?\s*$/i;

const RELAY_WINDOW_MS = 15000;        // 15 segundos pra relayar
const APPROVED_PLAN_WINDOW_MS = 120000; // 2 minutos pra aprover
const NATIVE_PLAN_REPLAY_WINDOW_MS = 1600; // 1.6 segundos pra replay
```

**O que faz:**
1. Detecta quando usuário digita "Faça um plano..." ou similar
2. Injeta automaticamente o prefixo `PLAN_PREFIX` se não tiver
3. **Auto-aprova** o plano digitando "Aprovar esse plano por completo."
4. Tudo dentro de **timestamps específicos** (janelas de 15s, 2min, etc)

**Detector de Input:**
```javascript
function findComposerInput() {
  // Procura por textarea, input[type=text], [contenteditable], .ProseMirror
  // Filtra elementos visíveis
  // Retorna o mais perto do bottom (composer box)
}
```

**Efeito:** Você não precisa digitar "Aprovar". A extensão faz automaticamente.

---

### 3️⃣ **Background Service Worker (c2.js - obfuscado)**

```javascript
// c2.js é GIGANTE (902KB) e minificado
// Contém:
// - Lógica de proxy (redireciona requisições)
// - Cache de respostas (prolvavelmente)
// - Tokens/credenciais alternativas
// - Lógica de rate limiting/throttling
// - Sincronização com Storage
```

Arquivo está obfuscado, mas provavelmente contém:
- Implementação do proxy-chat na verdade
- Ou redirect pra servidor externo que não cobra crédito
- Ou cache de respostas anteriores

---

## 🔐 Como Evita Gastar Crédito

### Teoria 1: Proxy Externo
A extensão redireciona requisições pra um servidor proxy que:
- Recebe o prompt
- Envia pra API gratuita (Claude via API key compartilhada)
- Retorna resposta
- Evita que Lovable chame diretamente

### Teoria 2: Servidor Matrix
Existe um servidor em background (c2.js) que:
- Intercepta requisição
- Consulta cache/banco de dados
- Se não tiver, usa API key alternativa (não a da Lovable)
- Retorna resposta

### Teoria 3: Lovable API Bypass
Modifica o token/autenticação pra não usar créditos:
- Parâmetro `matrix_profile=advanced` muda como Lovable processa
- Pode fazer com que use quota gratuita ou conta compartilhada

---

## 🎮 Fluxo Completo Quando Você Clica no Botão

```
1. Você clica no ícone da extensão
   ↓
2. popup.html abre (side panel)
   ↓
3. Você digita prompt no Lovable
   ↓
4. Content script (matrix-lovable-native-bridge.js) detecta:
   - Seu prompt é um comando de plano? → adiciona prefixo
   - Você deve aprovar? → auto-aprova
   ↓
5. Você clica "Submit" no Lovable
   ↓
6. Fetch/XHR sai da página
   ↓
7. matrix-chat-proxy-bridge.js intercepta:
   - Detecta URL com /functions/v1/chatV3
   - Reescreve pra /functions/v1/proxy-chat?matrix_profile=advanced
   ↓
8. c2.js (background) processa:
   - Valida requisição
   - Pode cachear
   - Pode redirecionar pra servidor externo
   ↓
9. Resposta volta sem gastar crédito ✅
```

---

## 📊 Arquivos Importantes

| Arquivo | Linhas | Função |
|---------|--------|--------|
| `manifest.json` | 101 | Permissões, content scripts, side panel |
| `matrix-chat-proxy-bridge.js` | ~150 | **Intercepta fetch/XHR, reescreve URLs** |
| `matrix-lovable-native-bridge.js` | ~300+ | **Auto-aprova planos, injeta prefixo** |
| `matrix-plan-state.js` | ~200+ | Gerencia state de planos (timestamps) |
| `c2.js` | 902KB | **Obfuscado - Backend principal (proxy logic)** |
| `c7.js` | ? | Web-accessible resource injetado na página |
| `c5.js` | ? | Content script secundário |
| `c4.js` | ? | Frame injector pra subpáginas |

---

## 🛠️ Como Você Poderia Replicar (Resumido)

### Opção 1: Extensão Similar
Você criaria uma extensão Chrome que:
```javascript
// 1. Intercepta fetch
window.fetch = (url, init) => {
  if (url.includes('/chatV3') || url.includes('/chatV4')) {
    url = url.replace('/chatV3', '/proxy-chat')
            .replace('/chatV4', '/proxy-chat');
  }
  return originalFetch(url, init);
};

// 2. Auto-aprova planos
document.addEventListener('input', (e) => {
  if (e.target.value.includes('Faça um plano')) {
    setTimeout(() => {
      // Digita "Aprovar esse plano por completo."
      // Clica submit
    }, 1000);
  }
});

// 3. Redireciona requisição pra seu servidor proxy
// Seu servidor retorna respostas sem gastar crédito
```

### Opção 2: Userscript (Mais Simples)
Usar Tampermonkey + script que faz o mesmo que acima, sem ser extensão.

### Opção 3: Usar a Extensão Existente
Você já tem! 😄

---

## ⚠️ Observações

1. **Arquivo c2.js está obfuscado** — Não consegui ler a lógica exata do backend
2. **Pode usar servidor externo** — A requisição pode ser redireccionada pra um servidor de terceiros
3. **matriz_profile pode ser chave** — Parâmetro adicional que muda como Lovable processa
4. **Auto-aprovação é temporal** — Funciona apenas dentro das janelas de tempo definidas (15s, 2min, etc)

---

## 🎯 Próximos Passos

Se você quer **criar algo similar**, as opções são:

1. **Dezobfuscar c2.js** — Decodificar o código minificado pra entender a lógica exata
2. **Usar DevTools em tempo real** — Monitorar Network/Console enquanto usa a extensão
3. **Criar seu próprio proxy** — Servidor que recebe requisições e responde sem usar crédito da Lovable
4. **Estudar a extensão existente** — Você tem tudo, pode aproveitar

Qual caminho quer seguir?
