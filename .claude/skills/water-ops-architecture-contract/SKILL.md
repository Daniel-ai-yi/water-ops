---
name: water-ops-architecture-contract
description: >
  Load BEFORE any structural change to the Water Ops app: refactors, adding or moving
  features, touching the screen system, the render/copy pipeline, or the IIFE script
  layout in index.html. Also load when you (or the user) ask WHY the app has no
  framework, no router, no build step, no backend, and no persistence — the answers
  here are deliberate decisions, not oversights. Contains the load-bearing decisions
  and their rationale, the invariants any change must preserve (esc()-before-innerHTML,
  preview===copy, single file, no external requests), and the known-weak points with
  their blast radius. NOT for routine small changes (use water-ops-change-control),
  adding a new tool (water-ops-new-tool-recipe), debugging symptoms
  (water-ops-debugging-playbook), or service-worker/PWA internals
  (water-ops-pwa-and-mobile-playbook).
---

# Water Ops — Architecture Contract

Water Ops is a single-file PWA (`index.html`, 890 lines as of 2026-08-10, plus
`manifest.webmanifest`, icons, and `sw.js` — note `sw.js` is absent from the local
working folder as of 2026-08-10; canonical content lives in
`water-ops-pwa-and-mobile-playbook`) used by water-treatment-plant operators
at a NALCO Water (Ecolab) site. Four tools — Quality Sampling, EQ Log, Parshall
Flume, Sludge Blanket — each format entered readings into a fixed plain-text block
the operator copies into the plant's logging channel. (Domain terms — EQ tank,
Lamella, Post Anoxic, Parshall flume, sludge blanket, HMI, Myron L — are glossed
one line where used below; full definitions live in `water-treatment-domain-reference`.)

This skill is the contract: what the architecture deliberately is, what must stay
true after any change, and where it is fragile. All line numbers were verified
against the real file on 2026-08-10. Line numbers drift — re-verify with the grep
commands in "Provenance and maintenance" before citing them in new work.

## When NOT to use this skill

| You are doing | Use instead |
|---|---|
| A routine change (copy tweak, field edit, deploy) | `water-ops-change-control` |
| Adding tool #5 | `water-ops-new-tool-recipe` |
| Chasing a bug symptom | `water-ops-debugging-playbook` |
| Service worker, caching, install, iOS/Android quirks | `water-ops-pwa-and-mobile-playbook` |
| Looking up output formats or how to run it | `water-ops-run-and-operate` |

## Load-bearing decisions and why they hold

Do not undo any of these without owner sign-off. Each one is doing work.

