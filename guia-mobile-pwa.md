# Guia: PWA + Push + Offline em projetos Mitra

Esse guia cobre PWA, push notifications, escrita offline (queue + Background Sync no Android), leitura offline (cache stale-while-revalidate) e os bugs sutis que aparecem nesse caminho.

O alvo é **qualquer projeto** — CRM, força de vendas, fluxo de caixa, gestão de obras, helpdesk. Os exemplos são genéricos: troque `comentario`/`post` pelo seu domínio (`pedido`/`cliente`, `lancamento`/`conta`, etc.).

---

## 0. Antes de implementar — alinhamento obrigatório com o usuário

> **Importante para a IA externa que estiver seguindo este guia:** antes de começar qualquer implementação, **explique ao usuário, de forma resumida e em linguagem acessível, o que é PWA e quais features mobile podem ser implementadas**. O usuário pode não saber o que cada uma significa nem o que ganha com elas.

### Resumo de PWA para apresentar ao usuário

PWA (Progressive Web App) é uma técnica que faz o sistema web se comportar como um app nativo no celular. O usuário **instala o sistema na tela inicial** (sem passar por loja de aplicativos) e o app abre em **tela cheia**, com ícone próprio. A aplicação continua sendo a mesma — não há código nativo separado para Android/iOS — mas ganha capacidades extras de aplicativo.

### Features que podem ser implementadas

Apresente cada uma ao usuário e pergunte quais ele quer no sistema dele:

| Feature | O que entrega | Plataformas |
|--------|---------------|------------|
| **PWA básico (instalação)** | Sistema instalável na tela inicial, abre em tela cheia, com ícone próprio | Android e iOS |
| **Splash screen e ícones** | Tela de carregamento e identidade visual ao abrir o app | Android e iOS |
| **Push notifications** | Notificações que chegam mesmo com o app fechado | Android (total) / iOS (apenas se instalado na home, iOS 16.4+) |
| **Leitura offline (cache de dados)** | Usuário vê os dados que já carregou antes mesmo sem internet | Android e iOS |
| **Escrita offline (fila de ações)** | Usuário cria/edita registros sem internet, e eles são enviados quando voltar a conexão | Android e iOS |
| **Background Sync** | Envio automático da fila offline mesmo com o app fechado | Apenas Android (iOS não suporta) |
| **Painel de diagnóstico** | Tela interna mostrando se o app está pronto para uso offline (versão, caches, erros) | Android e iOS |

### O que perguntar ao usuário antes de implementar

Conduza uma conversa rápida alinhando ao **objetivo do sistema**:

1. **Qual o uso esperado em mobile?** (uso casual no navegador, ou uso intensivo como app instalado?)
2. **Os usuários precisam usar o sistema sem internet?** (campo, áreas com sinal ruim, etc.)
3. **Vocês querem enviar notificações push?** (lembretes, alertas, novidades, aprovações)
4. **As ações dos usuários precisam funcionar offline?** (criar pedido, lançamento, comentário sem internet)
5. **Quais dispositivos predominam?** (Android, iOS ou ambos — para alinhar expectativa de Background Sync)
6. **Tem alguma feature mobile específica que não está nesta lista?** (deixe espaço pro usuário pedir algo fora do padrão)

> **Não implemente todas as features automaticamente.** Faça apenas o que o usuário confirmar que quer. PWA básico costuma ser sempre desejado; as demais (push, offline read/write, background sync) dependem do caso de uso. Confirme com o usuário e só então comece a seguir as seções deste guia.

---

## 1. O que é PWA (resumão)

Web app que se comporta como nativo: o usuário "instala" via Add to Home Screen, abre em tela cheia, recebe push notifications mesmo com o app fechado, e funciona offline com os dados que já viu.

Os 3 pilares:
- **`manifest.json`** — nome, ícones, cores, `display: standalone`
- **Service Worker** (`sw.js`) — script em background que intercepta requests, cacheia, recebe pushs e dispara sync
- **HTTPS** — obrigatório (já vem pronto na plataforma)

Limitação central: **iOS é parcial.** Push funciona só se o usuário instalar o PWA na home (não funciona no Safari aberto). Background Sync **não funciona no iOS** — flush de fila offline só acontece quando o usuário reabre o app.

---

## 2. Pilares básicos da PWA

### 2.1 `frontend/vite.config.ts`

⚠️ **Trampa importante:** se você usar `base: './'`, o `index.html` final referencia assets como `./assets/index-X.js`. Isso quebra qualquer regex de service worker que assume paths absolutos. Você tem dois caminhos:

- **Opção A (recomendada):** manter `base: './'` e fazer o SW normalizar `./` → `/` (ver §3.2).
- **Opção B:** trocar pra `base: '/'`. Mais simples no SW, mas pode quebrar o app se o servidor não tiver rewrite no path raiz.

### 2.2 `frontend/public/manifest.json`

```json
{
  "id": "/",
  "name": "Nome Completo do App",
  "short_name": "Nome",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "background_color": "#F5F1EA",
  "theme_color": "#F5F1EA",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "maskable" }
  ]
}
```

### 2.3 Assets em `frontend/public/`

Obrigatórios: `icon-192.png`, `icon-512.png`, `apple-touch-icon.png` (180x180), `favicon-32.png`, `favicon.ico`.

### 2.4 `frontend/index.html` — ponto crítico do offline

Esse arquivo é o **ponto único** onde:
- Captura erros antes de qualquer módulo carregar (senão erros de import são invisíveis).
- Mostra splash síncrono, eliminando tela branca.
- Filtra erros benignos (`Failed to fetch` do tracking do SDK não pode quebrar a UI offline).
- Tem failsafe de 8s — se o React não montou, mostra a tela de erro com o último erro registrado.

