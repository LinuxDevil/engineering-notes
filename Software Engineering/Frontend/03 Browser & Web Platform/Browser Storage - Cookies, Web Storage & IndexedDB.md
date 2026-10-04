---
title: "Browser Storage - Cookies, Web Storage & IndexedDB"
tags: [frontend, frontend/browser, frontend/storage]
aliases: [Browser Storage, Cookies vs Local Storage vs Session Storage, Local Storage vs Session Storage vs Cookie, IndexedDB, Web Storage, Client-side storage]
updated: 2026-09-27
---
# Browser Storage - Cookies, Web Storage & IndexedDB

<Note>
**TL;DR**

HTTP is stateless, so browsers give us several places to keep state: **cookies** (small, sent to the server automatically, the only option the server can read), **Web Storage** (`localStorage` / `sessionStorage`: small, synchronous, string-only, client-only), **IndexedDB** (large, async, structured data and blobs), the **Cache API** (Request/Response pairs, used by service workers) and **OPFS** (a private file system). Pick by: who needs to read it (server or client), how big it is, how long it must live, and how sensitive it is.
</Note>

## 1. The big comparison

![Untitled 23](/Attachments/Untitled%2023.png)

| | Cookies | localStorage | sessionStorage | IndexedDB | Cache API | OPFS |
|---|---|---|---|---|---|---|
| **Capacity** | ~4 KB per cookie (and a per-domain count limit) | ~5 MB per origin | ~5 MB per origin (per tab) | Large: a share of disk quota (GBs) | Shares the origin quota | Shares the origin quota |
| **Lifetime** | Session, or until `Expires`/`Max-Age` | Until cleared by code/user/eviction | Until the tab (page session) closes | Until cleared/evicted | Until cleared/evicted | Until cleared/evicted |
| **Sent to server** | **Yes, automatically** with every matching request | No | No | No | No | No |
| **Readable by** | Server; JS too unless `HttpOnly` | JS (main thread only) | JS (main thread, that tab only) | JS (window + workers) | JS (window + workers + service worker) | JS (workers get sync file access) |
| **API** | `Set-Cookie` header / `document.cookie` / `cookieStore` | Sync, strings only | Sync, strings only | **Async**, structured clone (objects, Blobs, Files, ArrayBuffers), indexes, transactions | Async, `Request` → `Response` | Async; sync access handles in workers |
| **Scope** | Domain + path (+ SameSite / partition) | Origin | Origin + tab | Origin | Origin | Origin |
| **Typical use** | Session IDs, auth tokens (HttpOnly), server-read preferences, A/B bucket | UI prefs, theme, small non-sensitive cache | Wizard/form progress for one tab, scroll positions | Offline data, drafts, large caches, files | Offline assets, network responses (PWA) | Large files, SQLite-in-the-browser (Wasm) |

Quick decision guide:
- The **server** must see it on every request → cookie (keep it tiny).
- Small, non-sensitive, client-only, must survive restart → `localStorage`.
- Only for this tab's session → `sessionStorage`.
- Big, structured, or needed offline / in a worker → IndexedDB.
- Caching HTTP responses for offline → Cache API in a service worker ([Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs)).
- Raw files / databases (e.g. SQLite Wasm) → OPFS.
- Shareable UI state (filters, pagination) → the URL, not storage ([URL as State](/Software%20Engineering/Frontend/06%20State%20Management/URL%20as%20State)).

## 2. Cookies

Small pieces of data the **server** (or JS) stores in the browser. Because HTTP is stateless, the browser automatically sends matching cookies with each request to that server, which lets the server identify the user and keep state across requests (session management, preferences, tracking).

```http
HTTP/1.1 200 OK
Set-Cookie: __Host-session=abc123; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=3600
Set-Cookie: theme=dark; Path=/; Max-Age=31536000; SameSite=Lax
```

```http
GET /api/bookings HTTP/1.1
Cookie: __Host-session=abc123; theme=dark
```

### Attributes