| # | Decision | Rationale | What breaks if you undo it |
|---|---|---|---|
| 1 | **One `index.html` holds everything** (CSS, markup, script) | Deploy = copy files to a static host. Nothing to build, nothing to install. The file survives being emailed, downloaded, or passed around as one artifact — which is literally how it has been iterated (see `water-ops-failure-archaeology`). | Splitting into modules introduces a build/bundle step and multi-file deploys for zero user-visible gain. |
| 2 | **No framework, no dependencies, no CDN, no analytics** | Confidentiality: this is NALCO/Ecolab company data, and the owner directive (2026-08-10) is "security is a must." No third-party code ever touches the data. Also: nothing to version-bump, nothing to supply-chain-audit. | Any external script or fetch is a data-exposure vector and violates a standing owner rule, not a preference. |
| 3 | **Vanilla ES5-style IIFE** (immediately-invoked function expression — one closure wrapping the whole script, lines 459–887) | One closure holds all state (`sVals`, `sLt`, `sbSel`, `currentScreen`). Only `showScreen` and `goHome` are exposed on `window` (lines 485–486) because inline `onclick` attributes in the HTML need them. Everything else is unreachable from outside — no global soup, no accidental console tampering surface. | New globals leak state and widen the attack/typo surface. |
| 4 | **Screen-toggle SPA, no router** | Screens are sibling `<div class="screen">` elements; CSS `display:none`/`block` via `.screen`/`.screen.active` (lines 32–33). `showScreen(id)` (line 477) flips the `active` class, toggles the actionbar, scrolls to top. There is **no History API integration: browser back exits the app.** Known, accepted limitation — operators use it as an installed home-screen app where back matters less. | A router adds code and state for four screens. If back-button support is ever wanted, it is a scoped feature, not a rewrite. |
| 5 | **Config-driven Quality Sampling** | `SF` (field catalog, lines 523–534: tss, ph, level, phosphate, sulfate, cod, fluoride, h2o2, alkalinity, turbidity — each with output label, placeholder, int/dec format, unit, `lt` "less than" support) and `ST` (tank templates, lines 535–541: eqA, eqB, lamA, lamB, paA — each listing which fields appear, in order) drive DOM building via `sBuildRow` (546) / `sRebuildFields` (583). **Adding a field or tank is data editing, not DOM surgery.** | Hand-building sample rows re-creates the copy-paste divergence this design exists to prevent. |
| 6 | **Sparse render: output contains ONLY entered readings** | Empty fields never appear in preview or copy. The UI promises it explicitly — the hint "Only the readings you enter appear in the copy." appears on three tool screens (lines 248, 355, 404) — and the canonical repo's history (https://github.com/Daniel-ai-yi/water-ops, 2026-07-06 → 2026-08-05, 18 commits) shows the app was born with sparse render and kept it throughout. (An earlier prototype design predating the repo is out of scope — see `water-ops-failure-archaeology`.) | Adding placeholder or empty lines breaks the UI's stated promise and every operator's paste expectation. |
| 7 | **Per-tool namespace prefixes** | Element ids and functions are prefixed per tool: `s-`/`s*` Quality Sampling, `l-`/`l*` EQ Log, `f-`/`f*` Parshall Flume, `sb-`/`sb*` Sludge Blanket. One flat namespace stays navigable because prefixes make collisions impossible and grep trivial. | Unprefixed additions collide silently in a single-file global-id world. |

## Mental model in 10 lines

1. `index.html` regions: CSS lines 14–165, markup 168–456, script 458–888 (IIFE 459–887).
2. Markup: home menu 175–212, then one `div.screen` per tool: sample 217–250, log 255–357, flume 362–406, sludge 411–445, shared actionbar 450–456.
3. Utilities: `$` 463, `esc` 464, `pad2` 465, `radioVal` 466; screen switching `showScreen` 477 / `goHome` 484.
4. Time dropdowns built once at load: `buildTimeOptions` 491 (30-min slots, auto-picks nearest), `buildHourlyOptions` 505 (sludge only).
5. Every tool follows the same family: **build lines → render preview → clear**, wired to `input`/`change` listeners.
6. Quality Sampling: `SF`/`ST` config → `sBuildRow`/`sRebuildFields` → `sFmtValue` → `sRender` 603 → `sClear` 616.
7. EQ Log: `lBuild` 637 (branches PA vs EQ A/B) → `lRender` 679 → `lClear` 683; `lUpdateTankView` 695 swaps field groups.
8. Parshall Flume: `fBuild` 741 → `fRender` 755 → `fClear` 759. Sludge Blanket: `sbToggle` 776, `sbBuild` 785 → `sbRender` 799 → `sbClear` 808.
9. Export: `getPlainText` 821 returns the copyable text for `currentScreen`; Copy 858 / Telegram 868 (deprecated) / Clear 874 handlers dispatch on it.
10. SW registration 884–886 (`./sw.js`, silent catch) — internals belong to `water-ops-pwa-and-mobile-playbook`.

## INVARIANTS — must hold after any change

Check every row before shipping. These are the contract.