```html
<!doctype html>
<html lang="pt-BR">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/png" sizes="32x32" href="/favicon-32.png" />
    <link rel="apple-touch-icon" href="/apple-touch-icon.png" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
    <meta name="theme-color" content="#F5F1EA" />
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-status-bar-style" content="default" />
    <link rel="manifest" href="/manifest.json" />
    <title>Nome do App</title>

    <script>
      // Captura erros antes de qualquer modulo. So mostra tela de erro
      // se o React nao montou. Erros benignos (heartbeat do SDK falhando
      // offline) sao silenciados — sem isso o app fica preso em "Algo travou".
      (function () {
        function appMounted() {
          var root = document.getElementById('root');
          return !!(root && root.childElementCount > 0);
        }
        function isBenignNetwork(msg) {
          if (!msg) return false;
          var s = String(msg).toLowerCase();
          return s.indexOf('failed to fetch') !== -1
              || s.indexOf('networkerror') !== -1
              || s.indexOf('load failed') !== -1
              || s.indexOf('the network connection was lost') !== -1
              || s.indexOf('network request failed') !== -1;
        }
        function save(payload) {
          try { localStorage.setItem('app-last-error', JSON.stringify(payload)); } catch (e) {}
        }
        function showFatal(payload) {
          try {
            var box = document.getElementById('boot-error');
            if (box && box.style.display !== 'block') {
              box.style.display = 'block';
              var msg = document.getElementById('boot-error-msg');
              if (msg) msg.textContent = (payload && payload.message) || 'erro desconhecido';
            }
          } catch (e) {}
        }
        window.addEventListener('error', function (e) {
          var p = { message: e.message, source: e.filename, line: e.lineno, ts: new Date().toISOString() };
          save(p);
          if (!appMounted()) showFatal(p);
        });
        window.addEventListener('unhandledrejection', function (e) {
          var r = e.reason || {};
          var msg = r.message || String(r);
          if (isBenignNetwork(msg)) { try { e.preventDefault(); } catch (_) {} return; }
          var p = { message: 'unhandledrejection: ' + msg, stack: r.stack, ts: new Date().toISOString() };
          save(p);
          if (!appMounted()) showFatal(p);
        });
      })();
    </script>

    <style>
      #boot-splash {
        position: fixed; inset: 0; display: flex; flex-direction: column;
        align-items: center; justify-content: center; background: #F5F1EA;
        color: #3A332C; font-family: system-ui, sans-serif; gap: 12px; z-index: 9999;
      }
      #boot-splash img { width: 56px; height: 56px; border-radius: 12px; }
      #boot-error { display: none; position: fixed; inset: 0; padding: 24px; background: #fff; overflow: auto; z-index: 10000; font-family: system-ui, sans-serif; }
      #root:not(:empty) ~ #boot-splash { display: none; }
    </style>
  </head>
  <body>
    <div id="root"></div>

    <div id="boot-splash">
      <img src="/icon-192.png" alt="" />
      <div style="font-size:13px;color:#6B6258">Carregando...</div>
    </div>

    <div id="boot-error">
      <h2 style="font-size:16px;margin:0 0 8px">Algo travou ao abrir o app</h2>
      <p id="boot-error-msg" style="font-size:13px;color:#64748b;margin:0 0 12px;word-break:break-word"></p>
      <p style="font-size:13px;color:#64748b">Se voce esta offline, conecte-se uma vez para o app baixar tudo.</p>
      <button onclick="window.location.reload()">Recarregar</button>
      <button onclick="(async()=>{try{if('serviceWorker' in navigator){const rs=await navigator.serviceWorker.getRegistrations();for(const r of rs)await r.unregister();}if(window.caches){const ks=await caches.keys();for(const k of ks)await caches.delete(k);}localStorage.clear();}catch(e){}window.location.reload();})()">Limpar cache e recarregar</button>
    </div>

    <script type="module" src="/src/main.tsx"></script>
    <script>
      if ('serviceWorker' in navigator) {
        navigator.serviceWorker.register('/sw.js');
      }
      // Failsafe: 8s sem mount = mostra erro com o ultimo registrado
      setTimeout(function () {
        var root = document.getElementById('root');
        if (root && root.childElementCount === 0) {
          var box = document.getElementById('boot-error');
          if (box && box.style.display !== 'block') {
            box.style.display = 'block';
            var msg = document.getElementById('boot-error-msg');
            try {
              var raw = localStorage.getItem('app-last-error');
              if (msg) msg.textContent = raw || 'O app nao montou em 8s.';
            } catch (e) {}
          }
        }
      }, 8000);
    </script>
  </body>
</html>
```

### 2.5 `src/main.tsx` — registrar SW + ErrorBoundary

```tsx
import { createRoot } from 'react-dom/client'
import { HashRouter } from 'react-router-dom'
import './index.css'
import App from './App.tsx'
import ErrorBoundary from './components/ErrorBoundary.tsx'

createRoot(document.getElementById('root')!).render(
  <ErrorBoundary>
    <HashRouter>
      <App />
    </HashRouter>
  </ErrorBoundary>,
)
```

`ErrorBoundary` é uma classe que captura erros de runtime do React e mostra UI de fallback (e salva em `localStorage` pro diagnóstico).

> **Por que HashRouter e não BrowserRouter?** Porque PWA + path-based routing exige rewrite no servidor. Hash routing funciona em qualquer servidor estático e simplifica o SW (toda navegação resolve no mesmo `/index.html`).

---

## 3. Service Worker — `frontend/public/sw.js`

### 3.1 Estratégia geral

- **Shell cache** (`app-shell-vN`) — só `/index.html`. Servido em todas as navegações offline.
- **Asset cache** (`app-assets-vN`) — JS, CSS, ícones, fonts. Cache-first com populate na rede.
- **Versão** (`vN`) no nome do cache. Cada deploy bump → novo cache → caches velhos deletados no `activate`.

### 3.2 Install — precache crítico

⚠️ **A trampa do Vite `base: './'`:** o regex tem que normalizar `./` → `/`, senão **nada de assets** é precacheado e a PWA quebra offline na primeira visita.

