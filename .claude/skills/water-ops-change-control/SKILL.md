---
name: water-ops-change-control
description: >-
  Safe-change PROCEDURE and rules for the Water Ops single-file PWA. Load this
  skill BEFORE editing index.html in any way, BEFORE any deploy (copying files
  to the static host), when deciding whether a proposed change is allowed or needs
  owner sign-off, when touching anything that renders via innerHTML or produces
  copy-output text, or when unsure how changes are verified in a repo with no
  build system and no tests. Also covers house code style (comments, JS idiom,
  element-id naming) and the current sanctioned-work list.
---

# Water Ops change control

How to change a single-file, no-build, no-test, no-CI app without breaking it.

**The app in one paragraph.** Water Ops is one static `index.html` (~890 lines: CSS + markup + a single IIFE of vanilla JS) plus `sw.js`, `manifest.webmanifest`, and three icon PNGs. It is a PWA (Progressive Web App — installable web page with an offline cache) used by water-treatment plant operators at a NALCO Water (Ecolab) site. Four tools — Quality Sampling, EQ Log, Parshall Flume, Sludge Blanket — each format hand-entered readings into a fixed plain-text block the operator copies into the plant's logging channel. Domain terms (EQ tank, Lamella, Parshall flume, sludge blanket, HMI, Myron L…) are defined in `water-treatment-domain-reference`; get glosses there, not here.

**There is no build step, no bundler, no npm, no test suite, no CI.** This is deliberate (see `water-ops-build-and-env`). Every change is a hand edit to a static file, verified by a human in a browser. That is why this procedure exists: the process is the only safety net.

## When NOT to use this skill

