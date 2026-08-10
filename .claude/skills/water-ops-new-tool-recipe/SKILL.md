---
name: water-ops-new-tool-recipe
description: >
  Step-by-step verified recipe for adding a NEW tool (tool #5+) to the Water Ops
  single-file PWA (index.html) — new screen div, home menu card, time selector,
  build/render/clear functions, getPlainText and clear-handler branches, output
  format sign-off, doc updates, QA and SW cache bump. Load this when the task is
  "add a new tool / screen / logging form / formatter to Water Ops", or when
  evaluating the owner's candidate "in-app tool maker" idea. Do NOT load for
  changes to an existing tool (use water-ops-change-control +
  water-ops-architecture-contract) or for debugging (water-ops-debugging-playbook).
---

# Water Ops: Adding a New Tool (Recipe)

Water Ops is a single-file PWA (`index.html`, vanilla HTML/CSS/JS, no build step,
no framework, no backend, no tests) used by water treatment plant operators for
field logging. As of 2026-08-10 it has four tools, and the owner has said more
tools WILL be added — this recipe is the sanctioned path for tool #5 and beyond.
It generalizes exactly how the existing four are wired. Every code pattern below
is quoted or minimally adapted from the real file; re-verify with the commands in
"Provenance and maintenance" before trusting line-level details.

The four existing tools and their prefixes:

| Tool | Screen id | Prefix | Cadence | Build/Render/Clear |
|---|---|---|---|---|
| Quality Sampling | `screen-sample` | `s-` | 30-min | `sRender` / `sClear` (config-driven via `SF`/`ST`; no separate `sBuild` — see Step 7 trap) |
| EQ Log | `screen-log` | `l-` | 30-min | `lBuild` / `lRender` / `lClear` |
| Parshall Flume | `screen-flume` | `f-` | 30-min | `fBuild` / `fRender` / `fClear` |
| Sludge Blanket | `screen-sludge` | `sb-` | hourly | `sbBuild` / `sbRender` / `sbClear` |

The Parshall Flume tool (a Parshall flume is an open-channel flow-measurement
structure — one-line gloss; full definitions live in
`water-treatment-domain-reference`) is the cleanest template: simple fields, a
proper `fBuild` returning an array of lines, sparse rendering. Model new tools
on it unless you need Quality Sampling's config-driven field catalog.

Standing rule (owner directive, 2026-08-10): **security is a must** — every
render path escapes user input with `esc()` before `innerHTML`, no secrets in
the client file, no third-party scripts/CDNs. This recipe bakes that in at
Step 5; never skip it.

Throughout, `<name>` is your tool's screen name and `<p>` is your prefix
(examples use `ct-` / "Contact Tank" as an illustration — it is NOT a real
tool; confirm real tool names and every plant fact with the owner).

## The recipe

### Step 1 — Pick a short unique prefix

Existing prefixes: `s-`, `l-`, `f-`, `sb-`. Pick a new one (1–2 letters) that
collides with none of them, and use it on EVERY id and function you add:
element ids (`ct-time`, `ct-out`), functions (`ctBuild`, `ctRender`,
`ctClear`), radio `name` attributes (`ct-dose`). The prefix is the only
namespacing the app has — id collisions break `$()` lookups silently.

### Step 2 — Add the screen div

Insert a new top-level `<div id="screen-<name>" class="screen">` before the
shared action bar comment block. Copy the flume screen's skeleton (real markup
from index.html, adapted):

```html
<!-- ================================================================
     CONTACT TANK SCREEN
     ================================================================ -->
<div id="screen-contact" class="screen">
  <div class="wrap tool-wrap">
    <button class="back-btn" type="button" onclick="goHome()">&#8249; Menu</button>
    <header>
      <h1>Contact Tank</h1>
      <p class="sub">Enter a 30-minute reading. It formats the same way every time, ready to copy.</p>
    </header>

    <!-- Time selector: every tool has one, wired in Step 4 -->
    <div class="field">
      <div class="label"><span class="name">Time</span><span class="unit">military &middot; 30-min</span></div>
      <select class="inp" id="ct-time" aria-label="Time"></select>
    </div>

    <!-- Numeric field pattern (from f-flow / f-wl): -->
    <div class="field">
      <div class="label"><span class="name">Flow</span><span class="unit">GPM &middot; 0.0</span></div>
      <input class="inp" id="ct-flow" type="number" inputmode="decimal" step="0.1" min="0" placeholder="0.0" autocomplete="off">
    </div>

    <!-- Output preview: -->
    <div class="out-wrap">
      <div class="out-top">Field Entry</div>
      <pre id="ct-out" class="out"></pre>
    </div>
    <p class="hint">Only the readings you enter appear in the copy.</p>
  </div>
</div>
```