```js
const SW_VERSION = 'v1';
const SHELL_CACHE = 'app-shell-' + SW_VERSION;
const ASSET_CACHE = 'app-assets-' + SW_VERSION;

self.addEventListener('install', (event) => {
  event.waitUntil((async () => {
    try {
      const indexResp = await fetch('/index.html', { cache: 'no-store' });
      const shellCache = await caches.open(SHELL_CACHE);
      // Cache em ambas as chaves: a que o SW serve E a que o browser pode pedir
      await shellCache.put('/index.html', indexResp.clone());
      await shellCache.put('/', indexResp.clone());

      const html = await indexResp.text();
      const urls = new Set();
      const re = /(?:src|href)="([^"]+)"/g;
      let m;
      while ((m = re.exec(html)) !== null) {
        let u = m[1];
        if (u.startsWith('./')) u = u.slice(1);          // normaliza Vite base relativo
        if (!u.startsWith('/')) continue;
        if (u.startsWith('http')) continue;
        if (
          u.startsWith('/assets/') || u.startsWith('/icon-') || u.startsWith('/favicon') ||
          u === '/manifest.json' || u === '/apple-touch-icon.png' ||
          /\.(js|mjs|css|png|jpg|jpeg|svg|webp|woff2?|ttf|json)$/i.test(u)
        ) urls.add(u);
      }

      const assetCache = await caches.open(ASSET_CACHE);
      await Promise.all([...urls].map(async (u) => {
        try {
          const r = await fetch(u, { cache: 'no-store' });
          if (r && r.ok) await assetCache.put(u, r.clone());
        } catch {}
      }));
    } catch (err) { console.warn('[sw] install', err); }
    self.skipWaiting();
  })());
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(names => Promise.all(
      names.filter(n => n !== SHELL_CACHE && n !== ASSET_CACHE).map(n => caches.delete(n))
    )).then(() => self.clients.claim())
  );
});
```

### 3.3 Fetch — navigation + assets

⚠️ **Outra trampa:** se `respondWith` retornar `undefined`, o browser mostra "Site can't be reached". Toda branch precisa retornar uma `Response` real.

```js
self.addEventListener('fetch', (event) => {
  const req = event.request;
  if (req.method !== 'GET') return;
  const url = new URL(req.url);

  // Navegação: network-first com fallback robusto
  if (req.mode === 'navigate') {
    event.respondWith((async () => {
      try {
        const resp = await fetch(req);
        const cache = await caches.open(SHELL_CACHE);
        cache.put('/index.html', resp.clone());
        cache.put('/', resp.clone());
        return resp;
      } catch {
        for (const key of ['/index.html', '/', req.url]) {
          const c = await caches.match(key); if (c) return c;
        }
        // Ultimo recurso — HTML inline em vez de tela branca
        return new Response(
          '<!DOCTYPE html><body style="font-family:system-ui;padding:40px;text-align:center"><h1>Sem conexao</h1><p>Conecte-se a internet uma vez para o app baixar tudo.</p><button onclick="location.reload()">Tentar de novo</button></body>',
          { status: 200, headers: { 'Content-Type': 'text/html' } }
        );
      }
    })());
    return;
  }

  // Assets same-origin: cache-first com populate; nunca retornar undefined
  if (url.origin === self.location.origin) {
    event.respondWith((async () => {
      const cached = await caches.match(req);
      if (cached) return cached;
      try {
        const resp = await fetch(req);
        if (resp && resp.status === 200 && resp.type === 'basic') {
          const cache = await caches.open(ASSET_CACHE);
          cache.put(req, resp.clone());
        }
        return resp;
      } catch {
        return new Response('Asset indisponivel offline: ' + url.pathname, { status: 504 });
      }
    })());
  }
});
```

### 3.4 Mensagens (diagnóstico)

```js
self.addEventListener('message', (event) => {
  if (event.data === 'SW_VERSION') {
    event.ports[0]?.postMessage({ version: SW_VERSION });
  }
  if (event.data === 'CACHE_INVENTORY') {
    (async () => {
      const result = {};
      for (const name of await caches.keys()) {
        const cache = await caches.open(name);
        result[name] = (await cache.keys()).map(r => new URL(r.url).pathname);
      }
      event.ports[0]?.postMessage(result);
    })();
  }
});
```

---

## 4. Push Notifications

### 4.1 Fluxo (visão geral)

```
[Usuario clica "Ativar"] -> Notification.requestPermission()
                        -> sw.pushManager.subscribe({ vapidPublicKey })
                        -> POST /interactions/executeServerFunction (criar/atualizar PUSH_SUBSCRIPTIONS)

[Server gatilho qualquer] -> SF JavaScript "enviarPush"
                          -> Lê PUSH_SUBSCRIPTIONS do(s) destinatario(s)
                          -> Para cada uma: web-push.sendNotification(sub, payload)

[Browser do destinatario] -> SW recebe 'push'
                          -> registration.showNotification(title, options)
                          -> User clica -> SW 'notificationclick'
                          -> abre URL no app (com prefixo /#/ pro HashRouter)
```

### 4.2 VAPID keys (gerar uma vez, guardar como variáveis do projeto)

Rode local:
```bash
npx web-push generate-vapid-keys
```
Saída:
```
Public Key:  BPv...
Private Key: hZ...
```

Salve no projeto via `setVariableMitra`:

```javascript
// backend/setup-backend.mjs (ou script avulso)
import 'dotenv/config';
import { configureSdkMitra, setVariableMitra } from 'mitra-sdk';

configureSdkMitra({ baseURL: process.env.MITRA_BASE_URL, token: process.env.MITRA_TOKEN, integrationURL: process.env.MITRA_BASE_URL_INTEGRATIONS });
const projectId = parseInt(process.env.MITRA_PROJECT_ID);

await setVariableMitra({ projectId, key: 'VAPID_PUBLIC_KEY', value: 'BPv...' });
await setVariableMitra({ projectId, key: 'VAPID_PRIVATE_KEY', value: 'hZ...' });
await setVariableMitra({ projectId, key: 'VAPID_SUBJECT', value: 'mailto:contato@empresa.com' });
```