| # | Invariant | Where enforced today (verified 2026-08-10) |
|---|---|---|
| I1 | **Every user-entered string passes through `esc()` before landing in `innerHTML`.** Security invariant, owner-mandated. `esc` (line 464) escapes `&` `<` `>` — sufficient for element-content context only; never use its output inside an HTML attribute. | All 8 `.innerHTML` sites enumerated below. New render code must use `.map(esc)` or `textContent`. |
| I2 | **Preview text === copied text, for every tool, character for character.** The operator trusts that what they see is what pastes. | EQ Log, Flume, Sludge share `lBuild`/`fBuild`/`sbBuild` between render and `getPlainText`. Quality Sampling does NOT — see weak point W1. |
| I3 | **Copy and Telegram handlers tolerate `getPlainText() === null`** (no tank selected, no valve selected). Both `if(!text) return;` guards exist (lines 860, 870). Keep the guard in any new handler. | Removing a guard makes an empty-state tap throw or copy garbage. |
| I4 | **Actionbar hidden on home, visible on every tool screen.** `showScreen` toggles `.hidden` when `id === 'screen-home'` (line 481). | Any new screen gets the actionbar for free; do not special-case around this. |
| I5 | **Single file, zero external requests at runtime.** No CDN, no fonts, no fetch, no analytics. The only outbound navigations: the deprecated Telegram share (`t.me/share/url`, line 871 — removal is sanctioned work, see `water-ops-platform-campaign`) and the SW caching same-origin assets. | Adding any external origin violates the confidentiality rule (owner, 2026-08-10). |
| I6 | **`sw.js` cache name bumped on every deployed change.** Cache-first SW means a stale cache serves the old app forever. Mechanics in `water-ops-pwa-and-mobile-playbook`; the invariant is listed here because forgetting it makes any change invisible to users. | Cache name is `waterops-v9` as of 2026-08-10. |
| I7 | **All state lives inside the IIFE.** Only `window.showScreen` and `window.goHome` (lines 485–486) are global, and only because inline `onclick` needs them. | Do not add globals; if new inline handlers are unavoidable, expose exactly the function and nothing else. |

### innerHTML call-site inventory (all 8, verified 2026-08-10)

| Line | Site | esc() status |
|---|---|---|
| 550 | `nm.innerHTML=f.label` in `sBuildRow` | Not escaped — input is the static `SF` catalog (contains the `H₂O₂` unicode label), never user text. If `SF` labels ever become user-editable, this becomes an XSS hole. |
| 584 | `box.innerHTML=''` in `sRebuildFields` | Static clear — safe. |
| 606 | `outEl.innerHTML='<span class="empty">…'` in `sRender` | Static string — safe. |
| 613 | `outEl.innerHTML = lines.map(esc).join('\n')` in `sRender` | Escaped. |
| 680 | `lBuild().map(esc).join('\n')` in `lRender` | Escaped. |
| 756 | `fBuild().map(esc).join('\n')` in `fRender` | Escaped. |
| 803 | static empty-state span in `sbRender` | Static string — safe. |
| 805 | `lines.map(esc).join('\n')` in `sbRender` | Escaped. |

Rule for new code: dynamic lines go through `.map(esc)`; anything else uses
`textContent` or a fully static string. Grep check is in Provenance below.

## KNOWN-WEAK POINTS — stated plainly, with blast radius

