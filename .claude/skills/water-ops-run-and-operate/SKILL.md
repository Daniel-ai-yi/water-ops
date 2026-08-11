---
name: water-ops-run-and-operate
description: >-
  Load this skill to run, serve, demo, or manually test the Water Ops app
  locally (file:// vs python3 http.server vs deployed HTTPS); to understand how
  plant operators actually use the app in the field (pick tool, enter readings,
  Copy, paste into the log channel); or to check what the copied output text
  must look like or WHY it is formatted that way. This skill OWNS the exact
  per-tool output contract (Quality Sampling, EQ Log, Parshall Flume, Sludge
  Blanket) including every separator, decimal-place, zero-padding, and header
  rule — including the EQ Log "Time:" colon vs "Time -" dash inconsistency and
  other format oddities — plus worked input->output fixtures and the
  time-dropdown auto-select rounding rule. Load it before touching any string
  that ends up in the copied text. NOT for debugging broken behavior
  (water-ops-debugging-playbook) or deploy/SW mechanics
  (water-ops-pwa-and-mobile-playbook).
---

# Water Ops — run, operate, and the output contract

Water Ops is a single-file PWA (`index.html`, vanilla HTML/CSS/JS, no build
step, no backend) used by water-treatment plant operators for field logging.
Four tools — Quality Sampling, EQ Log, Parshall Flume, Sludge Blanket — each
format entered readings into a fixed plain-text block the operator copies into
the company log channel. This skill tells you how to run it locally, how
operators use it, and — the part this skill owns — the exact text each tool
must produce.

Note on the local file: the canonical working copy on disk may carry a
browser-download name like `index[23].html` (see water-ops-failure-archaeology
for why). When deployed it is always `index.html`; this skill calls it
`index.html` throughout.

## 1. Running locally

The app is fully static. There is no build step, no npm, no test suite (see
water-ops-build-and-env). Two ways to run it:

**Option A — open directly (pure UI/logic checks only):**

```bash
open index.html        # macOS; or just double-click it
```

**Option B — serve it (realistic: secure context for service worker + clipboard):**

```bash
# stage under the deployed name if your local copy has a download suffix
mkdir -p /tmp/waterops
cp 'index[23].html' /tmp/waterops/index.html
cp manifest.webmanifest icon-192.png icon-512.png apple-touch-icon-180.png /tmp/waterops/ 2>/dev/null
cd /tmp/waterops && python3 -m http.server 8000
# then open http://localhost:8000
```

`localhost` counts as a secure context, so `navigator.clipboard` and service
worker registration behave as they do in production.

| Capability | `file://` | `http://localhost:8000` | Deployed HTTPS |
|---|---|---|---|
| Screens, time dropdowns, live output preview | Yes | Yes | Yes |
| Output-format verification (read the preview `<pre>`) | Yes | Yes | Yes |
| Copy button | Unreliable — `navigator.clipboard` availability on `file://` is browser-dependent; the code falls back to `fallbackCopy` (hidden textarea + `document.execCommand('copy')`). Do not validate clipboard behavior here. | Yes (async clipboard, real code path) | Yes |
| Service worker / offline | No — SW registration requires http(s); the `register('./sw.js').catch(function(){})` fails silently | Yes, if `sw.js` is present (as of 2026-08-10 `sw.js` is NOT in the local working dir — see water-ops-pwa-and-mobile-playbook) | Yes |
| Home-screen install / PWA update cycle | No | Partial (varies by browser) | Yes — the only real test |

For deep PWA testing (install, offline, cache-name-bump update cycle, the
stale-SW trap on the live origin) load **water-ops-pwa-and-mobile-playbook**.

Deploying means copying **6 files** to a static host: `index.html`,
`manifest.webmanifest`, `sw.js`, `icon-192.png`, `icon-512.png`,
`apple-touch-icon-180.png` (file list owned by water-ops-build-and-env) — and
bumping the SW cache name (see water-ops-change-control).

## 2. Operator workflow (what the app is for)

A plant operator on rounds:

1. Launches Water Ops from the phone home-screen icon (or a browser tab).
2. Picks a tool from the dashboard's U3 section. The action bar (Copy / Clear)
   appears on every tool screen and is hidden on home.
3. The Time dropdown has already auto-selected the nearest slot:
   - Quality Sampling, EQ Log, Parshall Flume use 30-minute slots
     (`buildTimeOptions`, 48 options `0000`…`2330`): current minute < 15 →
     `:00` of this hour; < 45 → `:30`; otherwise `:00` of the NEXT hour
     (wraps past midnight).
   - Sludge Blanket uses hourly slots (`buildHourlyOptions`, 24 options): it
     selects the CURRENT hour's `:00` — a floor, not nearest-rounding.
4. Enters ONLY the readings actually taken. The render is sparse: a field
   left blank produces no output line at all (no placeholder zeros).
5. Watches the live "Field Entry" preview, taps **Copy**, and pastes the text
   into the company log channel.

**Telegram button — REMOVED 2026-08-10** (commit 8ee7a8d). The action bar is
now Copy / Clear only. The Telegram bot that historically parsed entries is
retired; the exact-format contract below still stands because entries are
pasted into company records. History: water-ops-failure-archaeology.

**Login screen — since 2026-08-10 the app opens on a login screen**, not the
dashboard. On non-Azure hosts it shows labeled "preview mode" with a Continue
button (tap it to reach the tools). Details: water-ops-auth-activation.

**Nothing is saved.** There is no localStorage/sessionStorage/IndexedDB
anywhere. Leaving the page or reloading loses all entries. As of 2026-08-10
this is by design, not a bug (water-ops-architecture-contract owns this fact).
Switching between tools within a session does NOT lose entries — see section 4.

Domain terms (EQ tank, Lamella, Post Anoxic, HMI, Myron L, totalizer, TSS,
COD, NTU, sludge blanket, "less than") are glossed inline below once each;
full definitions live in **water-treatment-domain-reference**.

## 3. THE OUTPUT CONTRACT (owned here)

**Contract rule:** these strings were historically machine-parsed by a
Telegram bot and are pasted into company records. **Never alter any produced
text — not a separator, a unit, a decimal place, or the colon-vs-dash
inconsistency — without owner sign-off** (procedure in
water-ops-change-control). validation-and-qa checks copied output
character-for-character against this section.

Universal facts (verified against the code 2026-08-10):

- All separators are plain ASCII: `" - "`, `" = "`, `": "`. No en-dashes, no
  Unicode subscripts — the output says `Dosing CO2`, not `CO₂` (the UI label
  uses the subscript; the output does not).
- Time is rendered `HH:MM` (24-hour, zero-padded) from the dropdown value.
- Lines are joined with `\n`. The preview `<pre>` and the copied text are
  built from the same line arrays for EQ Log / Flume / Sludge; for Quality
  Sampling the copy path is a DUPLICATE of the preview logic inside
  `getPlainText()` (drift trap — owned by water-ops-architecture-contract)
  and additionally applies `.trimEnd()` (see section 4).

### 3.1 Quality Sampling

Shape (blank line after the two header lines; only entered fields appear, in
the tank template's fixed order):

```
Time - HH:MM
<Tank header>

<FIELD> - <value><unit>
...
```

Tank headers and field order (from the `ST` catalog):

| Dropdown choice | Header line | Fields, in output order |
|---|---|---|
| EQ Tank A | `EQ Sample Tank A` | TSS, pH, Level, Phosphate, Sulfate, COD |
| EQ Tank B | `EQ Sample Tank B` | TSS, Turbidity, Phosphate, Sulfate, H2O2, Alkalinity, COD, Fluoride |
| Lamella A | `Lamella A` | TSS, Turbidity, pH, Level, Phosphate, Sulfate, H2O2, Alkalinity, COD |
| Lamella B | `Lamella B` | TSS, Turbidity, pH, Level, Phosphate, Sulfate, H2O2, Alkalinity, COD |
| Post Anoxic A | `Post Anoxic A` | same 9 fields as Lamella A/B |

Per-field value formatting (from the `SF` catalog; separator is always
`<name> - <value>`):

| Output name | Format | Unit suffix | "less than" toggle? |
|---|---|---|---|
| `TSS` | integer (`String(Number(s))` — strips leading zeros) | ` mg/L` | yes |
| `pH` | `toFixed(2)` | none | no |
| `Level` | `toFixed(2)` | none | no |
| `Phosphate` | integer | ` mg/L` | yes |
| `Sulfate` | integer | ` mg/L` | yes |
| `COD` | integer | ` mg/L` | yes |
| `Fluoride` | `toFixed(1)` | ` mg/L` | yes |
| `H2O2` | `toFixed(1)` | none | no |
| `Alkalinity` | integer | ` mg/L` | yes |
| `Turbidity` | integer | ` NTU` | yes |

Value rules (`sIsPlain` / `sFmtValue`):

- Empty field → no line.
- Input matching `^-?(\d+\.?\d*|\.\d+)$` is formatted per the table.
- Anything else is passed through **verbatim** (defensive path; the inputs are
  `type=number` so browsers block most non-numeric typing, but e.g. `1e5` —
  which number inputs accept — fails the regex and is emitted as-is,
  unit still appended).
- If the field's "less than" button is toggled (marks a reading below the
  detection limit), the value is prefixed `less than ` — e.g.
  `TSS - less than 4 mg/L`.

### 3.2 EQ Log

**Verified inconsistency:** EQ Log's time line uses a COLON — `Time: HH:MM` —
while every other tool uses `Time - HH:MM`. This is real, shipped behavior.
Do NOT "fix" it without owner sign-off; downstream records contain it.

Shape (no blank line after the time line; every line below Time is optional
and appears only when its field is set):

```
Time: HH:MM
Tank - EQ Tank A              (or B; Post Anoxic: Tank - Post Anoxic A)
pH = 7.2                      (toFixed(1))
Tank Level = 07.5 ft          (toFixed(1), integer part zero-padded to 2 digits)
Dosing CO2 - in Auto (1.0 - 2.5)
Discharging                   (or: Not Discharging — bare line, no prefix)
Totalizer FM = 123456 cu ft   (Post Anoxic: gal)
```

Rules:

- **Post Anoxic A** replaces the single pH line with two:
  `pH = <v> at HMI` (HMI = the SCADA panel readout) and
  `pH = <v> with Myron L` (Myron L = handheld verification meter). Either
  appears independently if entered.
- **Tank Level**: `toFixed(1)`, then the integer part is `padStart(2,'0')` —
  `7.5` → `07.5 ft`, `12` → `12.0 ft`.
- **Dosing CO2** (CO2 feed for pH control) has three radio states: `No`,
  `in Auto`, `Yes`. `No`/`Yes` emit `Dosing CO2 - No` / `Dosing CO2 - Yes`.
  `in Auto` emits `Dosing CO2 - in Auto (<lo> - <hi>)` where each bound is
  `toFixed(1)` of the entered value, and an **unentered bound defaults to
  `0.0`** — both empty gives `Dosing CO2 - in Auto (0.0 - 0.0)`.
- **Discharge status** emits a bare `Discharging` or `Not Discharging` line.
- **Totalizer FM** (cumulative flow-meter reading) is a free string — no
  padding, no reformatting: `Totalizer FM = <raw> cu ft` for EQ A/B,
  `Totalizer FM = <raw> gal` for Post Anoxic.
- No tank selected → no `Tank -` line (the rest still emit if set).

### 3.3 Parshall Flume

(Parshall flume = open-channel flow-measurement structure; the operator READS
flow in GPM from an instrument — the app computes nothing.)

Fixed two-line header always present; each reading line only when entered:

```
Time - HH:MM
Parshall Flume
Flow = 842.0 GPM              (toFixed(1))
Water Level = 1.20 FT         (toFixed(2))
pH = 6.8                      (toFixed(1))
Dissolved Oxygen = 8.5 mg/L   (toFixed(1))
Temperature = 68.0 F          (toFixed(1))
```

Note the casing: `GPM`, `FT` (upper), `mg/L`, bare `F` (no degree sign).

### 3.4 Sludge Blanket

(Sludge blanket = settled-solids layer in a clarifier, located by which of 4
sample valves draws sludge. The header hardcodes `Sludge Profile Lamella B`
because profiling is done at Lamella B — a plant fact owned by
water-treatment-domain-reference.)

Shape (blank line after the two header lines):

```
Time - HH:MM
Sludge Profile Lamella B

Sludge Blanket is at Valve 2
Sludge pump effluent pressure: 12.0 psi
```

Rules:

- 1 valve selected → `Sludge Blanket is at Valve <n>`.
- 2 valves selected → `Between valves: <a> and <b>` — always ascending; the
  selection logic caps at 2 valves and keeps them sorted.
- Pressure line only if entered: `Sludge pump effluent pressure: <toFixed(1)> psi`
  — note this line's separator is a COLON, not `=`.

## 4. Empty and edge behaviors (verified 2026-08-10)

| Behavior | Detail |
|---|---|
| Copy is a **silent no-op** when `getPlainText()` returns null | Quality Sampling with no tank selected; Sludge Blanket with no valve selected — **even if a psi value was entered** (the copy guard checks only the valve selection; the preview will show the psi line but Copy does nothing). No error, no toast. |
| EQ Log copies with nothing entered | Copies just `Time: HH:MM` (plus any lines that are set). |
| Parshall Flume copies with nothing entered | Copies the two header lines `Time - HH:MM` + `Parshall Flume`. |
| Quality Sampling preview vs copy differ by one trailing blank line | The copy path applies `.trimEnd()`; with a tank selected but zero fields entered, the preview shows `Time / header / blank` while the copy is `Time / header`. With any field entered they match. |
| Clear resets ONLY the current tool | The Clear handler branches on the current screen: `sClear` / `lClear` / `fClear` / `sbClear`. Other tools' entries survive. |
| Switching tools does NOT clear anything | Screen switching only toggles CSS classes; all four tools' inputs keep their state until Clear or page reload. |
| Reload/close loses everything | No persistence of any kind (by design as of 2026-08-10). |
| Copy button label drifts | Starts as `Copy`, but after the first copy the "Copied!" flash restores it to `Copy entry` permanently. Known inconsistency — owned by water-ops-debugging-playbook. |

## 5. Worked examples (regression fixtures)

Compute-by-hand fixtures; water-ops-validation-and-qa uses these
character-for-character. Assume the stated Time is selected in the dropdown.

### Quality Sampling — EQ Tank A, 07:30

Inputs: TSS `4` with "less than" toggled; pH `7.2`; Level `3.5`;
Sulfate `250`; Phosphate and COD left blank.

```
Time - 07:30
EQ Sample Tank A

TSS - less than 4 mg/L
pH - 7.20
Level - 3.50
Sulfate - 250 mg/L
```

### EQ Log — EQ Tank A, 14:00

Inputs: pH `7.23`; Tank Level `7.5`; Dosing CO2 = in Auto with low `1`,
high `2.5`; Discharge = Discharging; Totalizer `123456`.

```
Time: 14:00
Tank - EQ Tank A
pH = 7.2
Tank Level = 07.5 ft
Dosing CO2 - in Auto (1.0 - 2.5)
Discharging
Totalizer FM = 123456 cu ft
```

### EQ Log — Post Anoxic A, 09:30

Inputs: pH at HMI `7.2`; pH with Myron L `7.14`; Tank Level `12`;
Dosing CO2 = No; Discharge = Not; Totalizer `98765`.

```
Time: 09:30
Tank - Post Anoxic A
pH = 7.2 at HMI
pH = 7.1 with Myron L
Tank Level = 12.0 ft
Dosing CO2 - No
Not Discharging
Totalizer FM = 98765 gal
```

### Parshall Flume — 10:00

Inputs: Flow `842`; Water Level `1.2`; pH `6.8`; Dissolved Oxygen `8.5`;
Temperature `68`.

```
Time - 10:00
Parshall Flume
Flow = 842.0 GPM
Water Level = 1.20 FT
pH = 6.8
Dissolved Oxygen = 8.5 mg/L
Temperature = 68.0 F
```

### Sludge Blanket — 13:00

Inputs: valves 2 and 3 selected; pressure `12`.

```
Time - 13:00
Sludge Profile Lamella B

Between valves: 2 and 3
Sludge pump effluent pressure: 12.0 psi
```

Single-valve variant (only valve 2 selected): the valve line becomes
`Sludge Blanket is at Valve 2`.

## When NOT to use this skill

| You want to… | Use instead |
|---|---|
| Verify a change before shipping (checklists; it references the fixtures above) | water-ops-validation-and-qa |
| Understand what a field/unit means operationally, realistic ranges | water-treatment-domain-reference |
| Install to home screen, offline behavior, SW cache updates, stale-app debugging | water-ops-pwa-and-mobile-playbook |
| Make a change safely / get output-format sign-off | water-ops-change-control |
| Remove Telegram or plan hosting/login/database work | water-ops-platform-campaign |

## Provenance and maintenance

All claims verified against the app file on 2026-08-10. Re-verify with these
(run from the project root; the `index*.html` glob matches the deployed name
and the local download-named copy alike):

```bash
grep -n "'Time: '" index*.html          # EQ Log colon (expect 1 hit, in lBuild)
grep -n "'Time - '" index*.html         # dash form (expect 4: sRender, fBuild, sbBuild, getPlainText)
grep -n "Sludge Profile Lamella B" index*.html   # hardcoded sludge header
grep -n "toFixed" index*.html           # all decimal-place rules
grep -n "padStart(2,'0')" index*.html   # pad2 + Tank Level zero-padding (lFmtLevel)
grep -n "in Auto (" index*.html         # CO2 auto-range line + '0.0' defaults nearby
grep -n "Totalizer FM" index*.html      # cu ft vs gal branch
grep -n "sIsPlain\|less than " index*.html   # numeric-vs-verbatim + lt prefix
grep -n "getPlainText\|trimEnd" index*.html  # copy path + sample trimEnd
grep -n "sbSel.length" index*.html      # sludge copy guard (null when no valve)
grep -n "t.me/share" index*.html        # Telegram handler (deprecated; removal sanctioned)
grep -n "localStorage\|sessionStorage\|indexedDB" index*.html  # expect ZERO hits (no persistence)
grep -n "m<15\|m<45" index*.html        # 30-min slot auto-select rule
grep -cn "http.server\|npm\|webpack" index*.html  # expect 0 — still no build tooling
```
