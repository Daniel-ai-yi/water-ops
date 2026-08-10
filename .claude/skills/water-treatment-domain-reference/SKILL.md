---
name: water-treatment-domain-reference
description: Water-treatment domain knowledge for the Water Ops app. Load when you hit an unfamiliar term (EQ tank, Lamella, Post Anoxic, Parshall flume, sludge blanket, totalizer, HMI, Myron L, TSS, COD, NTU, H2O2 residual, dosing CO2, "less than") or when reasoning about what a field measures, its unit, its plausible operational range, or whether a value/tank name/format is a plant-specific truth. This is the single home for domain-term definitions; other Water Ops skills give one-line glosses and point here.
---

# Water Treatment Domain Reference (Water Ops)

This skill explains the water-treatment domain behind the Water Ops app: what the
plant is, what each tool's fields physically measure, their units, how the app
formats them, and what values are operationally plausible. Every domain term used
across the Water Ops skill library is defined ONCE, here.

**Two kinds of facts appear below — keep them straight:**

- **App-verified fact** — checked against `index.html` (the app file; the local
  working copy is named `index[23].html`, a browser-download artifact — see
  water-ops-failure-archaeology). These are ground truth for the code.
- **General domain background / operational guidance** — textbook water-treatment
  knowledge or realistic value ranges. The app does NOT enforce these (its only
  input enforcement is a few sparse HTML `min`/`max` attributes, e.g. pH inputs
  have `max="14"`). Use them for sanity-checking values and reviews, never as
  validation rules the code supposedly implements.

Volatile plant/app facts below are current as of 2026-08-10.

## When NOT to use this skill

| You need | Use instead |
|---|---|
| Exact output-format spec (the copy contract, line-by-line) | `water-ops-run-and-operate` (OWNS formats; this skill only cross-references) |
| Code structure, invariants, SF/ST config pattern | `water-ops-architecture-contract` |
| Procedure for changing or adding fields/formats | `water-ops-change-control` |
| Adding a whole new tool | `water-ops-new-tool-recipe` |
| PWA / service-worker / install behavior | `water-ops-pwa-and-mobile-playbook` |

## Plant context

Water Ops serves operators at a **NALCO Water** (an Ecolab company) industrial
**wastewater treatment** site. Operators walk rounds at fixed intervals, read
instruments and take grab samples, and log the readings as plain-text entries
(the app formats them; the text is pasted into company logs). Confidentiality of
this company data is the project's one hard rule.

**Logging cadence, as the app actually implements it (app-verified):**

- Quality Sampling, EQ Log, and Parshall Flume time dropdowns are built by
  `buildTimeOptions()` — 48 half-hour slots `0000`…`2330` (military time),
  auto-selecting the nearest slot (exact rounding rule owned by
  `water-ops-run-and-operate` §2).
- Sludge Blanket uses `buildHourlyOptions()` — 24 hourly slots `0000`…`2300`,
  auto-selecting the current hour. Sludge blanket checks are **hourly**; the
  other three tools are on a **30-minute** cadence.

## Process chain and tank roster (plant-specific truths)

General background: an industrial wastewater train commonly runs
equalization → chemical/physical clarification → biological polishing →
metered discharge. This plant's roster, exactly as encoded in the app's `ST`
object (app-verified — these five keys/headers are PLANT-SPECIFIC truths; only
the project owner can change the roster):

| ST key | App header text | What it is |
|---|---|---|
| `eqA` | `EQ Sample Tank A` | **EQ (equalization) tank** — a buffer tank that evens out incoming wastewater flow rate and pH before treatment, so downstream stages see a steady feed. |
| `eqB` | `EQ Sample Tank B` | Second EQ tank (parallel/alternate buffering). |
| `lamA` | `Lamella A` | **Lamella clarifier** — an inclined-plate settler: closely spaced tilted plates give a large effective settling area in a small footprint; suspended solids settle onto the plates and slide down as sludge. |
| `lamB` | `Lamella B` | Second lamella clarifier. Sludge-blanket profiling is done here (see Sludge Blanket section). |
| `paA` | `Post Anoxic A` | **Post Anoxic** — a biological treatment stage downstream of an anoxic (oxygen-free, nitrate-present) zone, used for nitrogen polishing. |