⚠️ **Tem que ficar em variáveis do projeto** — não em arquivo. SF JavaScript não tem acesso ao filesystem; lê via `event.variables.VAPID_PRIVATE_KEY` (a engine injeta automaticamente).

### 4.3 Tabela `PUSH_SUBSCRIPTIONS`

```javascript
await runDdlMitra({
  projectId,
  sql: `CREATE TABLE PUSH_SUBSCRIPTIONS (
    ID INT AUTO_INCREMENT PRIMARY KEY,
    USUARIO_ID INT NOT NULL,
    ENDPOINT VARCHAR(500) NOT NULL,
    P256DH VARCHAR(200) NOT NULL,
    AUTH VARCHAR(200) NOT NULL,
    USER_AGENT VARCHAR(500),
    CRIADO_EM VARCHAR(19),
    ATUALIZADO_EM VARCHAR(19),
    UNIQUE KEY UQ_ENDPOINT (ENDPOINT(255))
  );`
});
```

Pra qualquer domínio: `USUARIO_ID` é o user que assinou. Um usuário pode ter várias subscriptions (um celular, um desktop, um tablet).

### 4.4 SFs de gerenciamento (SQL)

```javascript
// Upsert ao subscrever
await createServerFunctionMitra({
  projectId, name: 'salvarPushSubscription', type: 'SQL',
  code: `INSERT INTO PUSH_SUBSCRIPTIONS (USUARIO_ID, ENDPOINT, P256DH, AUTH, USER_AGENT, CRIADO_EM, ATUALIZADO_EM)
         VALUES (:VAR_USER, '{{endpoint}}', '{{p256dh}}', '{{auth}}', '{{userAgent}}', '{{agora}}', '{{agora}}')
         ON DUPLICATE KEY UPDATE
           USUARIO_ID = :VAR_USER,
           P256DH = '{{p256dh}}',
           AUTH = '{{auth}}',
           ATUALIZADO_EM = '{{agora}}';`,
});

// Remover ao usuário desabilitar
await createServerFunctionMitra({
  projectId, name: 'removerPushSubscription', type: 'SQL',
  code: `DELETE FROM PUSH_SUBSCRIPTIONS WHERE ENDPOINT = '{{endpoint}}';`,
});

// VAPID public key — frontend chama essa SF na primeira ativação.
// Em SF SQL nem todas as engines fazem placeholder de variavel; nesse caso
// crie uma SF JavaScript que retorne event.variables.VAPID_PUBLIC_KEY.
await createServerFunctionMitra({
  projectId, name: 'obterVapidPublic', type: 'JAVASCRIPT',
  code: `return { rows: [{ VAPID: event.variables.VAPID_PUBLIC_KEY || '' }] };`,
});
```

### 4.5 SF `enviarPush` (JavaScript)

Essa é a SF crítica. Recebe `userIds`, `title`, `body`, `url`, `tag`. Para cada usuário, busca todas as subscriptions e envia. Subscription expirada (410) é deletada.

```javascript
await createServerFunctionMitra({
  projectId,
  name: 'enviarPush',
  type: 'JAVASCRIPT',
  code: `
    const webpush = require('web-push');
    const { sdk } = require('mitra-sdk');

    const VAPID_PUBLIC = event.variables.VAPID_PUBLIC_KEY;
    const VAPID_PRIVATE = event.variables.VAPID_PRIVATE_KEY;
    const VAPID_SUBJECT = event.variables.VAPID_SUBJECT || 'mailto:noreply@empresa.com';
    webpush.setVapidDetails(VAPID_SUBJECT, VAPID_PUBLIC, VAPID_PRIVATE);

    const userIds = (event.userIds || []);
    if (userIds.length === 0) return { sent: 0 };

    const ids = userIds.join(',');
    const subsRes = await sdk.runDmlMitra({
      projectId: ${projectId},
      sql: 'SELECT ID, USUARIO_ID, ENDPOINT, P256DH, AUTH FROM PUSH_SUBSCRIPTIONS WHERE USUARIO_ID IN (' + ids + ')'
    });
    const subs = subsRes.rows || [];

    const payload = JSON.stringify({
      title: event.title || 'Notificacao',
      body: event.body || '',
      url: event.url || '/',
      tag: event.tag || 'general',
    });

    let sent = 0, failed = 0, removed = 0;
    for (const s of subs) {
      try {
        await webpush.sendNotification({
          endpoint: s.ENDPOINT,
          keys: { p256dh: s.P256DH, auth: s.AUTH },
        }, payload);
        sent++;
      } catch (err) {
        failed++;
        if (err.statusCode === 410 || err.statusCode === 404) {
          // Subscription expirada — remove
          await sdk.runDmlMitra({
            projectId: ${projectId},
            sql: 'DELETE FROM PUSH_SUBSCRIPTIONS WHERE ID = ' + s.ID
          });
          removed++;
        }
      }
    }
    return { sent, failed, removed, total: subs.length };
  `,
  description: 'Envia push notification para os usuarios em userIds. Remove subscriptions expiradas automaticamente.',
});
```

⚠️ **Pontos de atenção:**
- `event.variables.X` injeta variáveis do projeto.
- `web-push` está disponível na sandbox JavaScript do E2B — instalado on-demand.
- `userIds` chega como array (não string). Se chegar string concatenada, faça `String(event.userIds).split(',')`.
- **Nunca passe o body do usuário direto** no `body` do push — escape aspas/quebras de linha. Push payload é JSON.

### 4.6 Frontend — `src/lib/push.ts`

