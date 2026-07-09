# Kundali Web Bridge — Web Developer Contract & Implementation Guide

This repo hosts the **test harness** (`index.html`, live below) and the **contract** for the web page that the Dharmayana app embeds as the **Kundali Viewer**.

The app loads your web page inside a native container and talks to it over a small **JSON message bridge**. This document is everything you need to build the page so it behaves like a native screen: navigation into the app, native share (incl. sharing the kundali PDF), native reminders, analytics, and hardware-back handling.

- **Live test harness:** https://oit-prabhat-patel.github.io/kundali-bridge-test/
- **Source of the harness:** [`index.html`](./index.html) — a working reference implementation of everything below. When in doubt, copy from it.

> ⚠️ The app end is on branch `feature/kundali-webview-viewer` (PR #5241). See [Open items](#14-open-items-coordinate-with-the-app-team) for the remaining coordination points.

---

## 1. How the page is embedded

| Platform | Container | Transport |
|---|---|---|
| Android / iOS | native WebView (`webview_flutter`) | a JS channel named **`AppWebBridge`** |
| Web (Flutter web) | an `<iframe>` | **`window.postMessage`** |

Your page runs the **same code** on all three — the tiny SDK in §4 abstracts the transport. Detect the environment once and forget about it.

**Kundali viewer specifics:**
- The screen is **full-bleed — there is no native app bar.** Your page owns its own header and its own **back/close affordance** (important on iOS, which has no hardware back button).
- It runs **with an auth token**: the app sends it in **`init.auth.token`** (never in the URL). Use it to fetch the signed-in user's kundali.
- **The app loads the exact URL the order provides** — `order.digital_report_url` from the order-detail API, used **verbatim** (it already carries `?order_id=…`; the app appends nothing).
- Must be served over **HTTPS** (iOS App Transport Security).

**How the viewer is opened (app-side — relevant if you generate links):**
- From the **order-detail screen** (the kundali card): the app passes `order.digital_report_url` + `orderId`.
- Via **deeplink**: `dharmayana://orders/viewer?id=<orderId>&url=<url-encoded viewer URL>` — opens the viewer with the order-detail screen **behind** it (back returns there). The `url` must be **percent-encoded** (it carries its own `?order_id=…`) and must be an `https://*.dharmayana.in` URL. Auth-gated: a logged-out user is sent to login first.

---

## 2. Message envelope

Every message in either direction is a **JSON string** of this shape:

```jsonc
{
  "v": 1,                 // protocol version, always 1 for now
  "type": "navigate",     // message type (see §5 / §6)
  "id": "a12",            // OPTIONAL correlation id (only for request/response: actions, back)
  "payload": { }          // OPTIONAL, type-specific
}
```

- Always send `v: 1`.
- Only include `id` when you need a response correlated back (actions with a result, and back replies).
- Unknown/malformed messages are safely ignored by the app — and your page should ignore anything it doesn't recognize too.

---

## 3. Transport details (what the SDK does under the hood)

### Native (Android/iOS)
- **Page → App:** `window.AppWebBridge.postMessage(JSON.stringify(msg))`
- **App → Page:** the app calls **`window.__appWebBridge.receive("<json string>")`** — so your page **must define** `window.__appWebBridge = { receive: (s) => { ... } }` before/at load.

### Web (iframe)
- **Page → App:** `window.parent.postMessage(JSON.stringify(msg), APP_ORIGIN)`
- **App → Page:** the app posts a JSON string to the iframe; your page listens:
  ```js
  window.addEventListener('message', (e) => {
    // In production, verify e.origin === APP_ORIGIN before trusting it.
    if (typeof e.data !== 'string') return;
    handle(JSON.parse(e.data));
  });
  ```
- **Origin pinning (production):** post to the app's real origin — the Flutter web app runs at `https://app.dharmayana.in` (stage `https://app.stage.dharmayana.in`), not `'*'` — and verify inbound `e.origin`. The app pins **your** origin **subdomain-aware to `*.dharmayana.in`**: it loads, accepts messages from, and sends the auth token to `https://dharmayana.in` or **any** subdomain (`www.dharmayana.in`, `web.stage.dharmayana.in`, `www.web.stage.dharmayana.in`, …). Any non-`dharmayana.in` host is refused — it won't load and never receives the token.

---

## 4. Drop-in reference SDK (`AppBridge`)

Include this once. It handles transport detection, the envelope, request/response correlation for actions, and the back handler. (This is the exact SDK used by the live test harness.)

```html
<script>
const isNative = !!(window.AppWebBridge && window.AppWebBridge.postMessage);
const APP_ORIGIN = '*'; // testing only — in production set to the app origin:
                        // https://app.dharmayana.in (stage: https://app.stage.dharmayana.in)

function _post(msg) {
  msg.v = 1;
  const s = JSON.stringify(msg);
  if (isNative) window.AppWebBridge.postMessage(s);
  else window.parent.postMessage(s, APP_ORIGIN);
}

let _seq = 0;
const _pending = new Map();     // id -> {resolve, timer}
let _backHandler = () => false; // return true if the page handled back internally

const AppBridge = {
  // lifecycle
  ready:   () => _post({ type: 'ready' }),
  onInit:  (cb) => { AppBridge._onInit = cb; },

  // navigation into the app (fire-and-forget)
  navigate: (uri) => _post({ type: 'navigate', payload: { uri } }),

  // native actions (return a Promise that resolves with { ok, error? })
  action: (uri, data = {}) => new Promise((resolve) => {
    const id = 'a' + (_seq++);
    const timer = setTimeout(() => { _pending.delete(id); resolve({ ok: false, error: 'timeout' }); }, 8000);
    _pending.set(id, { resolve, timer });
    _post({ type: 'action', id, payload: { uri, data } });
  }),
  share:       (data) => AppBridge.action('app://share', data),
  sharePdf:    () => AppBridge.action('app://share-pdf'), // app resolves + shares the order's PDF
  setReminder: (data) => AppBridge.action('app://reminder', data),

  // analytics (fire-and-forget)
  track: (event, params = {}) => _post({ type: 'analytics', payload: { event, params } }),

  // screen control
  setTitle: (title) => _post({ type: 'setTitle', payload: { title } }),
  close:    () => _post({ type: 'close' }),
  error:    (message) => _post({ type: 'error', payload: { message } }),

  // back: fn() must return true if the page navigated back internally, false if at root
  setBackHandler: (fn) => { _backHandler = fn; },

  _receive(msg) {
    switch (msg.type) {
      case 'init':        AppBridge._onInit && AppBridge._onInit(msg.payload || {}); break;
      case 'backRequest': _post({ type: 'backResult', id: msg.id, payload: { handled: !!_backHandler() } }); break;
      case 'result': {
        const p = _pending.get(msg.id);
        if (p) { clearTimeout(p.timer); _pending.delete(msg.id); p.resolve(msg.payload || { ok: true }); }
        break;
      }
    }
  },
};

// wire transports
window.__appWebBridge = { receive: (s) => { try { AppBridge._receive(JSON.parse(s)); } catch (_) {} } };
if (!isNative) {
  window.addEventListener('message', (e) => {
    if (typeof e.data !== 'string') return;
    try { AppBridge._receive(JSON.parse(e.data)); } catch (_) {}
  });
}
window.AppBridge = AppBridge;
</script>
```

### Minimal usage

```js
AppBridge.onInit((init) => {
  // init = { locale, platform, data: { orderId, contentType }, auth: { token } }
  renderKundali(init.data.orderId, init.locale, init.auth.token);
});
AppBridge.setBackHandler(() => myRouter.canGoBack() ? (myRouter.back(), true) : false);
AppBridge.ready(); // call once your bridge + first paint are ready
```

---

## 5. Page → App messages (what you send)

| type | payload | result? | meaning |
|---|---|---|---|
| `ready` | – | no | You're ready; the app replies with `init`. Send once on load. |
| `navigate` | `{ uri }` | no | Open a native screen via an app deeplink (see §7). |
| `action` | `{ uri, data? }` (+ `id`) | yes (if `id`) | Run a native action: share / share-pdf / reminder (see §8). |
| `analytics` | `{ event, params }` | no | Log an analytics event (see §9). |
| `backResult` | `{ id, handled }` | – | Reply to a `backRequest` (see §10). |
| `close` | – | no | Ask the app to close the viewer screen. |
| `setTitle` | `{ title }` | no | Set the native app-bar title. **No-op for kundali** (app bar hidden). |
| `error` | `{ message }` | no | Tell the app rendering failed → it shows a retry UI. |

## 6. App → Page messages (what you receive)

| type | payload | when |
|---|---|---|
| `init` | `{ locale, platform, data, auth? }` | in response to your `ready` |
| `backRequest` | `{ id }` (no payload) | user pressed device/app back — you must reply `backResult` |
| `result` | `{ id, ok, error? }` | the outcome of an `action` you sent with an `id` |

**`init` payload:**
```jsonc
{
  "locale": "hi",            // 2-letter app language: en | hi | kn | gu | mr | ta | te
  "platform": "android",     // android | ios | web
  "data": {                  // consumer-specific — for kundali:
    "orderId": "CR7SUB-764076", // the order whose kundali to render
    "contentType": "kundali"
  },
  "auth": { "token": "..." } // PRESENT for kundali — use it to fetch the user's kundali
}
```
> Render using `data.orderId` **and** `auth.token`. The token arrives here over the bridge — it is **never** in the page URL.

---

## 7. Navigation (`navigate`)

Send an **app deeplink URI**; the app routes it through its internal deeplink dispatcher — so you can open **any screen the app supports**.

```js
AppBridge.navigate('dharmayana://pooja');                       // Pooja home
AppBridge.navigate('dharmayana://pooja/details?id=<poojaId>');  // a specific pooja
AppBridge.navigate('dharmayana://kundli_generation/details?id=<packageId>');
```

**Format:** `dharmayana://<host>[/<subfeature>][?<query>]`
- Scheme **must** be `dharmayana`. Any other scheme is rejected.
- Homepages need no id: `dharmayana://<host>`.

**Supported hosts** (the app's deeplink features):

`panchanga` · `prarthana` · `kundli_generation` · `festivals` · `muhurta` · `festival` · `prediction` · `rashifal` · `articles` · `order` · `settings` · `astro_consultation` · `ayurveda_consultation` · `dharmik_counselling` · `pooja` · `dhyana` · `chp` · `wallpaper` · `streak` · `referral` · `hindu_tithi`

> **Allow-list (kundali: none — all deeplinks allowed):** the kundali viewer accepts **any** `dharmayana://` deeplink, current and future. Deeplinks are a public entry point (any browser/link can already invoke them), so gating them here adds no real security — the app's deeplink handlers are the boundary. The scheme must still be `dharmayana://` (other schemes are rejected), and the bridge-only capabilities (**actions** §8 and **analytics** §9) remain gated because those *aren't* reachable from a browser. The bridge can optionally restrict hosts per-consumer, but kundali does not.

`navigate` is fire-and-forget — no `result`.

---

## 8. Native actions (`action`)

URI form `app://<name>`, allow-listed. Await the returned Promise for the `result`.

### 8.1 Share — `app://share`
```js
const res = await AppBridge.share({ text: 'Check my kundali', url: 'https://dharmayana.in' });
// or an image: AppBridge.share({ text: '...', imageUrl: 'https://.../chart.png' })
// res = { ok: true }
```
| field | type | notes |
|---|---|---|
| `text` | string? | share text |
| `url` | string? | appended to text |
| `imageUrl` | string? | if present, the app fetches + shares the image with `text` |

Opens the native OS share sheet. `ok:true` is best-effort (OS share completion isn't reliably reported).

### 8.2 Reminder — `app://reminder`
Opens the native reminder bottom sheet. **It toggles set vs delete automatically**, exactly like the app's order-detail remedy cards: the app looks up whether a reminder already exists (by `orderId` + `id` + `recurrenceKey`) and opens the sheet in **delete** mode if so, **set** mode otherwise.

```js
const res = await AppBridge.setReminder({
  orderId: 'TEST123',
  id: 'remedy-1',
  name: 'Gayatri Mantra',
  frequencyDays: 'monday,tuesday,wednesday,thursday,friday,saturday,sunday',
  reminderTime: '08:00 AM',
  reminderTimes: ['08:00 AM', '07:00 PM'],
  endDate: '30 Aug 2026',
  recurrenceKey: 'daily__08:00,19:00',
  reminderType: 'dhyana',   // or omit for a regular reminder
  orderType: 'kundali'
});
// native → res = { ok: true } ;  web → res = { ok: false } (reminder sheet is native-only)
```

| field | type | notes |
|---|---|---|
| `orderId` | string | identifies the reminder together with `id` + `recurrenceKey` |
| `id` | string | the remedy/item id |
| `name` | string | display name shown in the sheet |
| `frequencyDays` | string | comma-separated lowercase day names; daily = all 7 |
| `reminderTime` | string | first time, `hh:mm AM/PM` |
| `reminderTimes` | string[] | all times, `hh:mm AM/PM` |
| `endDate` | string | `dd MMM yyyy` |
| `recurrenceKey` | string | stable key that distinguishes schedules, e.g. `"<type>_<days>_<times>"` — **must be identical** across set & delete for the toggle to work |
| `reminderType` | string? | `"dhyana"` or omit |
| `orderType` | string | e.g. `"kundali"` |

> Reminder is **native-only**. On web the action resolves `{ ok: false }` — hide/disable the reminder UI when `init.platform === 'web'`.
> Not yet wired: the order-specific "mark remedy complete" API side-effect. Ask the app team if your page needs it.

### 8.3 Share PDF — `app://share-pdf`
Shares the order's **kundali PDF as a native file** (not a link). Takes **no data** — the app resolves the file itself: it fetches the order by the `orderId` from `init` and shares its `digital_shipment` attachment (a fresh signed URL), with a download-progress overlay.
```js
const res = await AppBridge.sharePdf();   // == AppBridge.action('app://share-pdf')
// res = { ok: true }  — false if the order has no PDF or the download failed
```
Use this to share the actual PDF document. (`app://share` still shares plain text/url/image payloads that *you* supply.)

### Adding new actions
`share`, `share-pdf`, and `reminder` exist today. New actions require an app-side change + allow-list entry — coordinate with the app team.

---

## 9. Analytics (`analytics`)
```js
AppBridge.track('kundali_viewed', { section: 'dosha' });
```
Fire-and-forget; routed to the app's analytics stack (Mixpanel / CleverTap / Firebase). Rules the app enforces:
- **Rejected:** event names starting with `be_` (reserved server-side namespace) or the reserved name `app_launched`.
- The app **stamps `source: "kundali_web"`** on every event and **overrides** any `source` you pass (anti-spoof). Don't rely on setting `source` yourself.
- Use clear `snake_case` event names + a flat `params` object of primitives.

---

## 10. Back button contract (web-history-first)

When the user presses the device back button (Android) or the browser back button (web), the app sends you a **`backRequest`**. You reply **`backResult`**: (kundali has no native app bar, so your own page control is the only on-screen back — see below.)

- `handled: true` → **you** navigated back within the page (the app does nothing).
- `handled: false` → you're at your root → **the app pops the viewer screen.**

The SDK does this for you via `setBackHandler`:
```js
AppBridge.setBackHandler(() => {
  if (myRouter.canGoBack()) { myRouter.back(); return true; }
  return false; // at root → let the app close the screen
});
```

Rules:
- **Reply fast** — if you don't reply within ~**400 ms**, the app pops anyway (so a hung page never traps the user). Keep the handler synchronous.
- On **web**, the browser back button *also* naturally steps through your iframe history first (use real `history.pushState` navigations), then leaves the screen.
- You can also proactively `AppBridge.close()` (e.g. an on-page ✕ or a "back to app" button). **`close` pops the viewer immediately, ignoring web history** — it's the "just pop the native screen" control, distinct from the `backRequest` round-trip above. Because the kundali screen has **no native app bar**, provide a visible back/close control in your page — essential on iOS.

---

## 11. Security model (what the app enforces)
- **Origin lock (subdomain-aware):** the viewer only loads — and only accepts messages from, and only sends the auth token to — `https://*.dharmayana.in` (§3). Enforced on the **initial load too**, so even a deeplink-supplied `url` (§1) outside `dharmayana.in` is refused. Links to other origins open in the system browser, not inside the viewer.
- **Config-key routing:** the app opens the viewer by a registered config key; the page URL is the order's `digital_report_url` or a deeplink-supplied `url`, **both origin-checked before load** — you can't be turned into an open redirect to a non-`dharmayana.in` host.
- **navigate:** scheme-checked (`dharmayana://` only), not host-restricted for kundali (§7); the **action allow-list** (§8) and **analytics denylist/source-tag** (§9) remain enforced (those aren't browser-reachable).
- **Do not** use `window.location = 'dharmayana://...'` or `<a href="dharmayana://...">` to drive app navigation — those are blocked. Use `AppBridge.navigate(...)`.

---

## 12. Testing

1. **Live harness:** open https://oit-prabhat-patel.github.io/kundali-bridge-test/ in a browser — it renders in "web" mode and logs every message. It has a button for every capability above (and negative tests).
2. **In the app:** the app is pointed at this page's origin for E2E; opening the viewer loads it, sends `init`, and the buttons exercise real native navigation / share / reminder / back.
3. Build your page to the same contract, host it, and give the app team its URL + origin to point at.

---

## 13. Web-page checklist
- [ ] Include the SDK (§4); set `APP_ORIGIN` to `https://app.dharmayana.in` (stage `https://app.stage.dharmayana.in`) for production.
- [ ] Call `AppBridge.ready()` on load; render from `onInit` (`data.orderId`, `locale`).
- [ ] Provide a visible **back/close** control (no native app bar); implement `setBackHandler`.
- [ ] Use `AppBridge.navigate(...)` for app links (never `window.location`).
- [ ] Gate reminder UI on `init.platform !== 'web'`.
- [ ] Use `history.pushState` for internal navigation (so web back works).
- [ ] Read the **auth token** from `init.auth.token` (present for kundali) — don't expect it in the URL.
- [ ] Give the app team the exact **`digital_report_url`** shape your order API returns (any `https://*.dharmayana.in` host + path is fine).

## 14. Open items (coordinate with the app team)
- ~~page URLs/origins~~ — **decided:** the app trusts **`https://*.dharmayana.in`** (apex + any subdomain, prod & stage) and loads the order's **`digital_report_url`** verbatim (it already carries `?order_id=…`; the app appends nothing).
- ~~auth~~ — **decided: kundali needs auth.** The app sends `init.auth.token` over the bridge (never in the URL).
- ~~navigate allow-list~~ — **decided: kundali allows all deeplinks** (no allow-list; deeplinks are a public surface).
- Exact **`init.data`** contents beyond `{ orderId, contentType }`, if anything.
- Confirm reminder field expectations match your data model.
- Whether the "mark remedy complete" side-effect is needed.