The EQ Log tool uses its own tank dropdown with values `A`, `B`, `PA` mapping to
"EQ Tank A", "EQ Tank B", "Post Anoxic A" (app-verified — note EQ Log says
"EQ Tank A" while Quality Sampling says "EQ Sample Tank A"; both spellings are
part of the output contract, do not "unify" them).

## Master glossary

Each term defined once. Other skills gloss and link here.

| Term | Definition |
|---|---|
| EQ (equalization) tank | Buffer tank smoothing incoming flow and pH before treatment. |
| Lamella (clarifier) | Inclined-plate settler removing suspended solids. |
| Post Anoxic | Biological polishing stage after an anoxic zone. |
| Parshall flume | Standardized open-channel constriction for flow measurement (see flume section). |
| Sludge blanket | The layer of settled solids at the bottom of a clarifier; its depth/level is monitored so sludge is wasted before it escapes with effluent. |
| TSS | Total Suspended Solids — mass of undissolved particles per volume, mg/L. Core solids-removal metric. |
| COD | Chemical Oxygen Demand — oxygen equivalent of chemically oxidizable matter, mg/L. Proxy for organic load. |
| Turbidity / NTU | Cloudiness of water; NTU = Nephelometric Turbidity Units (scattered-light measurement). Fast optical proxy for solids. |
| H2O2 (residual) | Hydrogen peroxide remaining in the water after peroxide dosing (used for oxidation/odor/pretreatment). Measured to confirm dose is consumed, not carried downstream. |
| Alkalinity | Water's acid-buffering capacity, mg/L (conventionally as CaCO3). Consumed by nitrification; low alkalinity lets pH crash. |
| Phosphate / Sulfate / Fluoride | Dissolved anions tracked for permit/process reasons, mg/L. |
| Dissolved Oxygen (DO) | O2 dissolved in water, mg/L. Aeration/biology health indicator. |
| HMI | Human-Machine Interface — the SCADA control-system panel/screen. "pH at HMI" = the online instrument's value as displayed on the panel. |
| Myron L | Brand of handheld water-quality meter. "pH with Myron L" = a manual cross-check reading taken to verify the HMI/online probe. Two pH lines exist so drift between online and handheld instruments is visible in the log. |
| Totalizer FM | Cumulative flow-meter total (FM = flow meter) — a running odometer-style volume count, not a rate. Units differ per tank (see EQ Log table). |
| Dosing CO2 | Carbon dioxide feed into the tank for pH control (CO2 forms carbonic acid, lowering pH without adding mineral acid). States: `No` (off), `in Auto` (controller modulates within a low–high flow range), `Yes` (on). |
| Discharge status | Whether the tank is currently releasing water downstream (`Discharging` / `Not Discharging`). |
| "less than" | Below the detection/quantification limit of the test — the lab/field kit can only certify the value is under some threshold. The app's `less than` toggle prefixes the entered number, e.g. `TSS - less than 4 mg/L`. |
| GPM | Gallons per minute (flow rate). |
| psi | Pounds per square inch (pressure). |
| Military time | 24-hour clock, `HHMM`; all app time dropdowns use it. |

## Field reference by tool

For each field: what it measures, its unit, and a realistic operational range.
**How the app formats each value (decimal places, padding, separators, line
shapes) is owned by `water-ops-run-and-operate` §3** — cross-reference it; the
tables below carry only the meaning, unit, and plant-fact notes this skill owns.

**Operational ranges are guidance only** — for sanity-checking entered/mocked
values and spotting typos in reviews. They are NOT app-enforced and NOT plant
permit limits (those are unknown to this repo). The only app-side enforcement
is sparse HTML attributes: `min="0"` on most numeric inputs and `max="14"` on
the EQ Log and Flume pH inputs; the dynamically built Quality Sampling inputs
have no min/max at all.

### 1. Quality Sampling (grab-sample results per tank)

Which fields appear depends on the selected tank (per-tank lists live in `ST`;
e.g. `eqA` shows only tss/ph/level/phosphate/sulfate/cod, `eqB` adds turbidity,
h2o2, alkalinity, fluoride but drops ph/level — app-verified). Per-field output
formatting — integer vs decimal places, the verbatim passthrough of non-numeric
input, the `less than ` prefix, the `Label - value unit` line shape, and which
fields support the "less than" toggle — is owned by `water-ops-run-and-operate`
§3.1; do not duplicate it here.

