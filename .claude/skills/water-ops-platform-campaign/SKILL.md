---
name: water-ops-platform-campaign
description: >
  Load this skill when work on the Water Ops app involves any of: removing the Telegram
  button/share integration; moving hosting to confidential hosting (Azure Static Web Apps,
  Azure Blob, private repo — as of 2026-08-10 the app is live on public GitHub Pages at
  daniel-ai-yi.github.io/water-ops); adding a login screen, username/password
  gating, authentication, or access control; sending entries directly to SharePoint or a
  database (replacing copy-paste); NALCO/Ecolab branding or theming; desktop/PC/responsive
  UX beyond the current phone-first layout; or the question "what's next / what's the
  roadmap for Water Ops". This is the multi-stage modernization roadmap with decision
  gates. It coordinates; it does not execute — for day-to-day change mechanics use
  water-ops-change-control, for PWA/hosting mechanics use water-ops-pwa-and-mobile-playbook.
---

# Water Ops platform campaign (decision-gated modernization roadmap)

**Status of everything below, as of 2026-08-10: OPEN. Nothing in Stages A–E is implemented.**
This skill records the owner's stated direction (given 2026-08-10) and the honest technical
shape of each stage. It exists so future sessions execute the right stage, in the right
order, without overselling or silently breaking what already works.

## The driving tension (state it honestly)

Water Ops today is a single static `index.html` (~890 lines, vanilla HTML/CSS/JS): no
build step, no framework, no backend, no external requests (Telegram link removed 2026-08-10),
works fully offline via a cache-first service worker. Those properties are why it is
reliable in the field.

The owner's destination conflicts with several of them: authenticated users (login),
entries flowing directly to SharePoint or a database (needs an API and credentials),
confidential hosting (the current repo is public and the app is live on public GitHub
Pages at `https://daniel-ai-yi.github.io/water-ops/`, verified 2026-08-10). Note the
owner's stance (2026-08-10): the app is static **today**, but a move to a dynamic app
(backend) is an accepted consideration for login and per-user personalization — the
gates below decide *when/how*, not *whether it may ever happen*. **Each stage below resolves
one piece of that conflict without breaking what works.** Do not collapse stages together;
each has its own decision gate.

## Standing rule across all stages (owner directive, 2026-08-10)

**SECURITY IS A MUST.** This is company data (NALCO Water / Ecolab site operations).
Maximum confidentiality; no third-party services that expose entry data externally; no
secrets or credentials ever in the client file (it is fully readable by anyone who can
fetch it); no third-party scripts/CDNs/analytics. Every stage's plan must pass this test
before its gate is even considered.

## Current state snapshot (verified against the code, 2026-08-10)

| Fact | Evidence in `index.html` (local WIP file `index[23].html`; re-verify line numbers) |
|---|---|
| Telegram touchpoints | REMOVED 2026-08-10 (Stage A complete, commit 8ee7a8d) — the app now makes zero external requests |
| Only external request in the app | That `t.me/share/url` `window.open` — nothing else (no fetch/XHR, no CDN) |
| No auth code of any kind | grep for `password`, `login`, `auth` finds nothing |
| Public repo + public origin | Canonical repo `https://github.com/Daniel-ai-yi/water-ops` is PUBLIC (anonymous clone works, verified 2026-08-10); its HEAD `index.html` is byte-identical to the current local working file (the repo is current). The app is live on public GitHub Pages at `https://daniel-ai-yi.github.io/water-ops/` (verified serving `waterops-v9`, 2026-08-10) |
| Phone-first layout | `.wrap{max-width:600px}` (~29), `.actionbar .inner{max-width:600px}` (~142), single column, stark black/white palette |
| Touch targets | `min-height:52px` on inputs/segments (~83, 95, 157); `.tg` button 54px |
| Deploy model | Copy 6 static files (index.html, sw.js, manifest.webmanifest, 3 icons) to a static host |

---

## Stage A — De-Telegram — ✅ COMPLETE 2026-08-10 (commit 8ee7a8d, merged to main)

Kept below for the record; the removal shipped together with the auth template and NALCO
dashboard redesign (see `water-ops-auth-activation`). `grep -in "tgBtn|t\.me" index.html`
now returns nothing.

