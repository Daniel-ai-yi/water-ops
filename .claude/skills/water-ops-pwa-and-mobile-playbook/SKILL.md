---
name: water-ops-pwa-and-mobile-playbook
description: >
  Deep PWA and mobile playbook for the Water Ops app. Load this for ANYTHING involving:
  install / "Add to Home Screen"; offline behavior; caching; the service worker (sw.js) and
  its lifecycle; the web app manifest (manifest.webmanifest) or icons; "my update isn't
  showing up" / stale app after deploy when you need the deep lifecycle and deploy-cycle
  detail (for first-stop symptom triage use water-ops-debugging-playbook); bumping the
  cache name; deploying to any live origin
  and the old-service-worker-controls-clients trap; iOS Safari / Android Chrome / desktop
  Chrome-Edge platform quirks; standalone-mode debugging; viewport, safe-area insets, notch
  padding, touch targets, iOS focus zoom; or file:// vs localhost vs HTTPS capability
  differences. This skill holds the deep dives on all of the above.
---

# Water Ops — PWA and Mobile Playbook

Water Ops is a single-file PWA (`index.html`, no build step, no backend) used by water
treatment plant operators for field logging (domain terms → `water-treatment-domain-reference`).
It is installed to phone home screens and used offline. That makes the PWA layer — manifest,
icons, service worker, cache updates — the app's hardest-problem area. This skill is the
deepest reference for it.

**When NOT to use this skill:**