| Field | Measures | Unit in output | Operational range (guidance, not enforced) |
|---|---|---|---|
| TSS | Undissolved solids | mg/L | Raw/EQ influent: tens–hundreds. Clarifier effluent: <30 typical of good settling; very low results usually logged as "less than" the method limit. |
| pH | Acidity/basicity | (none) | 6.0–9.0 is the common operating/discharge band; outside 5–10 investigate; 0–14 is the physical scale. |
| Level | Tank level reading recorded with the sample | (none — unit not printed) | Plant-specific; compare against the same tank's recent entries. |
| Phosphate | Dissolved PO4 | mg/L | Industrial wastewater: 0–tens mg/L; polished effluent often single digits or "less than". |
| Sulfate | Dissolved SO4 | mg/L | Tens–hundreds mg/L is unremarkable for industrial water. |
| COD | Oxidizable organic load | mg/L | Influent: hundreds–thousands. Treated effluent: tens–low hundreds. |
| Fluoride | Dissolved F | mg/L | Commonly <10 mg/L; elevated values are a flag in industrial streams. |
| H2O2 | Peroxide residual | (none — unit not printed) | Ideally near 0–low single digits mg/L; a persistent residual means overdosing. |
| Alkalinity | Buffering capacity | mg/L | ~50–500 mg/L (as CaCO3) typical; <50 risks pH instability in biological stages. |
| Turbidity | Optical cloudiness | NTU | Clarified water: <10 NTU good, <1 excellent; raw water can be hundreds. |

### 2. EQ Log (30-minute tank round)