| # | Weakness | Blast radius | Handling |
|---|---|---|---|
| W1 | **`getPlainText()` re-implements `sRender()`'s sample-screen line building.** Lines 825–828 duplicate lines 609–612 (time formatting, header array, sparse field loop), with the tank guard at 822–824 mirroring 604–607, instead of sharing one builder. They have **already diverged once**: `getPlainText` applies `.trimEnd()` (line 829); `sRender` does not. Any edit to one that is not mirrored in the other silently breaks invariant I2 — the preview stops matching the copy, and nothing errors. | Highest-risk trap in the file. Every Quality Sampling output change is a two-site edit. | Safest future fix: extract a shared `sBuildLines()` returning the lines array, called by both (mirrors how `lBuild`/`fBuild`/`sbBuild` already work). Until then: mirror every edit, and hand-verify preview vs pasted copy. |
| W2 | **No persistence layer.** No localStorage/sessionStorage/IndexedDB anywhere. Reload, tab eviction, or an OS killing the PWA loses all in-progress entries. | Operator mid-entry loses work; no data corruption possible since nothing is stored. | Deliberate. Owner (2026-08-10): persistence is nice-to-have, NOT required. Do not add it unprompted. |
| W3 | **No backend, no sync.** Copy-paste is the only data egress today. SharePoint/database submission is roadmap direction, not built — see `water-ops-platform-campaign`. | Entries exist only in the plant's logging channel after paste; the app holds nothing. | Accepted current state. Any egress feature is a decision-gated stage, not a patch. |
| W4 | **Inline `onclick` attributes couple HTML to window-exposed functions.** Menu cards and back buttons (e.g. lines 182, 219) call `showScreen`/`goHome` by name; renaming or scoping those functions breaks navigation with no console error until clicked. | Every navigation entry point. | Keep the two window exposures (I7) in sync with the HTML, or migrate to addEventListener as a deliberate refactor. |
| W5 | **Sludge header is hardcoded:** `sbBuild` emits `'Sludge Profile Lamella B'` (line 788). This encodes a plant fact — sludge profiling (locating the settled-solids layer in a clarifier) is done at Lamella B (an inclined-plate settler; see `water-treatment-domain-reference`). | If profiling ever happens at another unit, the output header lies. | Fine while the plant fact holds. If it needs to vary, a small config object like `ST` is the established pattern — do not add a free-text field. |
| W6 | **Duplicated "in Auto (lo - hi)" logic in `lBuild`.** The Dosing CO2 auto-flow-range block appears twice: PA (Post Anoxic — biological stage after the anoxic zone) branch lines 649–657, EQ A/B branch lines 664–672. Identical logic, different input ids (`l-pa-auto-*` vs `l-ab-auto-*`). | An edit to one branch (e.g. format change, empty-value handling) silently skips the other tank family. | Mirror edits across both branches, or extract a helper taking the two input ids. |

## Provenance and maintenance

All claims verified 2026-08-10 against the 890-line app file. Local WIP copy may be
named `index[23].html` (browser-download artifact — `water-ops-failure-archaeology`
owns that story); the globs below match either name. Re-verify before citing:

- File size/regions: `wc -l index*.html` (890) and `grep -n '<style>\|</style>\|<script>\|</script>' index*.html` (14/165/458/888)
- Screen toggle CSS: `grep -n '\.screen{display:none' index*.html` (32; the paired `.screen.active{display:block}` rule is line 33 — this pattern matches only the first)
- esc definition: `grep -n 'function esc' index*.html` (464)
- innerHTML inventory (must show 8 sites, dynamic ones with `map(esc)`): `grep -n 'innerHTML' index*.html`
- window exposure (exactly two): `grep -n 'window\.' index*.html | grep -vE 'scrollTo|window\.open'` (485–486)
- showScreen/goHome: `grep -n 'function showScreen\|function goHome' index*.html` (477, 484)
- SF/ST config: `grep -n 'var SF = {\|var ST = {' index*.html` (523, 535)
- Duplicated sample builder: `grep -n "lines=\['Time - '+tDisp, t.header" index*.html` (must show BOTH 611 and 827 — if only one, W1 was fixed; update this skill)
- trimEnd divergence: `grep -n 'trimEnd' index*.html` (829)
- null guards: `grep -n 'if(!text) return' index*.html` (860, 870)
- Actionbar toggle: `grep -n "toggle('hidden'" index*.html` (481)
- Sparse-render hint text: `grep -n 'Only the readings you enter appear in the copy' index*.html` (248, 355, 404)
- Hardcoded sludge header: `grep -n 'Sludge Profile Lamella B' index*.html` (788)
- Duplicated in-Auto blocks: `grep -n "in Auto ('+loF" index*.html` (must show two: 653, 668)
- No persistence: `grep -n 'localStorage\|sessionStorage\|indexedDB' index*.html` (must be empty)
- No external origins besides Telegram: `grep -n 'https://' index*.html` (only the t.me share URL, 871)
- SW cache name: `grep -n "CACHE" sw.js 2>/dev/null || echo "sw.js absent locally (lives in repo — see water-ops-pwa-and-mobile-playbook)"` (`waterops-v9` as of 2026-08-10)

If any command's output disagrees with a line number printed above, the file moved —
trust the grep, then update this skill.
