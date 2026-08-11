---
name: water-ops-debugging-playbook
description: >
  Symptom-to-cause triage playbook for the Water Ops PWA (single-file index.html,
  water treatment plant field-logging app). Load this when something is BROKEN or
  MISBEHAVING: users see a stale/old version after a deploy (FIRST STOP for
  triage — deep SW/deploy mechanics live in water-ops-pwa-and-mobile-playbook),
  the Copy button does
  nothing or copies the wrong text, the copy label changes unexpectedly, screens
  are blank or navigation acts oddly, the browser back button exits the app,
  offline mode doesn't work, the service worker never registers, iOS Safari or
  home-screen (standalone) quirks, number-input problems, the time dropdown
  selects a surprising slot, or entered data disappears after a reload. Contains
  a triage table with discriminating checks, per-symptom runbooks, and how to get
  a console on desktop, iOS, and Android.
---

# Water Ops Debugging Playbook

Symptom-first triage for the Water Ops app. Water Ops is a single-file PWA
(`index.html`, no build step, no framework, no backend, no tests) used by water
treatment plant operators to format field readings into fixed plain-text blocks
for copying into the plant's logging channel. For domain terms (EQ tank,
Lamella, Parshall flume, sludge blanket...), see `water-treatment-domain-reference`.

**How to use this file:** find your symptom in the triage table, run the
discriminating check (a check whose result separates this cause from the
look-alikes), then follow the numbered runbook.

**When NOT to use this skill:**

| You are trying to... | Use instead |
|---|---|
| Plan or make a code change safely | `water-ops-change-control` |
| Deep-dive PWA install/update mechanics, sw.js line-by-line, iOS vs Android install differences | `water-ops-pwa-and-mobile-playbook` |
| Understand how a past failure happened (git history, the upload workflow and the `index[23].html` naming) | `water-ops-failure-archaeology` |
| Learn the intended architecture and invariants | `water-ops-architecture-contract` |
| Verify output formats character-for-character | `water-ops-run-and-operate` (owns the output spec) |

All line numbers below were verified against the app file on 2026-08-10 and are
hints, not facts — re-verify with the grep commands in "Provenance and
maintenance" before trusting them in a changed file.

---

## Triage table