Two field sets, switched by tank: EQ A/B vs Post Anoxic (`PA`). One app fact
worth restating (verified 2026-08-10): Totalizer is a free string with no
padding (`lFmtTot` is a plain `String(v)` passthrough, unchanged in every
commit of the repo's history — do not add padding or parsing). All other
per-field formatting (decimal places, the zero-padded Tank Level, auto-flow
`0.0` defaults) is owned by `water-ops-run-and-operate` §3.2.

| Field | Tanks | Measures | Unit | Operational range (guidance) |
|---|---|---|---|---|
| pH | EQ A/B | Tank pH (single reading) | (none) | 6.0–9.0 typical; EQ tanks exist partly to hold this steady. |
| pH at HMI | PA only | Online probe value read off the SCADA panel | (none) | Same band as pH. |
| pH with Myron L | PA only | Handheld-meter cross-check of the same water | (none) | Should agree with HMI within ~0.2–0.3; larger gaps suggest probe fouling/calibration drift. |
| Tank Level | all | Water depth in tank | ft | Plant-specific; 0 to tank height. Sudden jumps between rounds are the thing to notice. |
| Dosing CO2 | all | CO2 feed state for pH control: `No` (off), `in Auto` (controller modulates within a low–high flow band), `Yes` (on) | — | Auto range is a controller flow band; lo ≤ hi, small positive numbers. |
| Discharge status | all | Whether tank is releasing downstream | — | — |
| Totalizer FM | all | Cumulative flow-meter volume (odometer-style) | **cu ft** for EQ A/B, **GALLONS (`gal`) for Post Anoxic** — app-verified: `lBuild` appends `(tank==='PA'?' gal':' cu ft')` (line ~675) | Monotonically increasing between rounds; a totalizer that went DOWN means a wrong reading or meter rollover. Do not "fix" the unit split — it reflects two different physical meters. |

### 3. Parshall Flume (30-minute discharge-point round)

| Field | Measures | Unit | Operational range (guidance) |
|---|---|---|---|
| Flow | Discharge flow rate, read off instrumentation | GPM | Plant-specific; compare to recent entries. Zero while "Discharging" elsewhere is suspicious. |
| Water Level | Upstream head (water depth) in the flume | ft | Typically fractions of a foot to ~2 ft in small flumes. |
| pH | Stream pH at the flume | (none) | 6.0–9.0 typical discharge band. |
| Dissolved Oxygen | O2 in the stream | mg/L | 2–10 mg/L; saturation is roughly 8–10 mg/L at these temperatures — values above ~12 are physically doubtful. |
| Temperature | Water temperature | °F | Roughly 40–95 °F for outdoor industrial streams; near-boiling or freezing values are typos. |

(Output casing and decimal places — e.g. the output prints `FT` and bare `F` —
are format-contract facts owned by `water-ops-run-and-operate` §3.3.)

**What a Parshall flume is (background):** a standardized open-channel
constriction — converging section, narrow throat, diverging section — whose
geometry forces critical flow, so the volumetric flow rate is a known function
of the upstream water depth ("head") alone. Under free-flow (non-submerged)
conditions the rating has the general form **Q = C·H^n**, where H is upstream
head and C and n depend on the throat width (n ≈ 1.52–1.60 for common sizes).
**Background only — NOT implemented.**

**Critical app fact (verified 2026-08-10): this app does NOT compute flow.**
The operator reads Flow (GPM) and Water Level (ft) from instrumentation and
types both in; there is no head→flow formula anywhere in `index.html`
(`fBuild`/`fFmt1`/`fFmt2` only format entered values). Do NOT "add the missing
formula" without explicit owner direction: the flume's throat width, unit
system of its rating, and free-flow vs submerged operating condition are all
unknown to this repo, so any formula you add would be a guess presented as an
instrument reading.

### 4. Sludge Blanket (hourly clarifier profile)

**Operational meaning:** the sludge blanket is the settled-solids layer in a
clarifier. If it rises too high, solids wash out over the effluent weir; the
hourly check tells operators when to pump sludge down. **In THIS plant** the
blanket is located by which of **4 sample valves** (mounted at different
heights) draws sludge when cracked open: the highest valve that pulls sludge
marks the blanket level.

App-verified behavior (`sbToggle`/`sbBuild`; exact output lines owned by
`water-ops-run-and-operate` §3.4):

| Field | Measures | Unit | Meaning of the selection |
|---|---|---|---|
| Valve selection | Blanket position among valves 1–4 | — | Exactly 1 valve = the blanket is at that valve's height; 2 valves (the app caps selection at two, kept sorted ascending) = the blanket surface sits between those two heights. |
| Sludge Pump Effluent Pressure | Discharge pressure of the sludge withdrawal pump | psi | Guidance: low tens of psi typical for sludge pumps; a sudden rise suggests line plugging, a drop suggests a losing prime/worn pump. |

**Plant fact:** the tool's output header is hardcoded to
`Sludge Profile Lamella B` (`sbBuild`, line ~788) — sludge-blanket profiling at
this plant is done at Lamella B specifically. This is intentional, not a bug;
only the owner can change it (e.g. if profiling expands to Lamella A).

## Why exact formats matter

Every tool's output text was historically parsed by a Telegram bot (now
deprecated, as of 2026-08-10) and is pasted into company logs — treat the exact
wording, punctuation, and spacing as a **contract**. That includes the oddities:
EQ Log's `Time:` (colon) vs every other tool's `Time -` (dash), `= ` vs ` - `
separators, `FT` vs `ft`, the zero-padded tank level, and the cu ft/gal split.
The full character-level spec is OWNED by `water-ops-run-and-operate` — cross-
reference it; do not duplicate or "normalize" formats from this skill. Any
format change needs owner sign-off via `water-ops-change-control`.

## Provenance and maintenance

App claims verified against `index.html` (local: `index[23].html`) on
2026-08-10. Re-verify before relying on line numbers or roster/format claims —
run from the project root; the `index*.html` glob matches the deployed name and
the local download-named copy alike:

```bash
grep -n "var ST" -A 6 index*.html          # tank roster keys + headers
grep -n "var SF" -A 11 index*.html         # sampling field catalog (units, dec/int, lt)
grep -n "Totalizer FM" index*.html         # totalizer line: cu ft vs gal ternary on tank==='PA'
grep -n "Sludge Profile Lamella B" index*.html   # hardcoded sludge header
grep -n "buildTimeOptions\|buildHourlyOptions" index*.html   # 30-min vs hourly cadence
grep -n 'max="14"' index*.html             # the only real input caps (pH)
grep -cn "toFixed\|padStart" index*.html   # formatting functions still present
grep -n "H\^\|Math.pow\|\*\*" index*.html  # must stay EMPTY of a head→flow formula
```