```ts
import { executeServerFunctionMitra } from 'mitra-interactions-sdk';
import { MITRA_CONFIG } from './constants';
import { SF } from './constants'; // registre os IDs de salvarPushSubscription, removerPushSubscription, obterVapidPublic

export function isPushSupported() {
  return 'serviceWorker' in navigator && 'PushManager' in window && 'Notification' in window;
}

export function isPushPermissionGranted() {
  return isPushSupported() && Notification.permission === 'granted';
}

function urlBase64ToUint8Array(base64: string) {
  const padding = '='.repeat((4 - (base64.length % 4)) % 4);
  const b64 = (base64 + padding).replace(/-/g, '+').replace(/_/g, '/');
  const raw = atob(b64);
  const arr = new Uint8Array(raw.length);
  for (let i = 0; i < raw.length; i++) arr[i] = raw.charCodeAt(i);
  return arr;
}

export async function subscribeToPush() {
  if (!isPushSupported()) throw new Error('push nao suportado');
  if (Notification.permission !== 'granted') {
    const perm = await Notification.requestPermission();
    if (perm !== 'granted') throw new Error('permissao negada');
  }
  const reg = await navigator.serviceWorker.ready;
  const vapidRes = await executeServerFunctionMitra({
    projectId: MITRA_CONFIG.projectId, serverFunctionId: SF.obterVapidPublic, input: {}
  });
  const vapid = vapidRes?.result?.output?.rows?.[0]?.VAPID;
  if (!vapid) throw new Error('vapid nao configurado');
  let sub = await reg.pushManager.getSubscription();
  if (!sub) {
    sub = await reg.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: urlBase64ToUint8Array(vapid),
    });
  }
  const json = sub.toJSON();
  await executeServerFunctionMitra({
    projectId: MITRA_CONFIG.projectId, serverFunctionId: SF.salvarPushSubscription,
    input: {
      endpoint: json.endpoint,
      p256dh: json.keys?.p256dh || '',
      auth: json.keys?.auth || '',
      userAgent: navigator.userAgent.slice(0, 500),
      agora: new Date().toISOString().slice(0, 19),
    },
  });
}

export async function unsubscribeFromPush() {
  const reg = await navigator.serviceWorker.ready;
  const sub = await reg.pushManager.getSubscription();
  if (!sub) return;
  await sub.unsubscribe();
  await executeServerFunctionMitra({
    projectId: MITRA_CONFIG.projectId, serverFunctionId: SF.removerPushSubscription,
    input: { endpoint: sub.endpoint },
  });
}
```

### 4.7 Botão de ativação (no Perfil)

```tsx
const [pushEnabled, setPushEnabled] = useState(isPushPermissionGranted());
const supported = isPushSupported();

if (!supported) return null;

return (
  <button onClick={async () => {
    if (pushEnabled) {
      await unsubscribeFromPush();
      setPushEnabled(false);
    } else {
      try { await subscribeToPush(); setPushEnabled(true); }
      catch { alert('Permissao negada ou push indisponivel'); }
    }
  }}>
    {pushEnabled ? 'Notificacoes ativadas' : 'Ativar notificacoes push'}
  </button>
);
```

### 4.8 SW — receber push e abrir o app

⚠️ **Trampa do HashRouter:** `notificationclick` precisa converter `/projeto/X` em `/#/projeto/X`, senão cai em rota 404 ou tela em branco.

```js
self.addEventListener('push', (event) => {
  if (!event.data) return;
  let data;
  try { data = event.data.json(); }
  catch { data = { title: 'Notificacao', body: event.data.text() }; }
  event.waitUntil(self.registration.showNotification(data.title || 'Notificacao', {
    body: data.body || '',
    icon: '/icon-192.png',
    badge: '/favicon-32.png',
    tag: data.tag || 'general',
    renotify: true,
    data: { url: data.url || '/' },
  }));
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  let raw = event.notification.data?.url || '/';
  let hashPath = '/';
  if (raw.startsWith('/#/')) hashPath = raw.substring(2);
  else if (raw.startsWith('/')) hashPath = raw;
  const fullUrl = self.location.origin + '/#' + hashPath;
  event.waitUntil(
    self.clients.matchAll({ type: 'window', includeUncontrolled: true }).then(list => {
      for (const c of list) {
        if (c.url.includes(self.location.origin) && 'focus' in c) {
          c.navigate(fullUrl); return c.focus();
        }
      }
      return self.clients.openWindow(fullUrl);
    })
  );
});
```

---

## 5. Offline READ — cache stale-while-revalidate

Para o usuário ver o que ele já viu antes mesmo offline. Cache em **IndexedDB** por `(serverFunctionId, input)`.

### 5.1 IndexedDB store

```ts
// src/lib/offline-queue.ts
const DB_NAME = 'app-offline';
const DB_VERSION = 3;
const STORE_SF_CACHE = 'sf_cache';

function openDB() {
  return new Promise<IDBDatabase>((resolve, reject) => {
    const req = indexedDB.open(DB_NAME, DB_VERSION);
    req.onupgradeneeded = () => {
      const db = req.result;
      if (!db.objectStoreNames.contains(STORE_SF_CACHE)) db.createObjectStore(STORE_SF_CACHE, { keyPath: 'key' });
      // ... outros stores (queue, auth — proximas secoes)
    };
    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
  });
}
```

### 5.2 `callSFCached` — wrapper que cacheia leitura

```ts
function cacheKey(sfId: number, input: Record<string, any>): string {
  const sorted: Record<string, any> = {};
  Object.keys(input).sort().forEach(k => { sorted[k] = input[k]; });
  return sfId + ':' + JSON.stringify(sorted);
}

export async function callSFCached(sfId: number, input: Record<string, any> = {}): Promise<any> {
  const key = cacheKey(sfId, input);
  const online = typeof navigator === 'undefined' || navigator.onLine !== false;

  if (online) {
    try {
      const result = await callSF(sfId, input);
      try {
        const db = await openDB();
        const t = db.transaction(STORE_SF_CACHE, 'readwrite');
        t.objectStore(STORE_SF_CACHE).put({ key, value: result, ts: Date.now() });
      } catch {}
      return result;
    } catch {
      return await readCache(key);
    }
  }
  return await readCache(key);
}

async function readCache(key: string): Promise<any> {
  try {
    const db = await openDB();
    return await new Promise((resolve, reject) => {
      const t = db.transaction(STORE_SF_CACHE, 'readonly');
      const req = t.objectStore(STORE_SF_CACHE).get(key);
      req.onsuccess = () => resolve(req.result?.value || null);
      req.onerror = () => reject(req.error);
    });
  } catch { return null; }
}
```

