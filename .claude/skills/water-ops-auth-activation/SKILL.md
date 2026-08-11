---
name: water-ops-auth-activation
description: >
  Continuation runbook for Water Ops authentication and the NALCO dashboard redesign.
  Load when: deploying Water Ops to Azure Static Web Apps; activating or testing login /
  sign-in / registration / admin approval; a user reports "stuck on the login screen",
  "preview mode showing", or "awaiting admin approval"; granting or revoking operator
  access; swapping the placeholder NALCO wordmark or QCELL text for official logo files;
  or resuming the feature/auth-dashboard branch work. This skill records exactly what was
  built on 2026-08-10, what is intentionally NOT yet active, and the step-by-step Azure
  activation procedure.
---

# Water Ops — Auth Activation & Redesign Continuation

**State as of 2026-08-10, branch `feature/auth-dashboard` (NOT yet merged to `main`):**
an authentication *template* and NALCO-branded dashboard were built and verified locally.
Authentication is NOT enforced anywhere yet — the app deliberately runs in labeled
"preview mode" on any host that is not Azure Static Web Apps (SWA). Merging + deploying
to SWA is what turns the gate on. Nothing here contradicts the no-build rule: deploy is
still copy-files; SWA's auth is host-level, not app code.

## What was built (verify each in `index.html` on the branch)

| Piece | Where | Verify |
|---|---|---|
| Login screen `#screen-login` (initial active screen) | HTML after `<noscript>` | `grep -n 'id="screen-login"' index.html` |
| Auth gate JS: `authInit`/`authShow`/`authEnter`, roles `['operator','admin']` | script, after screen mgmt | `grep -n "authInit\|AUTH_ROLES" index.html` |
| Server-side gate config (the REAL security) | `staticwebapp.config.json` | routes: `/*` requires `operator`/`admin`; 401→`/.auth/login/aad`; 403→`/unauthorized.html` |
| "Awaiting approval" page | `unauthorized.html` | anonymous+authenticated allowed |
| SW: cache `waterops-v10`, never intercepts `/.auth/*` | `sw.js` | `grep -n "waterops-v10\|/.auth/" sw.js` |
| NALCO brand theme (blue `#0072CE` approx.) + dashboard: NALCO wordmark ┃ QCELL, "Managed Operations Dashboard", section **U3** wrapping the four tool cards | CSS block "NALCO BRAND THEME"; `#screen-home` | `grep -n "unit-title\|qcell-mark\|dash-head" index.html` |
| Telegram REMOVED (button, `.tg` CSS, handler, `tgBtn` var) | — | `grep -c "tgBtn\|t.me" index.html` → 0 |

Client auth flow: `fetch('/.auth/me')` → no endpoint & online → **preview mode**
(labeled, Continue button); no endpoint & offline → enter (installed operator already
approved; file is local anyway); signed out → Sign in button → `/.auth/login/aad`;
signed in without role → **pending** screen + sign-out; signed in with `operator` or
`admin` role → dashboard.

## Honest security model (do not weaken, do not oversell)

- The ONLY real gate is SWA's server-side route rules in `staticwebapp.config.json`.
  The client JS is UX, not security — anyone can bypass client JS. Never "fix" the login
  by adding client-side passwords or credentials in the file (owner rule: no secrets
  client-side, ever).
- Offline wrinkle (accepted, by design): once an approved operator has the PWA cached,
  the cached app opens offline without re-auth. Server data (future SharePoint stage)
  must still authenticate per-request.
- Preview mode exists so GitHub Pages/localhost keep working during transition. It says
  so on screen. Removing that fallback before SWA is live would brick non-SWA hosting.

## Azure activation runbook (the part that was blocked on 2026-08-10 — no Azure account set up yet)

1. Merge `feature/auth-dashboard` → `main` (owner reviews the redesign first).
2. Azure Portal → **Create Static Web App** (Free tier): connect GitHub repo
   `Daniel-ai-yi/water-ops`, branch `main`, preset **Custom**, app location `/`, NO api,
   NO build (output location empty). This adds a GitHub Actions workflow file to the repo
   — that workflow only uploads files; it is not a build system (note it in
   water-ops-build-and-env when it appears).
3. First deploy → visit the SWA URL. Expect: redirect to Entra ID (`aad`) sign-in
   (server 401 override). After sign-in with an unapproved account: redirect to
   `unauthorized.html` ("awaiting approval").
4. Grant access: Azure Portal → your Static Web App → **Role management** → Invite →
   enter the user's identity, role `operator` (or `admin`). User revisits → dashboard.
   This IS the "register, then admin allows" flow the owner asked for: sign-in =
   registration; role invitation = admin approval. Revoke by deleting the role.
5. Verify the SW cycle on the new origin (pwa playbook §safe-deploy): first visit
   installs `waterops-v10`; confirm `/.auth/me` is NOT in Cache Storage.
6. Old origins: GitHub Pages was disabled when the repo went private (2026-08-10,
   site 404s). Any phone still running the cached old app must reinstall from the SWA
   URL — installed PWAs do not follow origin moves.
7. Auth provider is `aad` (Microsoft Entra ID — fits an Ecolab site). To use another,
   change `/.auth/login/aad` in BOTH `index.html` (sign-in click handler) and
   `staticwebapp.config.json` (401 redirect). Consider blocking unused providers with
   routes (`/.auth/login/github` → 404) to keep the door count low.

## Redesign follow-ups (open items, 2026-08-10)

- **Official logo assets pending**: the NALCO wordmark is an inline HTML/SVG
  recreation (`.nalco-mark`) and QCELL is styled text (`.qcell-mark`). When the owner
  supplies official PNG/SVG files, replace both markup instances (login card +
  dashboard brandbar), add the files to `sw.js` ASSETS, and bump the cache. Brand blue
  `#0072CE` in `--brand` is an approximation — confirm official hex with the owner.
- **U3 is the first unit section**; more units (U1 etc. — a `U1-WaterQuality`
  spreadsheet exists in the owner's files) will follow the same
  `<section class="unit">` + `.unit-title` pattern on the dashboard.
- Skills that still describe `main` (correctly, until merge): run-and-operate documents
  the Telegram button as deprecated-but-present; platform-campaign Stage A is marked
  open. **After merge, update both** plus build-and-env's file list (+2 files:
  staticwebapp.config.json, unauthorized.html) and debugging-playbook (login screen is
  now the initial screen — a "blank/stuck at login" symptom means authInit threw before
  showing a state; check console).

## When NOT to use this skill

Day-to-day change mechanics → `water-ops-change-control`. PWA/SW deploy mechanics →
`water-ops-pwa-and-mobile-playbook`. Roadmap context and later stages (SharePoint,
theming scope) → `water-ops-platform-campaign`. Output formats → `water-ops-run-and-operate`.

## Provenance and maintenance

Built and locally verified 2026-08-10 (login → preview → dashboard → tools; EQ Log
fixture output character-exact; zero console errors). Re-verify:
- `git -C <clone> branch -a | grep auth-dashboard` — branch still unmerged?
- `grep -n "screen-login" index.html` — login screen present?
- `grep -c "tgBtn" index.html` — expect 0 on the branch, non-zero on old main.
- `curl -s <deployed-origin>/.auth/me` — 200 JSON on SWA; 404 elsewhere (= preview mode).
- `grep -n "waterops-v" sw.js` — v10 on the branch.