**Goal.** Remove every Telegram touchpoint from the app.

**Why (owner's words, paraphrased).** The app was born as a formatter for a Telegram bot
that parsed entries into the plant log; the bot is retired, so the Telegram aspects should
be removed. Removal also fixes a standing confidentiality wart: company readings currently
have a code path that flows through a third-party service (t.me).

**Prerequisites.** None. This stage is independent of all others.

**Concrete first steps — exact removal inventory (verify each before deleting; line
numbers are as of 2026-08-10):**

1. Delete the `.tg` / `.tg:active` CSS rules (~lines 147–148).
2. Delete the `<button class="tg" id="tgBtn" type="button">Telegram</button>` element from
   `#actionbar` (~line 452).
3. Delete `var tgBtn = $('tgBtn');` (~line 471).
4. Delete the `tgBtn.addEventListener('click', ...)` handler containing the
   `window.open('https://t.me/share/url?url=&text=' + encodeURIComponent(text), '_blank')`
   call (~lines 868–872).
5. Confirm zero remaining references: `grep -in "tgBtn\|t\.me\|telegram\|\.tg" index*.html`
   must return nothing (beyond incidental substring matches you have inspected).

**Follow-ups in the same change:**
- The action bar drops from 3 buttons to 2 (Copy, Clear) — check the flex layout still
  looks right; buttons are `flex:1` so they will widen.
- Docs drift: `water-ops-run-and-operate` documents the Telegram button (as deprecated)
  and `water-ops-validation-and-qa`'s checklist mentions it — update both after the code
  change lands.

**Decision gate.** Owner confirms timing of the removal. Then run the full pre-ship
checklist (`water-ops-validation-and-qa`) and bump the SW cache name
(`water-ops-change-control` owns that rule — every deploy bumps `waterops-vN`).

**Risks & security notes.** Low risk; pure deletion. Net security gain (removes the one
external data egress path). Main failure mode: leaving a dangling `tgBtn` reference that
throws inside the IIFE and kills all button wiring — hence step 5.

**Status: OPEN** (sanctioned, not yet done — the grep in step 5 tells you if it has since landed).

---

## Stage B — Confidential hosting (Azure candidate) + repo privacy

**Goal.** Move the app off the public GitHub repo (and off any public origin that may be
serving it — establishing where it is actually deployed is part of this stage) to
confidential hosting; owner direction: "optimized for loading on a live static web site
such as Azure."

**Why.** Verified 2026-08-10: the canonical repo `https://github.com/Daniel-ai-yi/water-ops`
is PUBLIC — anyone can anonymously clone company tooling. That is a standing
confidentiality gap, plainly stated. (The repo is otherwise current: its HEAD `index.html`
is byte-identical to the local working file.) The live deployed origin is GitHub Pages at
`https://daniel-ai-yi.github.io/water-ops/` (owner-confirmed and verified serving
`waterops-v9`, 2026-08-10) — a PUBLIC origin, which doubles the confidentiality gap this
stage closes: private the repo, and move serving off the public Pages URL.

**Prerequisites.** None technically, but do Stage A first so the migrated app is already
Telegram-free.

**Concrete first steps (all CANDIDATES until the gate).**
- Evaluate **Azure Static Web Apps** (static hosting + HTTPS + optional built-in auth —
  the auth feature feeds Stage C directly) vs **Azure Blob static website hosting**
  (simpler, but no built-in auth).
- Make `Daniel-ai-yi/water-ops` private, or deploy from a non-GitHub-Pages pipeline;
  either way, ensure no public origin continues to serve the app. First determine the
  current deployed origin (if any) — it is unverified as of 2026-08-10.
- Keep the deploy model unchanged: copy the same 6 static files. No build step is
  introduced by this stage.

**Decision gate.** Owner picks the host, accepts account/cost implications, and signs off
before any migration.

**Risks & security notes.**
- **SW stale-origin migration trap:** installed home-screen apps point at whatever origin
  they were installed from and will NOT follow an origin move; a cache-first SW on the
  old origin will keep serving the old app there indefinitely. Plan explicit user comms
  ("delete the old icon, install from the new URL") and, if possible, deploy a
  self-destructing SW to the old origin. Mechanics live in
  `water-ops-pwa-and-mobile-playbook` §3 ("Origin migration / decommissioning an old
  origin") — cross-reference, do not improvise.
- Verify the new host serves HTTPS (required for SW and clipboard) and that access is
  restricted per the confidentiality rule (fully realized only after Stage C).

**Status: OPEN** (no host chosen; repo still public as of 2026-08-10).

---

## Stage C — Login screen / access control

**Goal.** Username/password gating so only authorized plant staff reach the app (owner
wants a login screen).

**Why.** Confidentiality rule: company data must not be open to anyone with the URL.

**The hard truth — state it bluntly.** A purely client-side login in a static single file
is **security theater**. Any credentials or checks embedded in `index.html` are readable
by anyone who fetches the file (violating the security directive), and the file itself
remains fetchable regardless of any JavaScript gate in front of it. Do not build a
JS-only login and call the requirement met. **NEVER hardcode credentials client-side.**

**Real options, honestly labeled (all CANDIDATES):**

| Option | What it is | Trade-off |
|---|---|---|
| 1. Host-level auth | Azure Static Web Apps built-in auth / Microsoft Entra ID in front of the site | Natural if Stage B picks SWA; likely SSO story for an Ecolab site; no app code changes for the gate itself |
| 2. Minimal backend / serverless function | Auth endpoint issuing sessions/tokens | Real auth, but **breaks "no backend"** — that is a decision gate, not a detail |
| 3. HTTP Basic Auth at the host layer | Host-configured username/password | Crude but honest; credentials managed at host, never in the file |

**PWA wrinkle (do not skip).** The cache-first SW serves `index.html` from cache while
offline — an already-installed client keeps working without re-authenticating, and the
cached file bypasses any per-request gate. The auth boundary must sit at the host/API
layer (protecting the origin and any future APIs), not inside the cached file. Decide
deliberately how offline use interacts with auth (e.g., authenticated install, host gate
on first fetch and on any sync).

**Prerequisites.** Stage B's hosting choice — option 1 only exists if SWA is chosen.

**Decision gate.** Depends on Stage B's outcome; owner decides the auth provider and how
users are managed (who adds/removes operators).

**Status: OPEN** (no auth code exists in the app as of 2026-08-10 — verified by grep).

---

## Stage D — Entries direct to SharePoint / database (replaces copy-paste egress)

**Goal.** Owner: "hoping for entries directly to a sharepoint or other database" —
submitted from the app instead of copy-pasted into the plant channel.

**Why.** Removes the manual copy-paste hop; pairs with the retired-Telegram-bot direction
(the bot used to parse entries; a database ingests them directly).

**What it truly requires — no sugarcoating.** An authenticated API path. For SharePoint
Lists that means Microsoft Graph: an Azure AD (Entra) app registration and access tokens —
which **must NOT live in the static client** (anything in `index.html` is public to its
readers). The realistic shape: a small serverless proxy (e.g., an Azure Function) holds
the credentials/token flow; the client sends it the already-formatted entry text or
structured fields; the proxy writes to SharePoint/DB.

**Prerequisites.** Stage B (hosting/platform), and it benefits strongly from Stage C
(authenticated users → attributable entries, and the proxy can require the same identity).

**Concrete first steps (CANDIDATES).** Pick the target store (SharePoint List vs other
DB); design the entry payload (formatted text block vs structured fields — prefer sending
both during transition); prototype one tool's submission path end-to-end before touching
the other three.

**Decision gate — explicit and non-trivial.** This stage **ends the pure-static /
no-backend era** of the app. The owner has signaled openness to that trade (2026-08-10:
a dynamic app is an accepted consideration for login and personalization), but openness
is not sign-off — the owner must still approve the concrete design before any code is
written. The gate also covers offline strategy: queue entries while offline and sync
later? That requires client-side persistence, which today does not exist anywhere in the
app and is currently a nice-to-have, not a requirement — adding it is part of this gate.

**Risks & security notes.**
- Credentials/tokens only in the serverless layer; the client gets, at most, a
  short-lived user-scoped token via real auth (Stage C).
- Keep **Copy as the fallback egress** during any transition — operators must never be
  blocked from logging because a network call failed.
- Preserve the exact output-format contract (owned by `water-ops-run-and-operate`) or
  version it deliberately with owner sign-off; downstream consumers may depend on the
  text shape.

**Status: OPEN** (the app makes zero network requests for data today — verified 2026-08-10).

---

## Stage E — NALCO Ecolab theming + desktop/responsive UX

**Goal.** Apply the NALCO Ecolab look; improve UX/UI (owner calls this VITAL); target
iOS AND Android AND desktop/PC ("mobile-only dependency not necessary").

**Why.** Current UI is deliberately stark: black-on-white, single column,
`max-width:600px` wrapper (verified ~line 29). Fine for a phone in the field; wasteful
and unbranded on a PC.

**Prerequisites.** None technically (independent-ish), but doing it after Stage A avoids
restyling a button that is being deleted.

**Concrete first steps (CANDIDATES).**
1. Obtain OFFICIAL brand assets/colors from company sources. **Do not scrape or guess
   trademarked assets or logos**; get owner approval on the palette before applying it.
2. Design a wider-viewport layout — e.g., multi-column field groups on desktop via media
   queries — **without breaking the phone-first single-column field workflow** operators
   rely on.
3. Keep touch targets at or above the current 52px min-height sizing (verified in CSS)
   on touch devices.
4. Stay within the existing constraint set: single file, no framework, no CDN fonts or
   external assets (confidentiality rule and offline cache both forbid them — embed any
   brand font/logo as local files added to the SW asset list, with owner approval).

**Decision gate.** Owner supplies or approves the brand direction and the desktop layout
approach.

**Status: OPEN** (UI is unbranded black/white as of 2026-08-10).

---

## Sequencing and dependencies (one glance)

| Stage | Depends on | Notes |
|---|---|---|
| A — De-Telegram | nothing | Sanctioned now; do first; pure deletion |
| B — Azure hosting + private repo | nothing (A first by convenience) | Choice made here shapes C and D |
| C — Login / access control | **B** (host choice determines auth options) | No client-side-only login, ever |
| D — SharePoint/DB submission | **B**; benefits from **C** | Ends the no-backend era — biggest gate |
| E — Theming + desktop UX | independent-ish (after A by convenience) | Blocked only on official brand assets |

## When NOT to use this skill

- Day-to-day fixes or small changes to the app → `water-ops-change-control`.
- Executing PWA / service-worker / hosting mechanics (cache bumps, SW lifecycle, the
  stale-origin trap in detail) → `water-ops-pwa-and-mobile-playbook`.
- Adding a fifth plant tool to the app → `water-ops-new-tool-recipe`.
- Anything about the exact copy-output text formats → `water-ops-run-and-operate`.

## Provenance and maintenance

All claims verified 2026-08-10 against the local WIP app file (`index[23].html`, deployed
name `index.html`) and the GitHub repo. Stages A–E were all OPEN at that date. One-line
checks so a future session can tell what has since landed:

- Stage A done? `grep -in "tgBtn\|t\.me\|telegram" index*.html` → empty means the removal landed.
- Stage B done? Check repo visibility: `gh repo view Daniel-ai-yi/water-ops --json visibility` (was PUBLIC as of 2026-08-10); check where the app is actually deployed — as of 2026-08-10 it was `https://daniel-ai-yi.github.io/water-ops/` (`curl -s https://daniel-ai-yi.github.io/water-ops/sw.js | grep CACHE` → `waterops-v9`) — and after any Azure move, whether the old Pages origin still serves the app (it must be taken down or redirected; installed home-screen apps pinned to it will not follow).
- Stage C done? `grep -in "password\|login\|auth" index*.html` (was empty) — but remember real auth lives at the host layer, so also check the host's auth configuration, not just the file.
- Stage D done? `grep -in "fetch(\|XMLHttpRequest" index*.html` (was empty — app made zero data network requests).
- Stage E done? `grep -n "max-width:600px" index*.html` (was the layout cap) and look for brand colors beyond black/white in the `:root` CSS custom properties.