| # | Symptom | Likely cause | Discriminating check | Fix / owning skill |
|---|---|---|---|---|
| 0 | App opens on a login screen / "preview mode" / "awaiting admin approval" (since 2026-08-10) | NOT a bug — the initial screen is now the auth gate; non-Azure hosts show labeled preview mode with a Continue button | Console errors? A truly BLANK login screen (no message after load) means `authInit` threw before setting a state | Expected behavior: `water-ops-auth-activation`; blank-screen case: check console, `authInit` in index.html |
| 1 | Deployed a new version; users still see the old app | Service worker (SW) cache-first staleness; cache name in sw.js not bumped | DevTools → Application → Service Workers: is there a "waiting" worker? Application → Cache Storage: is the cache name still the old `waterops-vN`? | Bump `CACHE` in sw.js, redeploy (runbook 1); deep dive: `water-ops-pwa-and-mobile-playbook` |
| 2 | Copy button does nothing (no "Copied!" flash) | `getPlainText()` returned null: no tank selected (Quality Sampling) or no valve selected (Sludge Blanket) — handler silently returns | Is a tank/valve actually selected on the current screen? Check BEFORE suspecting clipboard | Select the missing precondition (runbook 2) |
| 3 | Copy flashes "Copied!" but clipboard is empty/stale | Insecure context: `navigator.clipboard` absent on `file://` or `http://` LAN IP, falls to deprecated `execCommand` fallback which can fail silently | In console: `!!navigator.clipboard` → `false` means insecure context | Serve over HTTPS or `http://localhost` (runbook 2) |
| 4 | Copy button label reads "Copy entry" instead of "Copy" | Known cosmetic bug: HTML starts the button as "Copy" (line ~453) but `flashCopied` restores it to "Copy entry" (line ~846) | Reload the page: label is "Copy" until the first copy, then permanently "Copy entry" | Not a regression; safe to fix (runbook 3) |
| 5 | Quality Sampling preview shows X but copied text differs | `sRender` and `getPlainText` duplicate the line-building logic (trap owned by `water-ops-architecture-contract` W1); someone edited one and not the other | Diff the two code paths (runbook 4) | Mirror the change in both |
| 6 | Sludge Blanket preview shows psi reading but Copy does nothing | `sbRender` shows output when a valve OR psi is entered (~802), but `getPlainText` returns null when no valve is selected (~838) | Enter psi only, no valve: preview populates, Copy is a no-op | Select a valve; this asymmetry is verified real behavior (runbook 2, step 2; edge-behavior spec owned by `water-ops-run-and-operate` §4) |
| 7 | Old entries still visible after switching tools | Not a bug: `showScreen` only toggles CSS `.active`; screen state persists by design | Switch away and back: values intact — that is intended | Use the Clear button per screen (runbook 5) |
| 8 | Browser back button exits the app instead of going to menu | No History API integration exists — known limitation | Search index.html for `pushState`/`popstate`: zero hits | Use the on-screen "‹ Menu" button; a fix is a change → `water-ops-change-control` |
| 9 | No offline support / SW never appears in DevTools | sw.js missing from deploy (registration failure is silently swallowed, ~885) OR page opened via `file://` (SW needs HTTPS/localhost) | DevTools → Application → Service Workers empty + fetch `./sw.js` in the browser: 404 vs loads | Deploy sw.js / serve over HTTPS (runbook 6) |
| 10 | Time dropdown pre-selects a "wrong" or future time | Not a bug: `buildTimeOptions` (~491–503) rounds to the nearest half-hour — minutes ≥ 45 round UP to the next hour and can wrap past midnight | At 09:50, open any 30-min tool: dropdown shows 10:00 | Document, don't "fix" blind (runbook 7) |
| 11 | All data gone after reload/relaunch | No persistence exists, by design — no localStorage/sessionStorage/IndexedDB anywhere | Search index.html for `localStorage`: zero hits | Working as designed; see `water-ops-architecture-contract` (runbook 8) |
| 12 | iOS home-screen app shows old icon/name or behaves oddly | iOS caches manifest/icon aggressively; standalone-mode quirks | Compare installed icon/name vs manifest.webmanifest | Procedural steps in runbook 9; deep dive: `water-ops-pwa-and-mobile-playbook` |

---

## Runbook 1 — "I deployed but users still see the old app"

This is the app's most important failure mode. The SW (service worker — a
background script the browser installs that intercepts network requests) in
sw.js is **cache-first**: every GET is answered from the named cache if present,
and the network is only consulted on a cache miss. A stale SW therefore serves
the old app **forever** until a *new* SW activates. The cache-name bump is this
app's only update mechanism.

**Verified real-world evidence (as of 2026-08-10):** the repo's one-month
history shows nine cache-name versions (eight bumps), `waterops-v1` through
`waterops-v9` (current) — commit-by-commit ledger and narrative owned by
`water-ops-failure-archaeology` (E4). Update-staleness was this project's #1
recurring pain, and the cache-name bump was the recurring cure. As of 2026-08-10
the deployed origin is GitHub Pages at `https://daniel-ai-yi.github.io/water-ops/`
(serving `waterops-v9`); an Azure move is planned, so keep every check
origin-agnostic (step 3 below) and re-confirm the origin before trusting it. The end-to-end recovery procedure is owned by
`water-ops-pwa-and-mobile-playbook` — this runbook stays procedural.

**Discriminating checks (run in order):**