| You need | Go to |
|---|---|
| Quick triage of a generic app bug (copy fails, screen won't switch, label weirdness) | `water-ops-debugging-playbook` (it holds the symptom→check table; it cross-refs back here for SW/cache deep dives) |
| Deploy policy, sign-off rules, non-negotiables | `water-ops-change-control` |
| Hosting/auth roadmap (Azure move, private repo, login screen, SharePoint) | `water-ops-platform-campaign` |
| History of how the SW got to v9 | `water-ops-failure-archaeology` |
| Exact copy-output text formats | `water-ops-run-and-operate` |

**Deploy = copying 6 files to a static host:** `index.html`, `sw.js`, `manifest.webmanifest`,
`icon-192.png`, `icon-512.png`, `apple-touch-icon-180.png`. Nothing else exists. (Locally the
app file is named `index[23].html` — a download-numbering quirk explained in
`water-ops-failure-archaeology`; on any server it must be `index.html`.)

---

## 1. Manifest anatomy — the actual `manifest.webmanifest`, field by field

The deployed manifest (verified on disk 2026-08-10, 523 bytes):

```json
{
  "name": "Water Ops",
  "short_name": "Water Ops",
  "start_url": "./",
  "scope": "./",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#ffffff",
  "theme_color": "#ffffff",
  "icons": [
    { "src": "./icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "./icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    { "src": "./icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

What each field controls (platform behavior notes are platform-documentation knowledge as of
2026-08-10; browsers evolve — re-verify against MDN when it matters):

| Field | Value | What it does, per platform |
|---|---|---|
| `name` | `Water Ops` | Full app name. Android: install banner / splash screen. Desktop Chrome/Edge: install dialog and app-window title. |
| `short_name` | `Water Ops` | Label under the home-screen icon when `name` would be too long. Same string here, so no truncation surprises. |
| `start_url` | `./` | URL opened when the installed icon is tapped. Relative to the manifest's location, so the app works at any path on any static host (including the planned Azure move). Keep it relative — do not hardcode an origin. |
| `scope` | `./` | URL range the installed app "owns." Navigation outside scope opens a browser UI overlay instead of staying in the app window. `./` = everything under the deploy directory. |
| `display` | `standalone` | Installed app runs without browser chrome (no URL bar). This is what makes it feel like a native app — and what hides the console on iOS (see section 4). |
| `orientation` | `portrait` | Android locks the installed app to portrait. iOS ignores this field (platform-doc knowledge, 2026-08-10). Desktop ignores it. |
| `background_color` | `#ffffff` | Splash/launch background before first paint (Android splash screen; desktop window background). |
| `theme_color` | `#ffffff` | Tint for OS chrome around the app (Android status bar, desktop title bar). Duplicated as a `<meta name="theme-color">` in `index.html` head — keep the two in sync. |
| `icons[0]` | 192 `any` | Minimum Android home-screen icon. |
| `icons[1]` | 512 `any` | Splash screen + high-density contexts; 512 `any` is also a Chrome installability requirement. |
| `icons[2]` | 512 `maskable` (reuses `icon-512.png`) | See below. |

**What `maskable` means:** Android launchers crop icons into their own shape (circle,
squircle, rounded square). `purpose: "maskable"` declares "this image tolerates cropping —
all important content sits inside the central ~80% safe zone." This manifest reuses
`icon-512.png` for both `any` and `maskable`, which is fine ONLY as long as the logo stays
comfortably centered. If the icon art ever changes, verify the safe zone locally by
overlaying a circle covering the central 80% of the image (or use a third-party preview
site such as https://maskable.app — dev-preview only, and do not upload sensitive company
brand art to external services), or accept edge-cropping on Android. Icons on disk (verified 2026-08-10): `icon-192.png`
192x192, `icon-512.png` 512x512, `apple-touch-icon-180.png` 180x180, all PNG RGB.

**Why `apple-touch-icon-180.png` is ALSO required:** iOS Safari ignores manifest `icons`
for the home-screen icon (platform-doc knowledge, 2026-08-10 — Apple has partially adopted
the manifest over the years, but the `apple-touch-icon` link is still the reliable path).
Without it, iOS renders a screenshot-thumbnail as the icon. `index.html` provides it in the
head (line 12): `<link rel="apple-touch-icon" href="./apple-touch-icon-180.png">`.
180x180 is the size for current iPhones.

**The head metas in `index.html` (verified lines 4–13, 2026-08-10) — all load-bearing:**

| Line | Tag | Why it's there |
|---|---|---|
| 5 | `<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">` | Mobile scaling; `viewport-fit=cover` lets the page extend into the notch/home-indicator areas so the safe-area CSS in section 6 can manage them. |
| 6 | `<meta name="apple-mobile-web-app-capable" content="yes">` | Legacy iOS signal for standalone (full-screen) mode when launched from the home screen. |
| 7 | `<meta name="mobile-web-app-capable" content="yes">` | Non-Apple counterpart of the same legacy signal (Chrome now prefers the manifest but the meta is harmless belt-and-suspenders). |
| 8 | `<meta name="apple-mobile-web-app-status-bar-style" content="default">` | iOS standalone status-bar style: `default` = normal opaque status bar (matches the white theme). |
| 9 | `<meta name="apple-mobile-web-app-title" content="Water Ops">` | iOS home-screen icon label (iOS ignores manifest `short_name`; without this it uses `<title>`). |
| 10 | `<meta name="theme-color" content="#ffffff">` | Browser UI tint while running in a tab; mirrors the manifest field. |
| 11 | `<link rel="manifest" href="./manifest.webmanifest">` | Wires the manifest in. Without this link the manifest is inert. |
| 12 | `<link rel="apple-touch-icon" href="./apple-touch-icon-180.png">` | iOS home-screen icon (see above). |

**Manifest/icon staleness rule:** installed-app identity (icon, name, colors) is captured at
install time and refreshes lazily or never, depending on platform. After changing the
manifest or icons, expect installed users to need a re-install (remove icon, re-add) to see
the new identity. Do not chase this as a caching bug — it is install-time capture.

---

## 2. Service worker, line by line — canonical `sw.js` (v9)

**CRITICAL DEPLOY FLAG (as of 2026-08-10):** `sw.js` is NOT in the local working folder.
It DOES exist at HEAD of the canonical repo (https://github.com/Daniel-ai-yi/water-ops,
commit `a57840c`), byte-for-byte the v9 content embedded below, which the owner also
supplied directly on 2026-08-10. Any deploy set assembled from the local folder must pull
in `sw.js` explicitly or the deploy ships without it. `index.html` registers it at lines
884–886:

```js
if('serviceWorker' in navigator){
  navigator.serviceWorker.register('./sw.js').catch(function(){});
}
```

That `.catch(function(){})` is a **silent failure by design**: if `sw.js` is missing from a
deploy, registration 404s and NOTHING tells you. The app still works online but has no
offline support and no install-grade caching. Every deploy MUST include `sw.js`. Verify with
`curl` (see Provenance) — never assume.

Canonical `sw.js` v9:

```js
/* Water Ops offline cache */
var CACHE = 'waterops-v9';
var ASSETS = [
  './',
  './index.html',
  './manifest.webmanifest',
  './icon-192.png',
  './icon-512.png',
  './apple-touch-icon-180.png'
];
self.addEventListener('install', function(e){
  e.waitUntil(caches.open(CACHE).then(function(c){ return c.addAll(ASSETS); }).then(function(){ return self.skipWaiting(); }));
});
self.addEventListener('activate', function(e){
  e.waitUntil(caches.keys().then(function(keys){
    return Promise.all(keys.map(function(k){ if(k!==CACHE) return caches.delete(k); }));
  }).then(function(){ return self.clients.claim(); }));
});
self.addEventListener('fetch', function(e){
  if(e.request.method!=='GET') return;
  e.respondWith(
    caches.match(e.request).then(function(hit){
      return hit || fetch(e.request).then(function(res){
        return caches.open(CACHE).then(function(c){ c.put(e.request, res.clone()); return res; });
      }).catch(function(){ return caches.match('./index.html'); });
    })
  );
});
```

Walkthrough:

- **`var CACHE = 'waterops-v9';`** — the versioned cache name. **This string is the app's
  ONLY update mechanism.** Changing it (v9 → v10) is what makes a deploy actually reach
  users. Everything below explains why.
- **`ASSETS`** — the 6 deployable files plus `'./'` (the directory URL, which servers answer
  with `index.html`; both `'./'` and `'./index.html'` are cached because a navigation to the
  origin root requests `'./'`, not the filename).
- **`install`** — fires when the browser sees a NEW (byte-different) `sw.js`. Opens the
  versioned cache, `cache.addAll(ASSETS)` pre-caches all 6 assets atomically (any single
  404 rejects the whole install — another reason a missing file breaks deploys), then
  `skipWaiting()` tells the new worker not to sit in the "waiting" state until all tabs
  close, but to activate immediately.
- **`activate`** — fires when the new worker takes over. Enumerates ALL cache names on the
  origin and **deletes every cache whose name is not the current `CACHE`** — this is how
  `waterops-v8` (or any older cache left by a previous deploy — section 3) gets purged. Then
  `clients.claim()` seizes control of already-open pages without waiting for a reload.
- **`fetch`** — the runtime strategy:
  - `if(e.request.method!=='GET') return;` — GET-only guard; POSTs etc. pass through
    untouched (Cache API can't store them anyway).
  - **Cache-first:** `caches.match(e.request)` — if the response is in the cache, serve it
    and NEVER touch the network. This is what makes the app instant and fully offline.
  - **Network fallback with back-fill:** on cache miss, fetch from network and `c.put` a
    clone into the cache, so it hits next time.
  - **Offline navigation fallback:** if the network fetch also fails (offline, cache miss),
    serve `./index.html` — so any navigation while offline still lands on the app.

**The consequence you must internalize — why cache-first means "no updates without a
version bump":**

1. User's device has `waterops-v9` populated. Every request for `./index.html` is served
   from cache. Deploying a new `index.html` alone changes NOTHING for existing users, ever.
2. The only escape hatch: the browser periodically re-fetches **`sw.js` itself** (bypassing
   the SW, on navigation, at most every 24h per spec — near-immediate if the host serves
   sw.js with no-cache or short max-age headers, but budget for up to 24h. Platform-doc
   knowledge, 2026-08-10).
3. If the fetched `sw.js` is **byte-identical** to the installed one, nothing happens. If it
   differs — e.g. `waterops-v9` → `waterops-v10` — the new worker installs (fresh cache from
   the network, so it captures the new `index.html`), skipWaiting → activates, deletes the
   old cache, claims clients. The page a user is currently viewing may still be the old HTML
   until the next launch/reload, but the NEXT open shows the new app.

**Therefore, the iron discipline: bump `waterops-vN` on EVERY deploy that changes any of the
6 files.** Deploying content without bumping the cache name is a silent no-op for every
existing installed user. History proves the discipline: the repo's one-month history walks
through nine cache-name versions (eight bumps), `waterops-v1` through the current
`waterops-v9` (as of 2026-08-10) — commit-by-commit ledger and narrative owned by
`water-ops-failure-archaeology` (E4), matching the owner's statement that "updating cache
was a problem". Change-control lists the bump as a non-negotiable.

---

## 3. THE STALE-ORIGIN TRAP (origin-agnostic)

**Where is the app actually live?** As of 2026-08-10: GitHub Pages at
`https://daniel-ai-yi.github.io/water-ops/` (owner-confirmed; verified serving
`waterops-v9`, current with repo HEAD). The canonical repo is
https://github.com/Daniel-ai-yi/water-ops, and the owner plans a move to Azure hosting
(`water-ops-platform-campaign`) — after any move this section applies doubly to the OLD origin.
**Never assume the live origin's state — verify it (`curl -s <deployed-origin>/sw.js | grep CACHE`).** Wherever the app is (or was ever)
served, this trap applies:

**The trap:** any device that ever visited the deployed origin has that deploy's service
worker installed and controlling the origin, serving that deploy's cache-first. If you then
push a new `index.html` but (a) forget `sw.js` (remember: it is absent from the local
working folder — it lives in the repo), or (b) ship an `sw.js` byte-identical to the one
already deployed, existing users are permanently pinned to the old app. The old SW answers
every request from its old cache and never consults the network for cached URLs.

**Why the escape works:** every historical version of this app's `sw.js` (v1 through v9,
per the repo history) has the same lifecycle — skipWaiting + clients.claim +
delete-non-matching-caches — and `sw.js` fetches always bypass the SW's cache-first logic
(the browser's SW update check goes to the network per the HTTP rules in section 2). So a
new `sw.js` with a new cache name WILL be picked up, and its activate step deletes every
old cache regardless of what that cache was called. The update propagates — on a delay you
must be honest about.

### Safe-deploy runbook (any origin)

Below, `<deployed-origin>` = the HTTPS origin actually serving the app. As of 2026-08-10
that is GitHub Pages: `https://daniel-ai-yi.github.io/water-ops` (owner-confirmed, verified
serving `waterops-v9`); an Azure move is planned, so re-confirm before deploying — do not guess.

1. **Check the origin's current state FIRST:**
   ```bash
   curl -s https://<deployed-origin>/sw.js | grep "var CACHE"
   # Note the currently-deployed cache name. 404/empty = no SW deployed there (or wrong origin).
   ```
2. Assemble all 6 files. Confirm the new `sw.js` cache name is `waterops-vN` with N bumped
   PAST both the repo's HEAD version (`waterops-v9` as of 2026-08-10) and whatever step 1
   showed. Any byte-different name triggers the update, but stay on the monotonic
   `waterops-vN` series.
3. Sanity-check the bundle locally before pushing:
   ```bash
   ls index.html sw.js manifest.webmanifest icon-192.png icon-512.png apple-touch-icon-180.png
   grep "var CACHE" sw.js        # expect: var CACHE = 'waterops-vN';
   grep -c "serviceWorker.register" index.html   # expect: 1
   ```
4. Deploy (push to the repo the host builds from, or copy to the static host). Deploy is
   atomic in intent: all 6 files together, never `index.html` alone.
5. Verify the origin is serving the new worker:
   ```bash
   curl -s https://<deployed-origin>/sw.js | grep "var CACHE"
   # MUST print the new waterops-vN — if it still prints the step-1 name, the host hasn't rebuilt; wait and re-curl
   ```
6. Verify on a device/browser that previously used the app from this origin: open it, then
   in desktop Chrome/Edge DevTools → **Application → Service Workers**: you should see the
   new worker install and activate (with skipWaiting there is no lingering "waiting"
   state). Under **Application → Cache Storage**: new `waterops-vN` present, the old cache
   name from step 1 GONE (deleted by the new activate). On phones, use remote inspection
   (section 4).
7. Reload once more and confirm the visible change you shipped actually renders.

### Honest propagation timeline for existing users

- **Visit 1 after deploy:** the old SW serves the old app from its cache. In parallel, the
  browser's SW update check fetches the new `sw.js`, sees a byte difference, installs the
  new worker (which caches the NEW files from the network), activates (deleting the old
  cache), and claims the page. The user is likely still LOOKING at the old app this whole
  time.
- **Visit 2 (or a reload):** the new worker serves `waterops-vN` — the user sees the new
  version.
- So the user-facing instruction is simply: **"open it, close it, open it again"** (or pull
  a reload). The SW update check runs on navigation but can be deferred up to ~24h by HTTP
  caching of sw.js (host-dependent; many static hosts serve short max-age making it
  near-immediate — platform-doc knowledge, 2026-08-10). Tell users "within a day" to be
  safe.
- **Nuclear option (last resort only):** browser Settings → "Clear site data" for the
  origin (or DevTools → Application → Storage → Clear site data). This unregisters the SW
  and wipes caches, forcing a fully fresh load. It works, but it also proves you did
  something wrong in the deploy — the runbook above should never need it. On iOS the
  equivalent is Settings → Safari → Advanced → Website Data → delete the origin, or remove
  and re-add the home-screen app.

### Origin migration / decommissioning an old origin (self-destructing SW)

Referenced by `water-ops-platform-campaign` Stage B (the planned Azure move). Everything
in section 3 above covers staleness on the SAME origin; moving to a NEW origin is a
different problem:

- **Installed home-screen apps do NOT follow an origin move.** Each installed app is
  pinned to the origin it was installed from; its cache-first SW will keep serving the old
  app from the old origin's cache indefinitely (and even fully offline). There is no
  redirect that reaches an installed, offline-capable PWA reliably.
- **User comms are mandatory:** tell users to delete the old icon and install fresh from
  the new URL. This is the only reliable migration path for installed apps.
- **If you still control the old origin, deploy a self-destructing SW there** as the final
  deploy: it unregisters itself, wipes every cache, and reloads open clients, so the old
  origin at least stops serving the stale app (users online at the old origin then see
  whatever the old origin serves — ideally a static "we moved" page). Pattern (matches the
  app's ES5 style; adapt deliberately, and test it on a staging origin first):

  ```js
  /* self-destruct: final sw.js for a decommissioned origin */
  self.addEventListener('install', function(){ self.skipWaiting(); });
  self.addEventListener('activate', function(e){
    e.waitUntil(
      caches.keys().then(function(keys){
        return Promise.all(keys.map(function(k){ return caches.delete(k); }));
      }).then(function(){ return self.registration.unregister(); })
        .then(function(){ return self.clients.matchAll({type:'window'}); })
        .then(function(list){ list.forEach(function(c){ c.navigate(c.url); }); })
    );
  });
  ```

  It rides the same update mechanism as any bump: the browser fetches the byte-different
  `sw.js` (subject to the same up-to-24h delay as section 2), installs, activates, and the
  destruct runs. Devices that never come online against the old origin never receive it —
  hence the user comms above.

---

## 4. Install and update behavior per platform

Owner targets ALL of: iOS, Android, and desktop/PC browsers (owner decision, 2026-08-10).
All platform behaviors below are platform-documentation knowledge as of 2026-08-10 unless
tied to the app files; iOS especially changes across OS versions — re-verify on the current
OS before debugging "weird device behavior."

### iOS Safari

- **Install:** Safari → Share button → **Add to Home Screen**. There is no install prompt;
  users must be told the gesture. The icon comes from `apple-touch-icon-180.png`, the label
  from `apple-mobile-web-app-title` ("Water Ops").
- **Standalone quirks:** launched from the icon, the app runs standalone — no URL bar, no
  reload button, and **no visible DevTools console**. `console.log` goes nowhere you can
  see on-device.
- **Debugging an installed iOS app:** connect the iPhone to a Mac by cable → iPhone
  Settings → Safari (or Safari → Advanced on older iOS) → enable **Web Inspector** → Mac
  Safari → **Develop menu → [device name] → Water Ops**. Full inspector against the
  standalone app, including Console and Storage.
- **Eviction:** iOS may evict service workers and caches for web apps unused for several
  weeks (part of Safari's storage-pressure/ITP behavior). Symptom: an operator returns
  after a month and the app loads from network or shows first-visit behavior. **This is
  documented platform behavior to watch for, NOT an app bug** — the app re-registers and
  re-caches on next online launch. Only escalate if it happens with regular use.
- iOS ignores manifest `orientation`; portrait lock is not guaranteed on iPad.

### Android Chrome

- **Install:** Chrome shows an install affordance (menu → "Install app" / "Add to Home
  screen", sometimes a banner) when the installability criteria hold: served over HTTPS, a
  linked manifest with `name`, `start_url`, `display` standalone-or-better, and at least a
  192px and a 512px icon — Water Ops's manifest satisfies all of these — plus a registered
  service worker on most Chrome versions.
- **Update/identity:** Android respects manifest `orientation: portrait`, maskable icons,
  splash from `background_color` + 512 icon.
- **Debugging an installed Android app:** phone in USB-debugging mode → desktop Chrome →
  `chrome://inspect#devices` → the Water Ops WebView/tab appears → **inspect** gives full
  DevTools including Application tab (verify cache names remotely, exactly like desktop).

### Desktop Chrome / Edge (owner targets PC too, as of 2026-08-10)

- **Install:** icon in the address bar ("Install Water Ops") or menu → "Install app" /
  Edge: "Apps → Install this site as an app". Runs in its own window, standalone, title
  from manifest `name`.
- Desktop ignores `orientation`; the portrait-shaped, max-width-600px layout renders as a
  centered column — usable, but the desktop-UX polish is roadmap work
  (`water-ops-platform-campaign`).
- Desktop DevTools is the primary place to watch a SW update land (section 5 runbook).

### Context × capability test matrix

"Secure context" = HTTPS, or localhost (which browsers treat as secure for development).
Service workers and the async clipboard API (`navigator.clipboard`) exist ONLY in secure
contexts.

| Capability | `file://` (double-click the HTML) | `http://localhost:8000` | HTTPS origin (production host) |
|---|---|---|---|
| Page loads, tools render | YES (JS runs; noscript warns about file previews, not file://) | YES | YES |
| Service worker registers | **NO** — never; silent `.catch` hides it | YES (localhost = secure context) | YES |
| Offline after first visit | NO | YES (while server runs) | YES |
| `navigator.clipboard` (async copy) | Not available in mainstream browsers on `file://` — treat as absent (the copy button falls to the `fallbackCopy` execCommand path); do not validate clipboard behavior here | YES | YES |
| Install to home screen / desktop | NO | Partial (Chrome allows localhost installs for dev) | YES — the real thing |
| Good for | Pure UI/format eyeballing | Full PWA dev loop (section 5) | Production + final verification |

**Security tie-in (owner directive 2026-08-10: security is a must):** HTTPS on the deploy
origin is non-negotiable, and not only because it gates SW and clipboard — this app carries
company (NALCO/Ecolab) operational data, and HTTPS protects it in transit. GitHub Pages and
Azure Static Web Apps both give HTTPS by default; never stand up a plain-HTTP deploy.
Broader confidentiality roadmap (private repo, Azure, login) → `water-ops-platform-campaign`.

---

## 5. End-to-end update-cycle test runbook

Run this full cycle before calling ANY deploy done. It exercises the exact mechanism users
depend on. Prereq: a working directory containing all 6 files, with the app file named
`index.html` and `sw.js` present (create it from the canonical v9 in section 2 if absent).

```bash
# 1. Serve locally (localhost = secure context, SW works)
cd <deploy-staging-dir>
python3 -m http.server 8000
```

2. Open `http://localhost:8000/` in Chrome/Edge. Open DevTools → **Application** tab:
   - **Service Workers**: `sw.js` shows "activated and is running".
     (If nothing appears: sw.js is missing or errored — the silent `.catch` in index.html
     will not tell you. Check the Network tab for a 404 on sw.js.)
   - **Cache Storage**: exactly one cache, `waterops-v9` (or current N), containing the 6
     assets.
3. Prove cache-first: DevTools → Network tab → check **Offline** → reload. The app must
   load fully. Uncheck Offline.
4. Simulate a deploy:
   - Make a visible change in `index.html` (e.g. temporarily edit the `.sub` header text).
   - Bump the cache name in `sw.js`: `waterops-v9` → `waterops-v10`.
5. Reload the page ONCE. In Application → Service Workers, watch the new worker appear and
   — because of `skipWaiting` — go straight to "activated" (no stuck "waiting" entry). In
   Cache Storage, `waterops-v10` appears and `waterops-v9` is deleted.
6. Reload a SECOND time. The visible change must now render (first reload was likely still
   served by the old worker before claim — this two-reload rhythm is exactly what real
   users experience, see section 3 timeline).
7. Negative control: revert the visible change but do NOT bump the cache name; reload
   twice. The change must NOT appear on the second reload if you re-apply it without a
   bump — this proves to you, viscerally, why the bump is mandatory. Clean up: restore the
   file and set the cache name to the final N you will deploy.
8. **Once per deploy, repeat the happy path on a real phone** (Android via
   `chrome://inspect`, or iOS via Mac Safari remote inspect — section 4): install/open the
   old version, deploy, reopen twice, confirm the change and the cache name. A deploy is
   not done until one real device has shown the update landing.

Post-deploy origin check (also step 5 of the section 3 runbook; `<deployed-origin>` = the
confirmed live origin):

```bash
curl -s https://<deployed-origin>/sw.js | grep "var CACHE"
```

---

## 6. Viewport, safe areas, and touch — the actual CSS

All line numbers verified against the 890-line `index.html` on 2026-08-10 (re-verify before
citing; the file evolves).

**Safe areas.** "Safe area insets" are the regions iOS/Android reserve for the notch,
rounded corners, and home indicator; `env(safe-area-inset-*)` reads them in CSS, and only
has nonzero values because the viewport meta (line 5) sets `viewport-fit=cover` (otherwise
the OS letterboxes the page and the env() values are 0). The app's actual usage:

| Line | Rule | Purpose |
|---|---|---|
| 27 | `body{ padding-top:max(16px,env(safe-area-inset-top)); }` | Content clears the notch/status bar; `max()` keeps a 16px minimum on notchless devices. |
| 41 | `#screen-home{padding-bottom:calc(env(safe-area-inset-bottom) + 24px);}` | Home menu clears the home indicator. |
| 57 | `.tool-wrap{padding-bottom:calc(var(--bar) + env(safe-area-inset-bottom) + 12px);}` | Tool screens clear BOTH the fixed action bar (`--bar:74px`, line 20) and the home indicator, so the output preview is never hidden behind the bar. |
| 136 | `.actionbar{ padding:10px 16px calc(10px + env(safe-area-inset-bottom)); }` | The fixed bottom action bar itself extends its padding into the home-indicator zone instead of colliding with it. |

**Touch behavior:**

| Line(s) | Rule | Purpose |
|---|---|---|
| 26 | `-webkit-tap-highlight-color:transparent` on `body` | Kills the gray tap flash on iOS/Android; the app supplies its own `:active` states (`opacity:.85`, `background:var(--fill)`). |
| 83, 95, 157 | `min-height:52px` on `.seg label`, `.inp`, `.slvl-btn`; 54px on `.copy`/`.tg`/`.clear` (144/147/149) | Touch targets at or above the ~48px comfortable-tap minimum — operators use this with wet or gloved hands. |
| 46, 86, 117, 160 | `user-select:none` on `.menu-card`, `.seg label`, `.lt`, `.slvl-btn` | Prevents long-press text-selection/callout on tappable controls (the output `pre.out` deliberately stays selectable — operators copy from it). |
| 95 | `.inp{ font-size:18px; }` | **iOS auto-zooms the page when focusing any input whose font-size is under 16px** (platform behavior, 2026-08-10). 18px keeps focus zoom off. Never shrink an input/select below 16px. |
| 23 | `html{-webkit-text-size-adjust:100%;}` | Stops iOS from inflating text on orientation change. |

Rules when touching this layer:

- Any NEW fixed/sticky element at a screen edge must add the matching
  `env(safe-area-inset-*)` padding.
- Any NEW input/select gets `class="inp"` or at minimum `font-size:16px+` and
  `min-height:52px`.
- Test safe areas on a real notched iPhone or DevTools device emulation (iPhone-with-notch
  preset) — desktop windows report all insets as 0, so regressions are invisible there.

---

## Provenance and maintenance

Ground truth compiled 2026-08-10 and corrected the same day against: `index.html` (890
lines, locally `index[23].html`), `manifest.webmanifest`, the icon PNGs, and the canonical
repo **https://github.com/Daniel-ai-yi/water-ops** at HEAD `a57840c` (2026-08-05), whose
`index.html` is byte-identical to the local working file and whose `sw.js` is the v9
embedded in section 2 (owner-confirmed canonical repo, 2026-08-10; the live deployed
origin remains unverified). Platform-behavior claims not derivable from the repo are
labeled "platform-documentation knowledge, 2026-08-10" inline.

One-line re-verification commands (run before trusting drift-prone claims; `<clone>` = a
read-only clone of the repo, `<deployed-origin>` = the owner-confirmed live origin —
`https://daniel-ai-yi.github.io/water-ops` as of 2026-08-10, Azure move planned):

```bash
# SW registration still present, with silent catch (glob matches the local download-named copy or a checkout's index.html):
grep -n "serviceWorker.register" index*.html
# Head metas + manifest/icon links (expect lines ~4-13):
sed -n '4,13p' index*.html
# Manifest fields:
cat manifest.webmanifest
# Icon dimensions:
file icon-192.png icon-512.png apple-touch-icon-180.png
# Repo HEAD sw.js cache name (expect waterops-v9 or later):
git -C <clone> show HEAD:sw.js | grep "var CACHE"
# The v1->v9 bump-per-deploy history (9 sw.js-touching commits, monotonic):
git -C <clone> log --reverse --oneline -- sw.js
# Live origin's deployed SW cache name (must match repo HEAD after any deploy):
curl -s https://<deployed-origin>/sw.js | grep "var CACHE"
# Local working-folder sw.js (no output = still absent locally; pull it from the repo for any deploy set):
grep "var CACHE" sw.js 2>/dev/null || echo "sw.js absent from local working folder (lives in repo)"
# Safe-area / touch CSS anchors:
grep -n "safe-area-inset\|tap-highlight\|font-size:18px\|min-height:52px" index*.html
```

If any command's output contradicts this file, update the relevant section and re-stamp the
date.