| You are trying to… | Use instead |
|---|---|
| Debug an existing failure (stale app, clipboard broken, screen stuck…) | `water-ops-debugging-playbook` |
| Add a whole new tool (screen #5) | `water-ops-new-tool-recipe` (then come back here for the deploy steps) |
| Understand WHY the architecture is the way it is | `water-ops-architecture-contract` |
| Run the pre-ship manual checklist itself | `water-ops-validation-and-qa` |
| Plan/stage the big roadmap items (Azure, login, SharePoint) | `water-ops-platform-campaign` |
| Understand the exact copy-output text formats | `water-ops-run-and-operate` (it owns the format spec) |
| Anything service-worker/manifest/install-cycle deep | `water-ops-pwa-and-mobile-playbook` |

## The safe-change loop

Run this loop for EVERY change, however small.

1. **Read the whole relevant region first.** The file is one IIFE; state and handlers are interleaved. Never edit a function you haven't read together with its wiring (event listeners are registered in blocks after each tool's functions). Minimum read: the tool's HTML screen block AND its JS section AND the shared `COPY & CLEAR` section at the bottom.
2. **Check the duplication trap** (below) if the change touches Quality Sampling output in any way.
3. **Check the non-negotiables table** (below). If the change alters any produced output text, STOP and get owner sign-off first.
4. **Make the edit**, matching house style (below).
5. **Manually verify** per the `water-ops-validation-and-qa` checklist — every screen renders, copy output matches spec character-for-character, clear works, security greps pass. That skill owns the checklist; do not improvise a subset.
6. **Bump the service-worker cache name on EVERY deploy.** In `sw.js`, change `var CACHE = 'waterops-vN';` to `vN+1`. This is the app's ONLY update mechanism: the fetch handler is cache-first, so a stale service worker serves the old app forever until a byte-different `sw.js` installs, `skipWaiting()`s, and deletes old caches on activate. Deploying new `index.html` without bumping the cache name means users never see your change. As of 2026-08-10 the current name is `waterops-v9`, so the next deploy ships `waterops-v10`. Full lifecycle detail: `water-ops-pwa-and-mobile-playbook`.
7. **Deploy = copy static files.** There is nothing else. Copy the changed files (up to eight as of 2026-08-10, incl. `staticwebapp.config.json` and `unauthorized.html` — file list owned by `water-ops-build-and-env`) to the static host (as of 2026-08-10: GitHub Pages at `https://daniel-ai-yi.github.io/water-ops/`, i.e. push to the `Daniel-ai-yi/water-ops` repo's default branch; Azure move planned — re-confirm the origin if time has passed). Then verify the update actually propagates on a real device (QA skill, install/offline cycle item).

## Non-negotiables

Security first — owner directive (2026-08-10): "security is a must, always considered." These are standing rules, not roadmap items. Violating any of them is a blocked change regardless of who asked.

| # | Rule | Rationale |
|---|---|---|
| 1 | **Keep `esc()` on every innerHTML path.** Every string that reaches `.innerHTML` and contains user input must pass through `esc()` (defined near the top of the IIFE; escapes `&` `<` `>`). All four render functions (`sRender`, `lRender`, `fRender`, `sbRender`) do `lines.map(esc).join('\n')` — preserve that pattern in any edit and any new code. Known intentional exception: `sBuildRow` sets `nm.innerHTML = f.label` from the `SF` field catalog — that is developer-authored config (needed for the `H₂O₂` entity), not user input; do not extend this exception to anything user-typed. | The output `<pre>` renders via innerHTML. Unescaped operator input (e.g. a stray `<` in a totalizer note) is an XSS vector and a rendering bug. |
| 2 | **No secrets in the client file. Ever.** No credentials, API keys, tokens, or internal URLs-with-auth in `index.html` or `sw.js`. | The file is fully readable by anyone with the URL — as of 2026-08-10 the canonical repo (`github.com/Daniel-ai-yi/water-ops`) is PUBLIC and the app is served from public GitHub Pages (`https://daniel-ai-yi.github.io/water-ops/`), so the file must be treated as readable by anyone. Any future login must not hardcode credentials client-side (see `water-ops-platform-campaign` for honest limits of client-side auth). |
| 3 | **No third-party scripts, CDNs, analytics, or external services.** Everything self-hosted from the app's own origin. | This is company (NALCO/Ecolab) data; confidentiality is the one hard rule. External origins leak data and add supply-chain risk, and the offline cache can't vouch for them. |
| 4 | **HTTPS-only assumptions.** The service worker and `navigator.clipboard` require a secure context; treat HTTPS as the only real deployment target. | `file://` and plain HTTP silently degrade (SW registration has an intentional silent `.catch`, clipboard falls back to `execCommand`). Don't add features that only work insecurely. |
| 5 | **Output-format lines are a contract — never change produced text without owner sign-off.** Includes "obvious fixes": EQ Log's `Time:` (colon) vs every other tool's `Time -` (dash) is a real, verified inconsistency that downstream consumers may depend on. Do NOT normalize it. | The copied text feeds the plant's logging channel (historically parsed by a bot). Format spec is owned by `water-ops-run-and-operate`. |
| 6 | **Stay single-file / static / dependency-free.** No build step, no framework, no npm, no backend calls (until a platform-campaign stage gate explicitly changes this). | The whole operating model — deploy-by-copy, offline-first, auditable-at-a-glance — depends on it. Rationale deep-dive: `water-ops-architecture-contract`. |

## The duplication trap (regression risk #1)

There is no test suite. The single most likely silent regression is Quality Sampling **preview vs copy drift**: the on-screen preview (`sRender`) and the copied text (`getPlainText`, `screen-sample` branch) build the sample lines with DUPLICATED code, not a shared function. The authoritative line-level detail (current line pairs, the intentional `trimEnd`/`esc` differences, why it is this way) is owned by `water-ops-architecture-contract` (weak point W1). Your procedural rule is:

**Grep both before editing either** (the `index*.html` glob matches the local download-named copy — see the note in Provenance):

```
grep -n "ST\[tank\]" index*.html
grep -n "sFmtValue" index*.html
grep -n "Time - " index*.html
```

Any change to sample line-building (headers, field order, separators, `sFmtValue`, time display) MUST be applied identically in both places, then verified by comparing the preview to an actual paste of the copied text (QA checklist item). The branches are not textually identical (intentional differences enumerated in W1) — mirror the LOGIC, keep those differences.

The other tools (`lBuild`, `fBuild`, `sbBuild`) are safe from this: preview and copy share one build function. Keep it that way in new code.

## Sanctioned work (as of 2026-08-10)

Owner-approved directions. Anything NOT on this list that changes behavior, output text, or architecture needs owner sign-off first. Staging, prerequisites, and decision gates live in `water-ops-platform-campaign` — check there before starting any of these.

| Work item | Status |
|---|---|
| ~~Remove Telegram integration~~ | DONE 2026-08-10 (commit 8ee7a8d) — app now has zero external requests |
| NALCO Ecolab visual theming | Sanctioned |
| Login screen (username/password gate) | Sanctioned direction; design constraints in platform-campaign (no client-side credentials — rule 2) |
| Azure migration (off the public GitHub Pages origin `daniel-ai-yi.github.io/water-ops`; repo public as of 2026-08-10) | Sanctioned; closes the public-repo/public-origin confidentiality gap |
| SharePoint / database entry submission | Sanctioned direction; decision-gated (breaks "no backend") |
| Desktop-responsive UX (drop mobile-only assumptions) | Sanctioned |
| Persistence/history tracking | Nice-to-have, NOT required |
| In-app "tool maker" | Candidate idea only — not a commitment |

## House style (keep it light, match what's there)

- **Banner comments.** Match the existing section-banner style: JS sections use `/* ================ NAME ================ */` full-width banners (e.g. `QUALITY SAMPLING`, `COPY & CLEAR`), JS sub-sections use `/* ---- ... ---- */`, CSS groups use `/* ---- Screens ---- */`, HTML screen blocks use `<!-- ==== ... ==== -->` banner comments. New sections get the same treatment; don't invent a new style.
- **ES5-ish vanilla JS.** `var`, `function(){}` callbacks, no arrow functions, no classes, no template literals, no modules. Match the file even where ES6 would be nicer — consistency beats modernity in a file audited by eye.
- **Element ids: kebab-case with per-tool prefixes.** `s-` Quality Sampling (`s-time`, `s-tank`, `s-out`), `l-` EQ Log (`l-time`, `l-tot-unit`), `f-` Parshall Flume (`f-flow`), `sb-` Sludge Blanket (`sb-psi`, `sb-slvl-1`). Shared elements are unprefixed (`actionbar`, `copyBtn`). New ids follow the pattern.
- **Function naming mirrors the prefix:** `sRender`/`lRender`/`fRender`/`sbRender`, `sClear`/`lClear`/`fClear`/`sbClear`. New per-tool functions take the tool's letter prefix.
- Everything lives inside the single IIFE; only `showScreen` and `goHome` are exposed on `window` (needed by inline `onclick` in the HTML). Don't add window globals without the same necessity.

## Provenance and maintenance

Facts above verified against the app file and the owner-supplied `sw.js` on 2026-08-10. Local-name note: the working copy may be download-named `index[23].html` (deploys as `index.html` — quirk owned by `water-ops-failure-archaeology`), and `sw.js` is absent from the local folder (canonical content in `water-ops-pwa-and-mobile-playbook`) — the commands below use the `index*.html` glob and guard the sw.js grep so they run either way. Re-verify drift-prone claims before relying on them:

- esc() still guards render paths: `grep -n "map(esc)" index*.html` (expect 4 hits: sRender, lRender, fRender, sbRender)
- innerHTML uses that must stay escaped/audited: `grep -n "innerHTML" index*.html`
- Duplication trap still exists (two independent sample line-builders): `grep -n "lines=\['Time - '+tDisp, t.header, ''\]" index*.html` (2 hits = still duplicated; 1 hit = refactored, update this skill)
- Current SW cache name: `grep -n "var CACHE" sw.js 2>/dev/null || echo "sw.js absent locally (lives in repo)"`
- No third-party origins crept in: `grep -nE "https?://" index*.html sw.js 2>/dev/null` (expect NONE since the 2026-08-10 Telegram removal)
- No secrets: `grep -niE "password|token|api[_-]?key|secret" index*.html` (expect no hits, except login-screen UI text if/when that ships)
- Telegram removal status: `grep -n "tgBtn" index*.html` (hits = not yet removed)
- Id-prefix convention holds: `grep -oE 'id="(s|l|f|sb)-[a-z0-9-]+"' index*.html | sort -u`