Field markup patterns available in the existing CSS (reuse, do not invent new
CSS unless necessary):

- **Numeric input**: as above. Conventions from the file: `step="0.01"` +
  placeholder `0.00` for 2-decimal fields (`f-wl`), `step="0.1"` + `0.0` for
  1-decimal (`f-ph`, which also carries `min="0" max="14"`),
  `inputmode="numeric"` for integers (`l-tot`). The `.unit` span shows unit +
  format hint (`GPM &middot; 0.0`).
- **Segmented radio** (`.seg`) — from the EQ Log Dosing CO2 group:

  ```html
  <div class="seg" role="radiogroup" aria-label="Dosing CO2">
    <input type="radio" name="ct-dose" id="ct-doseN" value="No"><label for="ct-doseN">No</label>
    <input type="radio" name="ct-dose" id="ct-doseA" value="in Auto"><label for="ct-doseA">in Auto</label>
    <input type="radio" name="ct-dose" id="ct-doseY" value="Yes"><label for="ct-doseY">Yes</label>
  </div>
  ```

  Read it with the existing helper `radioVal('ct-dose')` (returns the checked
  value or `null`).
- **"less than" toggle** (`.lt`) — ONLY if a field can read below detection
  limit ("less than" = below detection/measurement limit notation — see
  `water-treatment-domain-reference`). The existing implementation is
  DOM-built inside Quality Sampling's `sBuildRow` (a `.row` wrapping the
  `.inp` plus a `button.lt` with `aria-pressed` toggling and a state map
  `sLt`). If you need it outside the sampling screen, adapt that pattern; do
  not add it speculatively.
- **Multi-select buttons** (`.seg-multi` / `.slvl-btn`) — sludge-blanket
  valve-picker style, with your own toggle function modeled on `sbToggle`
  (which caps selection at 2 and keeps the array sorted).

### Step 3 — Add the home-screen menu card

Inside `#screen-home`'s `.menu-grid`, append (exact pattern from the file):

```html
<button class="menu-card" type="button" onclick="showScreen('screen-contact')">
  <div class="card-body">
    <div class="card-title">Contact Tank</div>
    <div class="card-desc">Record contact tank readings</div>
  </div>
  <span class="card-arrow">&#8250;</span>
</button>
```

`showScreen` and `goHome` are exposed on `window` specifically so these inline
`onclick`s work (see `window.showScreen = showScreen;` in the script). The
action bar shows/hides automatically: `showScreen` does
`actionBar.classList.toggle('hidden', id === 'screen-home')`, so a new tool
screen gets the Copy/Clear bar for free — no per-tool action-bar work.

### Step 4 — Wire the time selector

Verified behavior of the two builders (do not guess; the exact auto-select
rounding rule is owned by `water-ops-run-and-operate` §2):

- `buildTimeOptions(selId)` — fills the select with 48 half-hour options
  `0000`…`2330`, then auto-selects the nearest slot. Used by `s-time`,
  `l-time`, `f-time`.
- `buildHourlyOptions(selId)` — 24 hourly options `0000`…`2300`, auto-selects
  the current hour. Used only by `sb-time`.

Add one line next to the existing calls in the script:

```js
buildTimeOptions('ct-time');    // 30-min cadence tools
// or: buildHourlyOptions('ct-time');   // hourly tools
```

Match the `.unit` hint in the markup (`military &middot; 30-min` vs
`military &middot; hourly`) to whichever you pick.

### Step 5 — Write the three functions (Build / Render / Clear)

House pattern, verified against `fBuild`/`fRender`/`fClear` (and
`lBuild`/`sbBuild`):

**`<p>Build()` returns an ARRAY OF LINES.** Sparse-render rule: header lines
always; a reading line only when the operator entered that reading. Real code
(`fBuild`, quoted):

```js
function fBuild(){
  var time=$('f-time').value;
  var flow=fFmt1($('f-flow').value), wl=fFmt2($('f-wl').value);
  var ph=fFmt1($('f-ph').value), dox=fFmt1($('f-do').value), temp=fFmt1($('f-temp').value);
  var tDisp=time.substring(0,2)+':'+time.substring(2,4);
  var lines=['Time - '+tDisp, 'Parshall Flume'];
  if(flow!==null) lines.push('Flow = '+flow+' GPM');
  if(wl!==null)   lines.push('Water Level = '+wl+' FT');
  ...
  return lines;
}
```