### 5.3 Quando usar `callSFCached` vs `callSF`

- **Use `callSFCached`** em: lista de itens da home, perfil do usuário, qualquer SELECT que serve a tela principal e que se beneficia de mostrar dados velhos quando offline.
- **Use `callSF`** em: mutações, contadores que mudam toda hora, dados sensíveis a frescor.

⚠️ **Trampa:** se o `profile` (USER_ID, nome) não tiver no cache, ações offline tipo "criar comentário" silenciosamente quebram porque o handler tem `if (!profile) return;`. **Sempre cacheie o profile.**

### 5.4 Banner offline

```tsx
const [isOffline, setIsOffline] = useState(navigator.onLine === false);
useEffect(() => {
  const on = () => setIsOffline(false);
  const off = () => setIsOffline(true);
  window.addEventListener('online', on);
  window.addEventListener('offline', off);
  return () => { window.removeEventListener('online', on); window.removeEventListener('offline', off); };
}, []);

{isOffline && <div className="banner-offline">Sem conexao — voce vera os ultimos dados carregados</div>}
```

---

## 6. Offline WRITE — fila de mutações

O usuário cria registros offline (comentário, pedido, lançamento), aparece na UI imediatamente como "pendente", e é enviado quando voltar a internet.

### 6.1 Estrutura do queue

```ts
const STORE_PENDING = 'pending_writes';

interface PendingWrite {
  localId: string;
  // payload generico — adapta pro seu domain (postId/comentario, clienteId/pedido, etc.)
  type: 'comentario' | 'pedido' | 'lancamento';
  payload: any;
  criadoEm: string;
  attempts: number;
}

export async function enqueueWrite(w: Omit<PendingWrite, 'localId' | 'attempts'>): Promise<PendingWrite> {
  const pending: PendingWrite = {
    ...w,
    localId: 'pw_' + Date.now().toString(36) + '_' + Math.random().toString(36).slice(2, 8),
    attempts: 0,
  };
  // ... salvar no IDB
  // Registrar Background Sync (Android Chrome)
  try {
    if ('serviceWorker' in navigator && 'SyncManager' in window) {
      const reg = await navigator.serviceWorker.ready;
      await (reg as any).sync.register('flush-writes');
    }
  } catch {}
  return pending;
}
```

### 6.2 UX: optimistic update + indicador visual

Sempre mostre o item **na hora** com um badge "Enviando quando voltar online". Se falhar, mostre vermelho. Se enviar, troque pelo real.

```tsx
const [pending, setPending] = useState<PendingWrite[]>([]);

async function handleSubmit() {
  const w = await enqueueWrite({ type: 'comentario', payload: {...}, criadoEm: now() });
  setPending(prev => [...prev, w]);
  toast.success('Salvo. Sera enviado quando voltar online.');
  flushPending().catch(() => {}); // tenta enviar agora se online
}

// Render:
{realItems.map(item => <RealCard {...item} />)}
{pending.map(p => <PendingCard {...p} />)}
```

### 6.3 `flushPending` — função do client

```ts
let isFlushing = false;
export async function flushPending() {
  if (isFlushing) return { sent: 0 };
  if (navigator.onLine === false) return { sent: 0 };
  isFlushing = true;
  let sent = 0;
  try {
    const all = await getAllPending();
    all.sort((a, b) => a.criadoEm.localeCompare(b.criadoEm));
    for (const p of all) {
      try {
        await sendOne(p);              // chama as SFs apropriadas pro tipo
        await removePending(p.localId);
        sent++;
      } catch {
        await incrementAttempts(p.localId);
        break; // para no primeiro fail pra manter ordem
      }
    }
  } finally { isFlushing = false; }
  if (sent > 0) window.dispatchEvent(new CustomEvent('app:offline-flush', { detail: { sent } }));
  return { sent };
}
```

### 6.4 Disparar flush em todos os momentos críticos

⚠️ **Aprendi na pele:** só ouvir `'online'` não basta. No Android Chrome, se o app tá em background, o evento pode não chegar. Adiciona TODOS:

```tsx
useEffect(() => {
  if (!profile) return;
  flushPending().catch(() => {});
  function tryFlush() {
    if (navigator.onLine === false) return;
    flushPending().catch(() => {});
  }
  function handleVisibility() { if (!document.hidden) tryFlush(); }
  window.addEventListener('online', tryFlush);
  window.addEventListener('focus', tryFlush);
  window.addEventListener('pageshow', tryFlush);
  document.addEventListener('visibilitychange', handleVisibility);
  return () => {
    window.removeEventListener('online', tryFlush);
    window.removeEventListener('focus', tryFlush);
    window.removeEventListener('pageshow', tryFlush);
    document.removeEventListener('visibilitychange', handleVisibility);
  };
}, [profile]);
```

---

## 7. Background Sync — flush automático no Android (mesmo com app fechado)

Aqui é onde a coisa fica boa **só pra Android**. iOS não suporta `'sync'` event. No iOS, o flush sempre depende do app reabrir (cobertura do §6.4).

### 7.1 Persistir auth no IDB

O SW precisa do token pra chamar a API. Token mora em `localStorage`, mas SW não tem acesso a `localStorage`. Solução: **espelhar no IndexedDB**.

```ts
const STORE_AUTH = 'auth';

// No upgradeneeded:
if (!db.objectStoreNames.contains(STORE_AUTH)) db.createObjectStore(STORE_AUTH, { keyPath: 'key' });

export async function saveAuthForSW(auth: { baseURL: string; token: string; projectId: number }) {
  const db = await openDB();
  const t = db.transaction(STORE_AUTH, 'readwrite');
  t.objectStore(STORE_AUTH).put({ key: 'current', ...auth });
}
```