1. On an affected browser: DevTools → Application → Service Workers.
   - Is there a worker in **"waiting"** state? Then a new SW arrived but has not
     activated (should not normally happen here — this sw.js calls
     `skipWaiting()` — so a waiting worker suggests the new sw.js failed during
     install, e.g. one of its `ASSETS` 404'd and `cache.addAll` rejected).
   - Check the SW script's source (click the sw.js link): what is `CACHE` set to?
2. DevTools → Application → Cache Storage. A stale `waterops-vN` still present
   and being served → cache name was not bumped, or the new sw.js never
   installed.
3. Confirm in the deployed source, origin-agnostically (substitute whatever
   origin you actually deployed to — do not assume one):

   ```bash
   curl -s https://<deployed-origin>/sw.js | grep CACHE
   ```

   Compare against the `CACHE` value in the sw.js you meant to deploy. If the
   deployed value equals what's already cached on user devices, the bump was
   skipped — that is your root cause.
4. Rule out the trivial case: hard-reload (Cmd/Ctrl+Shift+R) in a fresh
   incognito window (no SW controls a first visit until registration completes).
   If incognito shows the NEW app, the problem is definitely SW staleness, not
   the deploy itself.

**Fix path:**

1. In sw.js, bump the cache name (e.g. `waterops-v9` → `waterops-v10`). Any byte
   difference in sw.js triggers the browser's SW update check, but the cache-name
   change is what makes activation actually delete the stale cache.
2. Redeploy **all** files including sw.js.
3. What happens next in THIS sw.js (v9 canonical, 2026-08-10): the new SW
   installs, activates immediately (`skipWaiting`), deletes old caches, and
   claims open clients (`clients.claim`) — but open pages still show the old
   DOM until the next reload. Full lifecycle walkthrough and propagation
   timeline: `water-ops-pwa-and-mobile-playbook` §2–3.
4. Tell affected users: close the tab/app fully and reopen (twice if unsure —
   first launch swaps the SW, second is guaranteed fresh).
5. Add "bump SW cache name" to every deploy — `water-ops-change-control` lists
   it as a non-negotiable.

---

## Runbook 2 — "Copy button does nothing / copies nothing"

Three distinct causes. Check them in this order — precondition failures are far
more common than clipboard failures.

**Step 1 — precondition check (silent no-op by design).** The copy handler
(~858) calls `getPlainText()` (~821) and silently returns when it is null:

- Quality Sampling screen: null until a **tank** is selected (`s-tank`).
- Sludge Blanket screen: null until at least one **valve** (1–4) is selected
  (~838) — even if psi is entered and the preview shows text (see triage row 6;
  this preview/copy asymmetry is verified real behavior).
- EQ Log and Parshall Flume never return null (they always emit at least the
  Time line).

Discriminating check: does clicking Copy flash "Copied!"? **No flash at all** =
null text = precondition problem. Flash but bad clipboard = go to step 2.

**Step 2 — secure-context check.** `navigator.clipboard` exists only in secure
contexts: HTTPS or `http://localhost`. On `file://` and on `http://<LAN-IP>`
(common when testing from a phone against a laptop server) the app falls to
`fallbackCopy` (~849): a hidden textarea + `document.execCommand('copy')` —
deprecated, and it calls `flashCopied()` unconditionally even when the copy
threw, so "Copied!" can flash with nothing copied.

Discriminating check, in the console on the affected device:

```js
!!navigator.clipboard            // false → insecure context, fallback path in use
window.isSecureContext           // false confirms it
location.protocol                // "file:" or "http:" on a LAN IP = the culprit
```

Fix: test over `http://localhost` (e.g. `python3 -m http.server`) on desktop, or
HTTPS for phones. See `water-ops-run-and-operate` for local-serving options.

**Step 3 — permissions.** Rare: some browsers gate `clipboard.writeText` behind
a permission or require a user gesture. The handler already runs in a click
gesture, and on rejection it falls back to `fallbackCopy`, so this usually
self-heals; check the console for a `NotAllowedError` if steps 1–2 pass.

---

## Runbook 3 — Copy label flips "Copy" → "Copy entry" permanently

**Known cosmetic bug, verified 2026-08-10. Not a regression — do not bisect.**

- The button's HTML starts it as `Copy` (line ~453:
  `<button class="copy" id="copyBtn" type="button">Copy</button>`).