Formatting helpers follow this shape (return `null` for empty/invalid so the
`if(x!==null)` sparse checks work):

```js
function fFmt1(v){ if(v===''||isNaN(parseFloat(v))) return null; return parseFloat(v).toFixed(1); }
function fFmt2(v){ if(v===''||isNaN(parseFloat(v))) return null; return parseFloat(v).toFixed(2); }
```

**`<p>Render()` escapes then joins.** Real code (`fRender`, quoted):

```js
function fRender(){
  $('f-out').innerHTML = fBuild().map(esc).join('\n');
}
```

`esc()` is the file's XSS escape — `function esc(s){ return
s.replace(/[&<>]/g, ...); }` — and applying it to every line before
`innerHTML` is a SECURITY invariant (owner: security is a must). Even
"numeric" inputs can carry arbitrary text on some platforms, and Quality
Sampling deliberately passes non-numeric entries through verbatim. Never write
user-derived strings into `innerHTML` without `esc()`; never "optimize" to
`textContent` without checking the empty-state pattern (see below), and never
skip the map.

Optional empty state (sludge pattern, `sbRender`): if nothing is entered yet,
show a static prompt instead — this is the ONE place raw HTML is written, and
it must remain a hardcoded literal, never interpolated:

```js
el.innerHTML='<span class="empty">Select a valve to begin.</span>';
```

**`<p>Clear()` resets ONLY this tool's fields** then re-renders. Real code
(`fClear`, quoted):

```js
function fClear(){
  $('f-flow').value=''; $('f-wl').value=''; $('f-ph').value='';
  $('f-do').value=''; $('f-temp').value='';
  fRender();
}
```

Note what Clear does NOT touch: the time select (it keeps its auto-picked
value) and other tools' state. If you have radios, uncheck them the way
`lClear` does:
`document.querySelectorAll('input[name="ct-dose"]').forEach(function(r){ r.checked=false; });`

### Step 6 — Register listeners

The house pattern (real code, flume section) — `change` on everything,
`input` additionally on number inputs so the preview updates per keystroke:

```js
['f-flow','f-wl','f-ph','f-do','f-temp','f-time'].forEach(function(id){
  var el=$(id);
  el.addEventListener('change', fRender);
  if(el.type==='number') el.addEventListener('input', fRender);
});
fRender();
```

Adapt the id list to your fields, include your `<p>-time` select, and call
`<p>Render()` once at the end so the preview is populated on load. Radios use
a separate `querySelectorAll('input[name="..."]')` + `change` loop (see the
`l-dose` handlers, which also toggle the conditional Auto Flow row).

### Step 7 — Add the getPlainText() branch

`getPlainText()` is what the Copy button (and the deprecated Telegram button)
reads. Add a branch following the pattern the EQ Log/flume/sludge branches use
(real code):

```js
if(currentScreen==='screen-flume'){
  return fBuild().join('\n');
}
```

So for your tool:

```js
if(currentScreen==='screen-contact'){
  return ctBuild().join('\n');
}
```

**RULE: build functions are the single source of truth for output text.**
Do NOT re-implement line-building inside `getPlainText` — the legacy sample
screen does exactly that (its `getPlainText` branch duplicates `sRender`'s
line assembly instead of sharing a build function), and that duplication is a
known drift trap: change one and not the other, and the on-screen preview and
the copied text silently diverge. That trap is documented in
`water-ops-architecture-contract`; new tools must not reproduce it.

Return `null` when there is nothing meaningful to copy (sludge does:
`if(sbSel.length===0) return null;`) — the copy handler treats `null` as
"do nothing".

### Step 8 — Add the clearBtn branch

Extend the shared clear handler (real code, with your branch appended):

```js
clearBtn.addEventListener('click', function(){
  if(currentScreen==='screen-sample') sClear();
  else if(currentScreen==='screen-log') lClear();
  else if(currentScreen==='screen-flume') fClear();
  else if(currentScreen==='screen-sludge') sbClear();
  else if(currentScreen==='screen-contact') ctClear();
});
```

Forgetting this branch means the Clear button silently does nothing on your
screen — no error, just a dead button.

### Step 9 — Design the output format WITH the owner, before coding it

The plain-text output is a contract: it feeds the plant's logging channel
(historically parsed by a Telegram bot; future direction is SharePoint/DB).
Get explicit sign-off on the exact format before writing `<p>Build()` —
output-text changes require owner approval per `water-ops-change-control`,
and `water-ops-run-and-operate` owns the format specs.

House conventions (verified as of 2026-08-10):

- **Header line: `Time - HH:MM`** (dash style) — used by Sampling, Flume,
  Sludge. EQ Log's `Time: HH:MM` (colon) is a legacy EXCEPTION; do not copy
  it into new tools, and do not "fix" EQ Log's either without sign-off.
- Second line is the tool/location name (`Parshall Flume`,
  `Sludge Profile Lamella B`, tank header). Sampling and Sludge insert a
  blank line after the header block; EQ Log and Flume do not — agree on which
  your tool uses.
- Reading lines use `Label = value unit` (Flume; EQ Log's readings — though
  EQ Log also uses `-` for its `Tank -` and `Dosing CO2 -` lines) or
  `Label - value` (Sampling) — pick one with the owner and be consistent.
- **Sparse-render rule**: only entered readings emit lines — this is the
  house rule. The UI hint text promises it ("Only the readings you enter
  appear in the copy") and every existing build function implements it (both
  verifiable in index.html). Do not emit placeholder lines for missing
  fields. For history, see `water-ops-failure-archaeology`.
- Confirm every plant fact in the format with the owner — e.g. `sbBuild`
  hardcodes the header `'Sludge Profile Lamella B'` because sludge profiling
  is genuinely done at Lamella B at this plant (a plant-specific truth owned
  by `water-treatment-domain-reference`), not because hardcoding is free.

### Step 10 — Update the docs that now lie, then QA and ship

A new tool invalidates several sibling skills until you update them:

1. Add the tool's exact output format + a hand-worked fixture to
   `water-ops-run-and-operate` (it owns the format specs).
2. Extend `water-ops-validation-and-qa`'s per-tool checklist with the new
   screen (render, copy-matches-spec character-for-character, clear, time
   auto-select).
3. Run the FULL pre-ship manual checklist from `water-ops-validation-and-qa`
   — there are no automated tests; the checklist is the test suite.
4. Bump the cache name in `sw.js` (e.g. `waterops-v9` → `waterops-v10`)
   before deploying — cache-first serving means users never see the new tool
   otherwise (`water-ops-pwa-and-mobile-playbook` owns the SW update cycle).

## Wiring checklist (all 10 must be checked before ship)

| # | Done | Item |
|---|---|---|
| 1 | [ ] | Unique prefix chosen; used on every id, function, and radio `name` |
| 2 | [ ] | `<div id="screen-<name>" class="screen">` added with `.wrap.tool-wrap`, back button, header, fields, `<pre id="<p>-out" class="out">` |
| 3 | [ ] | Home `.menu-card` added with `onclick="showScreen('screen-<name>')"` |
| 4 | [ ] | `buildTimeOptions('<p>-time')` or `buildHourlyOptions('<p>-time')` call added; `.unit` hint matches cadence |
| 5 | [ ] | `<p>Build()` returns array of lines; `<p>Render()` does `.map(esc).join('\n')` into `innerHTML`; `<p>Clear()` resets only this tool |
| 6 | [ ] | Listeners registered (`change` all fields, `input` on number inputs, time select included); initial `<p>Render()` called |
| 7 | [ ] | `getPlainText()` branch added, delegating to `<p>Build().join('\n')` — no duplicated line-building |
| 8 | [ ] | `clearBtn` handler branch added |
| 9 | [ ] | Output format signed off by owner; header uses `Time - HH:MM` dash style |
| 10 | [ ] | run-and-operate format spec + fixtures updated; validation-and-qa checklist extended; full pre-ship checklist run; `sw.js` cache name bumped |

## Common mistakes

- **Forgetting the `getPlainText` branch** — Copy (and Telegram) silently
  no-op: the handler does `if(!text) return;`, so a missing branch returns
  `null` and the button does nothing, with no error anywhere.
- **Forgetting the `clearBtn` branch** — Clear is a dead button on your
  screen only; easy to miss in testing.
- **Skipping `esc()`** in Render — an XSS hole and a violation of the owner's
  standing security rule. Every user-derived line goes through `esc()` before
  `innerHTML`, no exceptions.
- **Reusing another tool's prefix** — duplicate ids make `$()` return the
  first match; the wrong tool's field gets read or written with no error.
- **Hardcoding plant facts without owner confirmation** — `'Sludge Profile
  Lamella B'` is hardcoded because it is TRUE at this plant; your tool's tank
  names, valve counts, and units must be confirmed the same way, not guessed
  (cf. `water-treatment-domain-reference`). Cautionary tale: a GitHub Copilot
  PR (#1, 2026-07-26) added a pH field to EQ Sample Tank B and was reverted
  two days later — Tank B deliberately has no pH (details owned by
  `water-ops-failure-archaeology`).
- **Re-implementing line-building in `getPlainText`** — the legacy sample-
  screen duplication is a known drift trap, not a pattern to follow (Step 7).
- **Shipping without the `sw.js` cache bump** — installed users stay on the
  old cached app indefinitely (Step 10).

## CANDIDATE (idea, not commitment): the in-app "tool maker"

On 2026-08-10 the owner mused: "maybe a tool maker that would create things on
the app... would be cool." Status: candidate idea only. Nothing is designed,
scheduled, or promised — do not build it or scaffold for it without an
explicit owner decision.

What it would mean: config-driven tools — a tool defined as data (name,
header, cadence, field list with labels/units/formats) rendered by generic
code, instead of hand-wired HTML+JS per tool. The app already half-proves
this pattern: Quality Sampling's `SF` (field catalog) / `ST` (tank templates)
objects drive `sBuildRow`/`sRebuildFields`/`sRender` generically — one
rendering engine, five tank configurations. A tool maker would generalize
that from "configs hardcoded in the file" to "configs authored in the app".

Open questions that gate it:

- **Where do configs persist?** The app has no backend and (deliberately) no
  localStorage/IndexedDB today; a user-authored tool that vanishes on reload
  is useless, and adding persistence is its own decision
  (`water-ops-architecture-contract` owns the no-persistence stance).
- **Security of user-authored configs.** Config-driven rendering must treat
  every config string as untrusted input — same `esc()` discipline — and must
  never evaluate config content as code. Note that `sBuildRow` currently sets
  the field label via `nm.innerHTML=f.label` (safe today only because `SF` is
  hardcoded, e.g. for the `H₂O₂` label); user-authored labels
  through that path would be an XSS vector and would need `textContent` or
  escaping.
- **Output-contract governance**: hand-authored formats still need owner
  sign-off (Step 9); a tool maker doesn't remove that gate, it moves it.

If the owner green-lights it, the migration path is incremental: first refactor
new tools onto an SF/ST-style config (in-file), only then discuss in-app
authoring and persistence.

## When NOT to use this skill

- Modifying an EXISTING tool (fields, formats, behavior) → use
  `water-ops-change-control` + `water-ops-architecture-contract`.
- Debugging a broken tool or button → `water-ops-debugging-playbook`.
- Just running/serving/testing the app → `water-ops-run-and-operate`.
- Deploy/SW/cache questions → `water-ops-pwa-and-mobile-playbook`.

## Provenance and maintenance

All patterns verified against the app file on 2026-08-10 (890 lines; local
working copy is named `index[23].html` — a download artifact, deployed as
`index.html`; see `water-ops-failure-archaeology`. The `index*.html` glob below
matches either name). Re-verify before relying on this recipe:

- Prefix/function roster still four tools: `grep -n "function [slf]Build\|function sbBuild\|function [slf]Render\|function sbRender\|function [slf]Clear\|function sbClear" index*.html`
- Screen ids and menu cards: `grep -n "screen-\|menu-card" index*.html`
- Time builders and their call sites: `grep -n "buildTimeOptions\|buildHourlyOptions" index*.html`
- `getPlainText` branches (and whether the sample-screen duplication still exists): `grep -n "getPlainText\|currentScreen===" index*.html`
- Clear handler branches: `grep -n "clearBtn" index*.html`
- esc() discipline (every `innerHTML` write should be esc'd or a hardcoded literal): `grep -n "innerHTML" index*.html`
- Header-line style (dash vs EQ Log's colon exception): `grep -n "'Time - \|'Time: " index*.html`
- Action bar auto-toggle: `grep -n "actionBar.classList.toggle" index*.html`
- SW cache name to bump on ship: `grep -n "waterops-v" sw.js 2>/dev/null || echo "sw.js absent locally (lives in repo — see water-ops-pwa-and-mobile-playbook)"`