E em `mitra-auth.ts`:

```ts
export function saveSession(session: MitraSession): void {
  localStorage.setItem(STORE_KEY, JSON.stringify(session));
  saveAuthForSW({ baseURL: session.baseURL, token: session.token, projectId: PROJECT_ID });
}

function configureSdk(session: MitraSession): void {
  saveAuthForSW({ baseURL: session.baseURL, token: session.token, projectId: PROJECT_ID });
  configureSdkMitra({
    ...session, projectId: PROJECT_ID, authUrl: AUTH_URL,
    onTokenRefresh: (newSession) => saveSession(newSession), // manter IDB sempre fresh
  });
}
```

### 7.2 SW — handler do `sync`

```js
const SYNC_TAG = 'flush-writes';
const IDB_NAME = 'app-offline';
const IDB_VERSION = 3;
const STORE_PENDING = 'pending_writes';
const STORE_AUTH = 'auth';

function idbOpen() {
  return new Promise((resolve, reject) => {
    const req = indexedDB.open(IDB_NAME, IDB_VERSION);
    req.onupgradeneeded = () => {
      // Idempotente — caso o SW abra antes do client
      const db = req.result;
      if (!db.objectStoreNames.contains(STORE_PENDING)) db.createObjectStore(STORE_PENDING, { keyPath: 'localId' });
      if (!db.objectStoreNames.contains('sf_cache')) db.createObjectStore('sf_cache', { keyPath: 'key' });
      if (!db.objectStoreNames.contains(STORE_AUTH)) db.createObjectStore(STORE_AUTH, { keyPath: 'key' });
    };
    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
  });
}

async function execSF(auth, serverFunctionId, input) {
  const url = auth.baseURL.replace(/\/+$/, '') + '/interactions/executeServerFunction';
  const headers = { 'Content-Type': 'application/json' };
  if (auth.token) headers['Authorization'] = auth.token.startsWith('Bearer ') ? auth.token : 'Bearer ' + auth.token;
  const r = await fetch(url, {
    method: 'POST', headers,
    body: JSON.stringify({ projectId: auth.projectId, serverFunctionId, input }),
  });
  if (!r.ok) throw new Error('SF ' + serverFunctionId + ' HTTP ' + r.status);
  const data = await r.json();
  let out = data?.result?.output;
  if (typeof out === 'string') { try { out = JSON.parse(out); } catch {} }
  return out;
}

self.addEventListener('sync', (event) => {
  if (event.tag !== SYNC_TAG) return;
  event.waitUntil((async () => {
    try {
      const db = await idbOpen();
      const auth = await new Promise((res) => {
        const r = db.transaction(STORE_AUTH).objectStore(STORE_AUTH).get('current');
        r.onsuccess = () => res(r.result); r.onerror = () => res(null);
      });
      if (!auth?.token) return;
      const all = await new Promise((res) => {
        const r = db.transaction(STORE_PENDING).objectStore(STORE_PENDING).getAll();
        r.onsuccess = () => res(r.result || []); r.onerror = () => res([]);
      });
      all.sort((a, b) => String(a.criadoEm).localeCompare(String(b.criadoEm)));
      let sent = 0;
      for (const p of all) {
        try {
          // dispatch por tipo — duplica logica do client.sendOne aqui
          if (p.type === 'comentario') {
            await execSF(auth, 30 /* criarComentario */, p.payload);
            // ... outras SFs do fluxo
          } else if (p.type === 'pedido') {
            await execSF(auth, 50 /* criarPedido */, p.payload);
          }
          await new Promise((res) => {
            const r = db.transaction(STORE_PENDING, 'readwrite').objectStore(STORE_PENDING).delete(p.localId);
            r.onsuccess = () => res(); r.onerror = () => res();
          });
          sent++;
        } catch (err) { console.warn('[sw sync]', err); break; }
      }
      // Notifica clients abertos
      const list = await self.clients.matchAll({ type: 'window', includeUncontrolled: true });
      for (const c of list) c.postMessage({ type: 'app:offline-flush', sent });
    } catch (err) { console.warn('[sw sync] fatal', err); }
  })());
});
```

### 7.3 Client escuta o resultado

```tsx
function handleSwMessage(e: MessageEvent) {
  if (e.data?.type === 'app:offline-flush') {
    window.dispatchEvent(new CustomEvent('app:offline-flush', { detail: e.data }));
    refreshPending();
    toast.success(`${e.data.sent} item(ns) enviado(s)`);
  }
}
navigator.serviceWorker.addEventListener('message', handleSwMessage);
```

### 7.4 Trampas

- **Schema do IDB:** o SW pode abrir o IDB antes do client. Defina `onupgradeneeded` no SW também, idempotente, criando todos os stores.
- **Versão do IDB:** mantenha sincronizada entre client e SW. Bump dos dois juntos.
- **Token expirado:** SW não tem renovação automática. Se voltar 403, marque a fila como "precisa do client" e aguarde reabrir.
- **Sync silencioso:** Android pode adiar o sync por bateria, doze mode, etc. Não dependa de delay garantido.

---

## 8. Erros benignos do SDK — não quebrar a UI offline

⚠️ **Trampa importante:** o SDK Mitra dispara um heartbeat de tracking via `fetch(...)` sem `.catch`. Offline, isso vira `unhandledrejection: Failed to fetch`.

Se você captura unhandledrejection e mostra tela de erro, **o app fica preso em erro mesmo funcionando perfeitamente**.

A solução do §2.4 já trata: `isBenignNetwork(msg)` ignora "Failed to fetch" e similares. **Sempre filtre antes de mostrar erro fatal.**

---

## 9. Painel de diagnóstico (no Perfil)

Pra o usuário (e você) saberem se o app tá pronto pra offline:

```tsx
const allUrls: string[] = cacheInventory ? Object.values(cacheInventory).flat() : [];
const hasShell = allUrls.includes('/index.html');
const hasJs = allUrls.some(u => /\.m?js(\?|$)/i.test(u));
const hasCss = allUrls.some(u => /\.css(\?|$)/i.test(u));
const offlineReady = hasShell && hasJs && hasCss;

return (
  <div>
    <div>App: <b>{APP_VERSION}</b> · SW: <b>{swVersion || '...'}</b></div>
    <div>Pronto pra offline: {offlineReady ? 'Sim' : 'Nao'}</div>
    <div>shell {hasShell ? 'OK' : 'X'} · js {hasJs ? 'OK' : 'X'} · css {hasCss ? 'OK' : 'X'}</div>
    {lastError && <pre>{lastError}</pre>}
    <button onClick={clearAllCaches}>Limpar cache do app</button>
  </div>
);
```

`getServiceWorkerVersion` e `getCacheInventory` mandam `MessageChannel` pro SW.

⚠️ **Não basta contar arquivos.** Mostrar "5 arquivos em cache" não diz nada — o JS pode estar faltando. Sempre valide os 3 críticos: shell, JS, CSS.

---

## 10. Versionamento App + SW

- `src/lib/version.ts`: `APP_VERSION = 'v14'` + `APP_BUILD_AT = '2026-05-06 21:30'`.
- `sw.js`: `const SW_VERSION = 'v14'`.
- **Sempre bump os dois juntos** quando mudar SW ou cache.
- O painel mostra "Atualizado" se baterem, "Recarregue" se divergirem (= o usuário ainda tá com SW antigo, precisa fechar e reabrir).

---

## 11. Checklist de implantação (passo a passo)

Pro outra IA seguir do zero:

1. **PWA básico:** `manifest.json`, ícones, meta tags no `index.html` (§2). Bate o build, abre no celular, instala via Add to Home Screen.
2. **Splash + error capture:** copiar o `index.html` do §2.4 inteiro. Substituir nome/cor.
3. **Service Worker mínimo:** cria `sw.js` (§3.1-3.3). Registra no `index.html`. Confirma no DevTools que tá ativo.
4. **VAPID:** gera as chaves (`npx web-push generate-vapid-keys`), salva nas variáveis do projeto (§4.2).
5. **Tabela e SFs de push:** roda backend setup (§4.3-4.5). Confere que `enviarPush` aparece em `listServerFunctionsMitra`.
6. **Frontend push:** `lib/push.ts` (§4.6) + botão no Perfil (§4.7). Testa: ativa → vê confirmação → checa que `PUSH_SUBSCRIPTIONS` populou.
7. **SW push handlers:** `'push'` + `'notificationclick'` (§4.8). Testa: dispara `enviarPush` manualmente → notificação chega → clique abre o app na rota certa.
8. **Cache de leitura:** `callSFCached` (§5). Aplica em 2-3 telas-chave. Testa offline.
9. **Cache do profile:** crítico — sem isso ações offline quebram silenciosas.
10. **Queue de writes:** `enqueueWrite`, optimistic UI, `flushPending` (§6).
11. **Flush triggers:** todos os listeners (§6.4). Testa em background → foreground.
12. **Background Sync:** auth no IDB (§7.1) + handler `sync` no SW (§7.2). Testa Android: comenta offline → fecha app → liga internet → espera 30s → abre app, deve estar enviado.
13. **Painel de diagnóstico** (§9). É o que salva sua vida quando o usuário disser "não funciona".
14. **Versionamento** (§10) — bump em cada deploy de SW.

---

## 12. iOS — o que muda

- ✅ PWA via Add to Home Screen.
- ✅ Push (apenas instalado na home, e só iOS 16.4+).
- ✅ Notification click abre o PWA.
- ❌ Background Sync.
- ⚠️ Cache de SW: o iOS limita storage e pode purgar agressivamente.
- ⚠️ Push payload limitado a ~3KB (Apple Push Service).
- ⚠️ A primeira vez que pedir permissão **tem que ser via clique direto** (gesto do usuário). Não dispare automático.

Mensagem honesta pro usuário iOS: "Comentários offline são enviados quando você reabrir o app". Não prometa background.

---

## 13. Bugs comuns — checklist de troubleshooting

| Sintoma | Causa provável | Onde olhar |
|---|---|---|
| "This site can't be reached" offline | SW retornou `undefined` em algum branch | §3.3 (todos respondWith retornam `Response`) |
| Tela branca offline | Asset (JS/CSS) não no cache | §3.2 (regex de precache normaliza `./`) |
| Tela branca + "Failed to fetch" | Heartbeat do SDK não filtrado | §2.4 + §8 |
| Push chega mas clique cai em rota 404 | HashRouter sem prefixo `/#/` | §4.8 |
| Comentário offline some quando reabre | Profile não cacheado, handleSubmit faz early-return | §5.3 (cache profile) |
| Comentário não envia em background no Android | Falta Background Sync registrado | §7 |
| "Pronto pra offline: Sim" mas não funciona | Diagnóstico só conta arquivos sem validar shell/JS/CSS | §9 |
| Push 410 Gone | Subscription expirada — apaga da tabela | §4.5 (já trata no `enviarPush`) |
| SW novo não ativa | `skipWaiting` faltando ou usuário não fechou todas as abas | §3.2 (`self.skipWaiting()` no install) |

---

## 14. Resumão das decisões de arquitetura

- **HashRouter > BrowserRouter** pra simplificar SW e deploys estáticos.
- **IndexedDB > localStorage** pra dados offline (acessível ao SW, sem limite de 5MB).
- **`base: './'` no Vite** com SW que normaliza paths — funciona em qualquer servidor estático.
- **Token espelhado no IDB** pro SW conseguir mandar requests em background.
- **Versionamento explícito** App + SW no painel — usuário sabe se atualizou.
- **Filtro de erros benignos** — heartbeat falhar offline é normal, não é fatal.
- **Optimistic UI sempre** — usuário vê o item na hora, depois sincroniza.
- **iOS é parcial e tudo bem** — comunique a limitação claramente.