- `flashCopied` (~844) sets it to `Copied!`, then after 1.8 s restores it to
  `Copy entry` (line ~846) — a label that never appeared in the HTML.

So after the first successful copy of a session the label is permanently
"Copy entry" until reload. Discriminating check: reload → label is "Copy";
copy once → label ends as "Copy entry".

Safe to fix (make the two strings agree; which one wins is an owner/UX call).
Route the change through `water-ops-change-control`.

---

## Runbook 4 — Quality Sampling preview differs from copied text

Root cause: the sample screen's line-building logic exists **twice** —
`sRender` writes the on-screen preview, and `getPlainText`'s `screen-sample`
branch rebuilds the same lines independently for the clipboard. A change
applied to one and not the other makes preview and copy diverge. (The other
three tools share one builder — `lBuild`/`fBuild`/`sbBuild` — and cannot drift
this way.) The authoritative line-level detail — current line pairs and the
intentional `esc()`/`.trimEnd()` differences — is owned by
`water-ops-architecture-contract` (weak point W1).

**Discriminating check:** locate both blocks, then read them side by side and
diff the logic:

```bash
grep -n "lines=\['Time - '+tDisp, t.header" index*.html   # expect 2 hits: the sRender and getPlainText builders
# then sed -n '<start>,<end>p' around each hit to compare the surrounding blocks
```

Both must: build `['Time - HH:MM', tankHeader, '']`, then append
`SF[id].out + ' - ' + sFmtValue(id)` for each non-null field, in `ST[tank].fields`
order — differing only in the intentional ways W1 enumerates.

**Fix:** mirror the missing change into the other block, then hand-verify the
copied text against the output spec in `water-ops-run-and-operate`. The lasting
fix (extract a shared `sBuild()`) is a refactor — see
`water-ops-architecture-contract` (owns this trap) and route through
`water-ops-change-control`.

---

## Runbook 5 — Screen-state and navigation "bugs"

How navigation actually works (verify at ~477–486):

- `showScreen(id)` removes `.active` from every `.screen`, adds it to the
  target, hides the action bar only when `id === 'screen-home'`, and scrolls to
  top. That is the ENTIRE navigation system — CSS `display:none/block` toggling.
- `showScreen` and `goHome` are assigned onto `window` because the menu cards
  and back buttons use inline `onclick` attributes.

Consequences that look like bugs but are design:

| Report | Reality |
|---|---|
| "Switching tools didn't clear my entries" | Intended. DOM is never rebuilt on navigation; per-tool state (`sVals`, input values, `sbSel`...) persists until Clear or reload. Operators rely on this to hop between tools mid-round. |
| "Back button exits the whole app" | True and known. There is zero History API usage (`pushState`/`popstate` absent from index.html), so browser/OS back has nothing to pop and leaves the page. In installed (standalone) mode on Android, the system back gesture closes the app. Honest limitation — fixing it is a feature change (`water-ops-change-control`). |
| "Action bar visible on the menu" / "missing on a tool" | Should be impossible via `showScreen`; if seen, someone toggled `.active`/`.hidden` outside `showScreen`. Check for stray manual class manipulation in recent edits. |

Real screen bugs to check for after edits: a new screen div missing the
`screen` class (never hides), or a menu card's `onclick` pointing at a
nonexistent id (`$(id).classList` throws — check the console).

---

## Runbook 6 — "SW never registers / no offline"

Registration (~884–886) is:

```js
navigator.serviceWorker.register('./sw.js').catch(function(){});
```

The `.catch(function(){})` swallows every failure **silently** — no console
error, no UI hint. So "no offline" gives zero signal by default. Two dominant
causes:

**Cause A — sw.js missing from the deploy.** sw.js is a separate file and (as of
2026-08-10) is not even present in the local working directory — it is easy to
forget when copying files to the host. Discriminating check: open
`https://<origin>/sw.js` directly. A 404 (or the host's HTML 404 page) = this
cause. Fix: deploy sw.js alongside index.html (canonical v9 content is embedded
in `water-ops-pwa-and-mobile-playbook`), bump the cache name, redeploy.

**Cause B — insecure context.** Service workers require HTTPS or localhost.
`file://` and `http://<LAN-IP>` never register. Discriminating check: DevTools
console →

```js
'serviceWorker' in navigator   // true, but…
window.isSecureContext         // false → SW can never register here
```

Fix: `python3 -m http.server` + `http://localhost:8000` for desktop testing;
HTTPS hosting for real devices.

**Confirm recovery:** DevTools → Application → Service Workers shows an
"activated and is running" worker for the origin; Cache Storage shows the
current `waterops-vN`; then toggle DevTools Network → Offline and reload — the
app must still load.

---

## Runbook 7 — Time dropdown auto-select "surprises"

`buildTimeOptions` (~491–503) fills 48 half-hour options `0000`…`2330`, then
auto-selects based on the current minutes (rule owned by
`water-ops-run-and-operate` §2; kept inline here because the runbook needs it):

| Current minutes | Selected |
|---|---|
| 0–14 | this hour `:00` |
| 15–44 | this hour `:30` |
| 45–59 | **next** hour `:00` — `h=(h+1)%24`, so 23:5x wraps to **00:00** |

So at 09:50 the dropdown pre-selects 10:00 (a future slot), and at 23:50 it
pre-selects 00:00 (which reads as "yesterday's midnight" in a log). **This is
nearest-slot rounding, not a bug** — operators log on the half-hour and a
reading taken at :50 belongs to the next slot. Do not "fix" it without owner
sign-off (`water-ops-change-control`).

Note the sludge screen differs: `buildHourlyOptions` (~505–513) is hourly and
always selects the CURRENT hour `:00` — it truncates, never rounds up. The two
selectors disagreeing near the top of the hour is expected.

Also expected: the dropdowns are populated once at load. A tab left open
overnight keeps its stale auto-selection — the operator picks manually.

---

## Runbook 8 — "My data disappeared after reload"

No persistence layer exists — by design, not omission. Verify:

```bash
grep -nE "localStorage|sessionStorage|indexedDB" index.html   # zero hits
```

Reload, tab close, standalone-app relaunch, and (on iOS) the OS evicting a
backgrounded PWA all lose every entry. The intended workflow is: enter →
copy → paste into the logging channel → data lives there. Persistence/history
is a nice-to-have on the roadmap, not required (owner decision, 2026-08-10).
Architecture rationale: `water-ops-architecture-contract`. Do not add storage
ad hoc — route through `water-ops-change-control`.

---

## Runbook 9 — iOS Safari / standalone quirks (procedural)

Deep-dive content (install cycles, manifest anatomy, per-OS differences) is
owned by `water-ops-pwa-and-mobile-playbook`. Quick triage only:

- **Number inputs:** every numeric field is `type="number"` with
  `inputmode="decimal"` (or `numeric` for integer fields like the totalizer and
  TSS). `inputmode="decimal"` is what guarantees the iOS decimal-point keypad;
  if a field shows the full text keyboard, check its `inputmode` attribute
  first. Also note iOS `type=number` silently blanks the field's `.value` on
  some invalid intermediate input — the sparse-render design tolerates this
  (blank fields simply drop out of the output).
- **Stale home-screen icon or name:** iOS caches `apple-touch-icon-180.png` and
  manifest values at install time. Bumping the SW cache does NOT refresh the
  icon. Fix: remove the home-screen app and re-add it after the new version is
  live.
- **Stuck old version in standalone mode:** the standalone app has no reload
  button. Force-quit the app (swipe away) and relaunch twice after a deploy
  (first launch swaps the SW, second loads fresh). If still stale, delete and
  re-add from Safari.
- **Action bar vs home indicator:** the sticky `#actionbar` pads its bottom with
  `env(safe-area-inset-bottom)` and the page uses `viewport-fit=cover`. If
  buttons sit under the iPhone home indicator, verify those two are intact
  (CSS ~line 136, meta viewport line ~5) — losing either in an edit is the
  usual cause.

---

## Tools: getting a console on each platform

Console errors are **invisible on mobile** — the app has no error UI and the SW
registration swallows failures — so a real inspector is mandatory for anything
non-obvious.

**Desktop (Chrome/Edge/Brave):** F12 or Cmd+Opt+I (macOS) / Ctrl+Shift+I.
Everything SW-related lives in **Application** tab → Service Workers + Cache
Storage. "Update on reload" checkbox there forces SW refresh while debugging.
Firefox: `about:debugging#/runtime/this-firefox` for SW inspection.

**iOS Safari (needs a Mac + cable):**
1. iPhone: Settings → Safari → Advanced → enable **Web Inspector**.
2. Mac Safari: Settings → Advanced → enable "Show features for web developers".
3. Cable-connect, open the page (or the installed home-screen app) on the
   phone, then Mac Safari → **Develop** menu → \<your iPhone\> → select the page.
   Standalone PWA instances appear there too, as their own entries.
4. No Mac available → no console on iOS; fall back to on-page debugging (e.g.
   temporarily render errors into the DOM in a WIP build — never ship that).

**Android Chrome (any desktop OS + cable):**
1. Phone: enable Developer Options → **USB debugging**.
2. Desktop Chrome: open `chrome://inspect/#devices`, cable-connect, accept the
   trust prompt on the phone.
3. The page (and installed PWA) appear under the device → **inspect** opens full
   DevTools including the Application tab.

**Serving locally for realistic tests:** `file://` disables SW and
`navigator.clipboard` — run `python3 -m http.server` and use
`http://localhost:8000` instead. Details in `water-ops-run-and-operate`.

---

## Provenance and maintenance

Everything above was verified 2026-08-10 against the 890-line index.html
(local WIP copy carries a browser-download suffix in its name —
`water-ops-failure-archaeology` explains) and the owner-supplied sw.js v9.
Line numbers drift with every edit; re-verify before citing:

```bash
grep -n "id=\"copyBtn\"" index*.html                    # button HTML, initial label "Copy" (~453)
grep -n "Copy entry" index*.html                        # flashCopied restore label (~846)
grep -n "function flashCopied\|function fallbackCopy\|function getPlainText" index*.html   # ~844 / ~849 / ~821
grep -n "function sRender\|function buildTimeOptions\|function showScreen" index*.html     # ~603 / ~491 / ~477
grep -n "serviceWorker.register" index*.html            # silent .catch registration (~885)
grep -nE "localStorage|pushState" index*.html           # expect zero hits (no persistence, no history API)
grep -n "sbSel.length===0" index*.html                  # sludge null-guard in getPlainText (~838) and sbRender empty state (~802)
grep -n "CACHE" sw.js 2>/dev/null || echo "sw.js absent locally (lives in repo)"   # current cache name (waterops-v9 as of 2026-08-10)
# Cache-bump history (ledger owned by water-ops-failure-archaeology E4):
git log -p --follow -- sw.js | grep -E "^\+var CACHE"  # run in a clone of the repo (read-only)
# Deployed-origin staleness (as of 2026-08-10 the origin is https://daniel-ai-yi.github.io/water-ops; Azure move planned — check whatever origin you deploy to):
curl -s https://<deployed-origin>/sw.js | grep -m1 CACHE
```

If a grep no longer matches, the underlying fact may have been fixed — update
the affected runbook rather than deleting it (mark the fix and date).