| Attribute | Meaning |
|---|---|
| `Expires=<date>` / `Max-Age=<seconds>` | Makes the cookie **persistent**. Without either it's a **session cookie**, deleted when the browser session ends (browsers with "restore session" may keep them). `Max-Age` wins if both are set. `Max-Age=0` deletes. |
| `Domain` | Which hosts receive it. Omitted → only the exact host that set it (safest). `Domain=example.com` → also all subdomains. |
| `Path` | URL path prefix that receives it (not a security boundary). |
| `Secure` | Only sent over **HTTPS** (and can only be set from secure origins). |
| `HttpOnly` | **Not readable by JavaScript** (`document.cookie` can't see it). Limits damage from XSS: an attacker's script can't steal the token, although it can still make requests as the user. |
| `SameSite=Strict` | Never sent on cross-site requests (even top-level link clicks from another site). |
| `SameSite=Lax` | Sent on same-site requests and on top-level **GET navigations** from other sites; not on cross-site POSTs, iframes, or `fetch`. Chromium treats cookies with no `SameSite` as `Lax`. |
| `SameSite=None` | Sent in all contexts, **must** also be `Secure`. Needed for real third-party use (embedded widgets), increasingly blocked by browsers' third-party cookie restrictions. |
| `Partitioned` (**CHIPS**) | "Cookies Having Independent Partitioned State": a third-party cookie that is stored **per top-level site**. An embedded chat widget on `a.com` and on `b.com` gets two separate cookie jars, so it works without enabling cross-site tracking. Requires `Secure`. |

Cookie name prefixes (browser-enforced):
- `__Secure-` : must be set with `Secure` from an HTTPS page.
- `__Host-` : must be `Secure`, `Path=/`, and have **no `Domain`**. It is locked to the exact host, so a compromised subdomain can't overwrite it. The best choice for session cookies.

<Info>
**Reading & writing cookies from JS**

```js
// Old API: one big string
document.cookie = 'theme=dark; Max-Age=31536000; Path=/; SameSite=Lax';
const theme = document.cookie.split('; ').find(c => c.startsWith('theme='))?.split('=')[1];

// Cookie Store API: async, promise-based, also usable in service workers
await cookieStore.set({ name: 'theme', value: 'dark', expires: Date.now() + 365 * 864e5 });
const c = await cookieStore.get('theme');
cookieStore.addEventListener('change', (e) => console.log(e.changed, e.deleted));
```
The Cookie Store API is not supported in every browser; feature-detect `'cookieStore' in window`.
</Info>

### Cookies: pros & cons (for session tokens)
- **Pros**: sent automatically with each request; built-in security flags (`HttpOnly`, `Secure`, `SameSite`); the server controls lifetime.
- **Cons**: vulnerable to **CSRF** if `SameSite`/CSRF tokens aren't used; small size; sent on **every** request (bandwidth, including images and static files on the same domain, so serve static assets from a cookie-less domain/CDN); scoped by domain/path, not by strict origin.

Deep dive on tokens: [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage), [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies), CSRF in [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).

## 3. Web Storage: localStorage & sessionStorage

Both implement the same `Storage` interface: key/value **strings**, synchronous, per origin.

```js
localStorage.setItem('recentSearches', JSON.stringify(['DXB-LHR', 'CAI-JED']));
const recent = JSON.parse(localStorage.getItem('recentSearches') ?? '[]');
localStorage.removeItem('recentSearches');
localStorage.clear();
localStorage.length; localStorage.key(0);

sessionStorage.setItem('checkoutStep', '2');
```

### localStorage
Stores key/value pairs with **no expiration date**. It stays until your code removes it, the user clears site data, or the browser evicts it.
- **Pros**: persists across restarts; ~5 MB per origin (the owner's notes mention 5–10 MB; in practice most browsers give 5 MB); data is never sent to the server; simple API.
- **Cons**: **plaintext** and readable by any script on the page, so an XSS can steal everything (never store tokens or PII); strings only, so everything must be serialized; **synchronous**, blocks the main thread (a big `JSON.parse` on startup is noticeable); not available in workers or service workers; client-side only.
- Uses: user preferences (theme, language, currency), dismissed banners, small caches, app state that should survive reloads.

### sessionStorage
Same API, limited lifecycle:
- Data lives for the **page session**: it survives reloads and restores of the same tab, and is cleared when the **tab or window is closed** (not the whole browser).
- It is **per tab**: opening the same URL in two tabs gives two separate `sessionStorage`s. "Duplicate tab" copies the current values into the new tab.
- Used for temporary data: multi-step form progress, per-tab state, scroll restore.
- As a token store: cleared automatically when the session ends (a pro), but requires re-authentication in every new tab and is still XSS-readable.

### The `storage` event (cross-tab sync)

```js
// Fires in OTHER tabs of the same origin when localStorage changes
window.addEventListener('storage', (e) => {
  if (e.key === 'auth:logout') location.assign('/login'); // log out everywhere
});
localStorage.setItem('auth:logout', Date.now().toString());
```
For richer cross-tab messaging use `BroadcastChannel`.

<Warning>
**Common pitfalls**

- `localStorage` may throw: `QuotaExceededError` when full, and `SecurityError` when storage is blocked (some privacy modes, sandboxed iframes). Wrap reads/writes in `try/catch`.
- Storing `true` gives back `"true"`; storing an object without `JSON.stringify` gives `"[object Object]"`.
- It's not available during SSR (`window` is undefined on the server). Read it in an effect or guard with `typeof window !== 'undefined'`, and beware hydration mismatches.
- Safari's Intelligent Tracking Prevention can delete script-writable storage (localStorage, IndexedDB, etc.) for sites the user hasn't interacted with in 7 days (installed home-screen web apps are exempt). Don't treat it as permanent.
</Warning>

## 4. IndexedDB

A low-level, **asynchronous**, transactional database in the browser for structured data, files and blobs. Values are stored with the **structured clone** algorithm (objects, arrays, Dates, Maps, Blobs, ArrayBuffers; no functions or DOM nodes).

Concepts:
- **Database** (versioned) → **object stores** (like tables) → records keyed by a `keyPath` or auto-increment key.
- **Indexes** on properties for fast lookups and range queries (`IDBKeyRange`).
- **Transactions**: `readonly` or `readwrite`, auto-commit when no requests are pending. Schema changes happen only in `onupgradeneeded`.
- Available in windows, **web workers and service workers**.

The raw API is event-based and verbose. Most apps use a tiny promise wrapper like **`idb`** (by Jake Archibald) or a higher-level library such as **Dexie**.

<Info>
**Offline saved searches with `idb`**

```js
import { openDB } from 'idb';

const db = await openDB('flights', 2, {
  upgrade(db, oldVersion) {
    if (oldVersion < 1) {
      const store = db.createObjectStore('searches', { keyPath: 'id' });
      store.createIndex('byCreatedAt', 'createdAt');
    }
    if (oldVersion < 2) db.createObjectStore('drafts'); // out-of-line keys
  },
});

await db.put('searches', { id: 'DXB-LHR-2026-10-01', from: 'DXB', to: 'LHR', createdAt: Date.now() });
const latest = await db.getAllFromIndex('searches', 'byCreatedAt'); // sorted ascending

// Atomic multi-write
const tx = db.transaction(['searches', 'drafts'], 'readwrite');
await Promise.all([tx.objectStore('drafts').put(formState, 'checkout'), tx.done]);
```
</Info>

<Info>
**The raw API (for interviews)**

```js
const req = indexedDB.open('flights', 1);
req.onupgradeneeded = () => req.result.createObjectStore('searches', { keyPath: 'id' });
req.onsuccess = () => {
  const db = req.result;
  const tx = db.transaction('searches', 'readwrite');
  tx.objectStore('searches').put({ id: 'a', from: 'DXB' });
  tx.oncomplete = () => console.log('saved');
};
req.onerror = () => console.error(req.error);
```
</Info>

Pitfalls: a transaction auto-commits if you `await` something unrelated (like `fetch`) in the middle; opening a newer version while another tab holds the old one triggers `blocked`/`versionchange` (handle it by closing the old connection).

## 5. Cache API

Stores `Request` → `Response` pairs. Designed for service workers but also available on `window`.

```js
const cache = await caches.open('static-v3');
await cache.addAll(['/', '/app.css', '/app.js']);
const res = await caches.match('/app.css');
await caches.delete('static-v2'); // clean up old versions on activate
```

It is **not** the HTTP cache: nothing expires automatically and you manage versions yourself. Caching strategies are in [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs).

## 6. Origin Private File System (OPFS)

A sandboxed, per-origin file system (not visible to the user), via the File System API.

```js
const root = await navigator.storage.getDirectory();
const file = await root.getFileHandle('export.csv', { create: true });
const w = await file.createWritable();
await w.write(csvText);
await w.close();

// In a worker: very fast synchronous I/O, used by SQLite Wasm
const handle = await file.createSyncAccessHandle();
handle.write(new TextEncoder().encode('row\n'), { at: 0 });
handle.flush(); handle.close();
```

It's supported in all current major browsers, and is the recommended backend for running SQLite in the browser.

## 7. Quotas, eviction and persistence

- All of IndexedDB, Cache API, OPFS (and service worker registrations) share one **per-origin quota**. The quota is a share of the device's free disk space and differs per browser. Web Storage has its own small ~5 MB limit.
- By default storage is **"best-effort"**: under storage pressure the browser may evict the whole origin (least recently used first).
- Ask for **persistent** storage to be exempt from automatic eviction:

```js
const { usage, quota } = await navigator.storage.estimate();
console.log(`Using ${(usage / 1e6).toFixed(1)} MB of ${(quota / 1e6).toFixed(0)} MB`);

if (navigator.storage.persist && !(await navigator.storage.persisted())) {
  const granted = await navigator.storage.persist(); // Chrome auto-decides (e.g. installed/bookmarked/engaged); Firefox may prompt
  console.log(granted ? 'Persistent storage' : 'Best-effort storage');
}
```

- Handle `QuotaExceededError` on writes and have an eviction policy of your own (LRU cleanup of old cached data).
- Users can always clear site data; persistent does not mean permanent.

<Info>
**🆕 2025–2026**

- **Storage Buckets API**: lets one origin create several independently evictable buckets with their own durability/persistence settings, e.g. keep drafts `strict` and persisted while a media cache stays best-effort. Status: **Limited, Chromium only (Chrome 122)**.
  ```js
  const drafts = await navigator.storageBuckets.open('drafts', { durability: 'strict', persisted: true });
  const idb = drafts.indexedDB; // this bucket's own IndexedDB / caches / OPFS
  ```
- **IndexedDB `getAllRecords`** is part of **Interop 2026** (announced 2026-02-12), so expect it to become cross-browser.
- **Web Locks** (`navigator.locks.request`) is Baseline Widely available (since 2024-09): use it to coordinate IndexedDB migrations or sync jobs across tabs.
- Third-party cookie restrictions continue; `Partitioned` (CHIPS) cookies are the supported path for embedded widgets that need state.
</Info>

## 8. Application state: access optimization & memory offloading

From the FE system design outline (*Application State*):

**Search / access optimization**: store client data in a shape that makes reads cheap.
- **Normalize** entities by id (`{ byId: {...}, allIds: [...] }`), as RTK's `createEntityAdapter` does; lookups are O(1) and updates don't duplicate data ([Redux & Redux Toolkit](/Software%20Engineering/Frontend/06%20State%20Management/Redux%20%26%20Redux%20Toolkit)).
- Precompute **indexes** for frequent queries (a `Map` from airline code to flights; an IndexedDB index on `departureTime`).
- Memoize derived data (selectors) rather than recomputing on every render.

**Memory offloading**: don't keep everything in JS heap memory.
- Keep only the visible window in memory/DOM ([Virtualization & Large Lists](/Software%20Engineering/Frontend/10%20Performance/Virtualization%20%26%20Large%20Lists)); page older items out to IndexedDB and reload on scroll-back.
- Move heavy parsing/filtering into a **Web Worker** ([Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs)).
- Store large binary data as `Blob`s in IndexedDB/OPFS and use `URL.createObjectURL` (then `revokeObjectURL`).
- Cap in-memory caches (LRU, or `WeakMap`/`WeakRef` for caches keyed by objects, see [Collections - Map, Set, WeakMap & WeakRef](/Software%20Engineering/Frontend/01%20JavaScript/Collections%20-%20Map%2C%20Set%2C%20WeakMap%20%26%20WeakRef)).

## 9. Security summary

- Never put secrets, tokens or PII in `localStorage`/`sessionStorage`/IndexedDB if XSS is a realistic risk: any injected script can read them.
- Auth session → `HttpOnly; Secure; SameSite=Lax|Strict` cookie, preferably with the `__Host-` prefix, plus CSRF protection for state-changing requests.
- A strong CSP and Trusted Types reduce XSS risk in the first place ([Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)).
- Storage is per origin; subdomains don't share `localStorage`, but they can share cookies via `Domain`.

## Interview questions

<Accordion title="Compare cookies, localStorage and sessionStorage.">
Cookies: ~4 KB, sent automatically to the server with every request, lifetime via Expires/Max-Age, can be HttpOnly/Secure/SameSite. localStorage: ~5 MB, client-only, persists until cleared, sync string API. sessionStorage: same API, scoped to one tab and cleared when it closes.
</Accordion>

<Accordion title="Where should you store an auth token?">
Preferably in an `HttpOnly; Secure; SameSite` cookie (ideally `__Host-`), which JS can't read, plus CSRF defenses. Access tokens can also be kept in memory with a refresh token in an HttpOnly cookie. `localStorage` is readable by any XSS payload.
</Accordion>

<Accordion title="What does SameSite=Lax protect against, and what doesn't it?">
It stops cookies being sent on cross-site subresource requests and cross-site POSTs, which blocks most CSRF. It still sends cookies on top-level GET navigations, so GET endpoints must never change state. It does nothing against XSS.
</Accordion>

<Accordion title="What are CHIPS / Partitioned cookies?">
Third-party cookies stored separately per top-level site, so an embed keeps state on each site it's embedded in without being able to track users across sites.
</Accordion>

<Accordion title="When would you choose IndexedDB over localStorage?">
For large or structured data, binary blobs, querying via indexes, access from workers/service workers, or when a synchronous API would block the main thread (offline-first apps, caches of API data, drafts).
</Accordion>

<Accordion title="How does the browser decide to evict storage? How do you prevent it?">
Origins are best-effort by default and evicted LRU under disk pressure. `navigator.storage.persist()` requests persistent mode; `navigator.storage.estimate()` reports usage/quota.
</Accordion>

<Accordion title="Is sessionStorage shared between tabs?">
No. Each tab has its own; duplicating a tab copies it. localStorage is shared across all tabs of the origin and fires `storage` events in the others.
</Accordion>

## Related
- [Authentication & Token Storage](/Software%20Engineering/Frontend/11%20Security/Authentication%20%26%20Token%20Storage)
- [JWT vs Cookies](/Software%20Engineering/APIs%20%26%20Auth/JWT%20vs%20Cookies)
- [Authentication vs Authorization](/Software%20Engineering/APIs%20%26%20Auth/Authentication%20vs%20Authorization)
- [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)
- [Web Workers, Service Workers & PWAs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Web%20Workers%2C%20Service%20Workers%20%26%20PWAs)
- [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)
- [URL as State](/Software%20Engineering/Frontend/06%20State%20Management/URL%20as%20State)
- [Ultimate NextJs Course](/Software%20Engineering/Frontend/Courses%20%26%20Books/FE%20Masters/Ultimate%20NextJs%20Course) (HTTP state management section)

## Sources
- MDN: [Using HTTP cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies), [Set-Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie), [CHIPS](https://developer.mozilla.org/en-US/docs/Web/Privacy/Privacy_sandbox/Partitioned_cookies), [Web Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API), [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API), [Storage quotas and eviction criteria](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria), [Origin private file system](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system), [Cookie Store API](https://developer.mozilla.org/en-US/docs/Web/API/Cookie_Store_API)
- web.dev: [Storage for the web](https://web.dev/articles/storage-for-the-web), [Persistent storage](https://web.dev/articles/persistent-storage), [SameSite cookies explained](https://web.dev/articles/samesite-cookies-explained)
- Chrome: [Storage Buckets](https://developer.chrome.com/docs/web-platform/storage-buckets), [CHIPS](https://developer.chrome.com/docs/privacy-sandbox/chips/)
- Libraries: [idb](https://github.com/jakearchibald/idb), [Dexie](https://dexie.org/)
- Interop 2026: [web.dev/blog/interop-2026](https://web.dev/blog/interop-2026)
