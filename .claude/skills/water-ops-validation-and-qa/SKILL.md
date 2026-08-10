---
name: water-ops-validation-and-qa
description: Manual pre-ship verification for the Water Ops PWA. Load this BEFORE shipping, committing, or deploying ANY change to index.html or sw.js, or whenever asked "how do I test this", "did I break anything", "verify this change", or "is this safe to ship" in the Water Ops project. Contains the tiered pre-ship checklist (screens, per-tool output regression, formatting spot-checks, PWA deploy checks, security greps) and the hand-computation method for verifying formatting changes. There is NO automated test suite — this checklist IS the test suite.
---

# Water Ops — Validation and QA

## The reality

Water Ops has **no automated tests, no CI, no build step, and no test framework** (as of 2026-08-10). Do not invent one, do not assume one exists, do not "run the tests" — there are none. Verification is manual, in a browser, against this checklist.

Manual verification is genuinely load-bearing here because the app has a **duplicated render path**: on the Quality Sampling screen, the on-screen preview is built by `sRender()` while the text actually copied to the clipboard is built independently by `getPlainText()` — near-identical line-building logic written twice. A change to one that misses the other ships an app whose preview looks right while the copied entry is wrong. Only a human (or a model driving a browser) comparing preview against pasted clipboard text catches that.

The output blocks are a de-facto contract: operators paste them into the plant's logging channel, and format drift breaks downstream parsing/reading. Character-for-character comparison is the standard, not "looks about right".

Cross-references (one line each):
- **When to run this checklist**: `water-ops-change-control` defines when a change is allowed and mandates this checklist before ship.
- **What the outputs SHOULD be**: `water-ops-run-and-operate` owns the exact output-format spec and worked fixtures; this skill tells you how to verify against them.

**When NOT to use this skill**: diagnosing a failure you already found → `water-ops-debugging-playbook`. Deciding whether a change is allowed at all → `water-ops-change-control`. Looking up what an output block should contain → `water-ops-run-and-operate`. What a field means operationally (TSS, HMI, totalizer…) → `water-treatment-domain-reference`.

## How to run the app for testing

Open `index.html` directly (`file://` works for all UI and formatting checks). For clipboard and service-worker realism, serve it: `python3 -m http.server` and open `http://localhost:8000/` (localhost counts as a secure context). Details in `water-ops-run-and-operate`. Note: the working file may be locally named `index[23].html` (download artifact — see `water-ops-failure-archaeology`); it deploys as `index.html`.

## Pre-ship checklist

Run tiers top to bottom. Tier 1 on **every** change, however small. Tier 2 for any change that touches a tool's inputs, formatting, or output. Tier 3 for any change to a formatting function. Tier 4 when touching the PWA surface or deploying. Tier 5 (security) on **every** change — owner directive 2026-08-10: security is a must, always.

### Tier 1 — every change (smoke)

| # | Step | Pass condition |
|---|------|----------------|
| 1.1 | Load the page with DevTools console open | Page renders; **zero** console errors (a failed `sw.js` registration is silently caught by design and produces no error) |
| 1.2 | From home, open each of the 4 tools: Quality Sampling, EQ Log, Parshall Flume, Sludge Blanket | All 4 tool screens render; each shows its Time dropdown pre-selected to a sane nearby slot |
| 1.3 | On each tool, tap "‹ Menu" | Returns to home every time (5 screens total reachable: home + 4 tools) |
| 1.4 | Observe the bottom action bar (Telegram / Copy / Clear) | **Hidden on home, visible on every tool screen** |

### Tier 2 — per-tool output regression

For **each** of the 4 tools: enter the worked-fixture inputs (canonical fixtures live in `water-ops-run-and-operate`), then verify **both** the preview `<pre>` text **and** the actually-copied clipboard text match the expected block **character-for-character**. Method: click Copy, paste into a plain-text editor (TextEdit in plain-text mode, VS Code, `pbpaste` in a terminal), and compare against the preview and against the spec — spacing, `-` vs `=` vs `:`, unit strings, blank lines, everything.

| # | Tool | Step | Pass condition |
|---|------|------|----------------|
| 2.1 | Quality Sampling | Pick a tank, enter values in 2–3 fields, compare preview vs pasted copy | **preview === copied, character-for-character.** This is THE duplication trap (`sRender` vs `getPlainText`) — never skip. Known allowed difference: with a tank selected but zero readings entered, the copied text drops the trailing blank line (`getPlainText` calls `trimEnd()`); with at least one reading they must match exactly |
| 2.2 | Quality Sampling | On an `lt`-capable field (e.g. TSS), enter a value and toggle the "less than" button | Line reads `TSS - less than <value> mg/L`; toggle off removes the prefix; preview and copy both reflect it |
| 2.3 | Quality Sampling | Enter a non-plain numeric in a field (e.g. `1e5` in TSS — the inputs are `type="number"`, so plain alphabetic text like `TNTC` cannot be typed; exponent notation is the realistic probe) | Passed through verbatim (no `NaN`, no crash) — see Tier 3 |
| 2.4 | EQ Log | Select EQ Tank A or B, fill pH / Tank Level / dose / discharge / totalizer | Block matches spec: first line uses `Time:` with a **colon** (all other tools use `Time -` — a real, intentional inconsistency; do NOT "fix" it, see `water-ops-run-and-operate`); totalizer line ends ` cu ft` |
| 2.5 | EQ Log | Select **Post Anoxic A** | Field set swaps; output shows `pH = X at HMI` and `pH = Y with Myron L` lines (HMI = SCADA panel readout, Myron L = handheld meter — `water-treatment-domain-reference`); totalizer line ends ` gal` |
| 2.6 | EQ Log | Set Dosing CO₂ to "in Auto" (both A/B and PA branches) | Auto Flow Rate low/high inputs appear; output line is `Dosing CO2 - in Auto (lo - hi)` with each bound `toFixed(1)`; an empty bound renders as `0.0` |
| 2.7 | Parshall Flume | Enter flow, water level, pH, DO, temperature | `Flow = X.X GPM`, `Water Level = X.XX FT` (2 decimals), pH/DO/temp 1 decimal; empty fields produce no line |
| 2.8 | Sludge Blanket | Select **one** valve + psi | `Sludge Blanket is at Valve N` and `Sludge pump effluent pressure: X.X psi` under the hardcoded header `Sludge Profile Lamella B` |
| 2.9 | Sludge Blanket | Select **two** valves (tap a second; tapping a third is ignored — max 2) | Line becomes `Between valves: X and Y` with valves in ascending order regardless of tap order |
| 2.10 | Sludge Blanket | Enter psi only, no valve | Preview shows the psi line, but **Copy does nothing** (`getPlainText` returns null without a valve selected). Current behavior as of 2026-08-10 — verify unchanged unless your change intentionally alters it |
| 2.11 | Any two tools | Enter data in tool A, switch to tool B, enter data, press Clear on B, return to A | **Clear resets only the current tool** — tool A's entries are still there |
| 2.12 | Any tool | Press Copy twice | Button flashes "Copied!" then settles on "Copy entry" (it starts life as "Copy" — known pre-existing label inconsistency owned by `water-ops-debugging-playbook`; do not fail QA on it, do not silently "fix" it) |

### Tier 3 — formatting spot-checks by hand

Run these whenever a formatting function (`sFmtValue`, `lFmtPh`, `lFmtLevel`, `lFmtTot`, `fFmt1`, `fFmt2`, or a `*Build` function) was touched — and at least one of them on any Tier-2 run as a canary.

| # | Input | Where | Expected | Why it's a good probe |
|---|-------|-------|----------|----------------------|
| 3.1 | pH `7.25` | EQ Log pH (`lFmtPh`, `toFixed(1)`) | `pH = 7.3` | Rounding boundary. Caution: JS `toFixed` operates on binary floats, so some ties round "down" (`(1.005).toFixed(2)` is `"1.00"` because 1.005 is stored as 1.00499…). **Verify what the code actually produces in the console before asserting an expected value** — e.g. run `(7.25).toFixed(1)` yourself |
| 3.2 | Tank Level `7.5` | EQ Log (`lFmtLevel`) | `Tank Level = 07.5 ft` | Integer part zero-padded to 2 digits — the only padded field. Also try `100.26` → `100.3` (padStart is a no-op ≥2 digits, toFixed rounds). Do NOT use a float-tie input like `100.05` as a fixture — `(100.05).toFixed(1)` is actually `"100.0"` (binary-float tie, exactly the 3.1 caution; verified in console 2026-08-10) |
| 3.3 | Empty field | any tool | Line omitted entirely (sparse render), never `NaN`, never a placeholder | Sparse-render invariant |
| 3.4 | `0` | Flume Flow (`fFmt1`) | `Flow = 0.0 GPM` | Zero is a real reading, must not be treated as empty |
| 3.5 | `1.234` | Flume Water Level (`fFmt2`) | `Water Level = 1.23 FT` | >2 decimals truncate-by-rounding via `toFixed(2)` |
| 3.6 | `1e5` (valid in a number input, fails `sIsPlain`) | Quality Sampling TSS | `TSS - 1e5 mg/L` — verbatim passthrough | `sIsPlain` regex gates numeric formatting; strings that fail it pass through raw. The verbatim fallback exists only in Quality Sampling's `sFmtValue`; EQ Log/Flume formatters return null for non-numerics. (Inputs are `type="number"` everywhere, so exponent notation is the realistic enterable probe — plain text like `TNTC` is blocked by the browser) |
| 3.7 | Totalizer `007` | EQ Log (`lFmtTot`) | `Totalizer FM = 007 cu ft` — free string, NO reformatting | `lFmtTot` is a deliberate free-string passthrough — `String(v)`, no parsing, no padding, in every commit of the repo's history. Do not add padding or parsing (output contract; sign-off required). Verify: `grep -n "function lFmtTot" index*.html` |

### Tier 4 — PWA surface / deploying

Run when touching the manifest, icons, service worker, or doing any deploy. Procedure detail is owned by `water-ops-pwa-and-mobile-playbook` — cross-ref, don't improvise.

| # | Step | Pass condition |
|---|------|----------------|
| 4.1 | Check the deploy file set | `sw.js` is **present in the deploy set**. As of 2026-08-10 it is NOT in the local working directory (canonical v9 content lives in `water-ops-pwa-and-mobile-playbook`) — a deploy that omits it silently breaks offline |
| 4.2 | Open `sw.js` being deployed | Cache name (`waterops-vN`) has been **bumped** for this deploy — cache-first fetch means an unbumped name can leave devices on the old app forever |
| 4.3 | Install/update cycle on at least one real device | Fresh install works; an already-installed device picks up the new version after the SW update cycle (exact procedure: `water-ops-pwa-and-mobile-playbook`, including verifying the deployed origin's `sw.js` cache name before and after) |

### Tier 5 — SECURITY (every change; owner directive 2026-08-10: security is a must, always)

Copy-pasteable, run from the project directory (the `index*.html` glob matches the deployed name `index.html` and the local download-named copy alike):

| # | Command | Pass condition |
|---|---------|----------------|
| 5.1 | `grep -n "innerHTML" index*.html` | Every assignment of **user-derived** content goes through `esc()` (`.map(esc)` on the line arrays). As of 2026-08-10 there are exactly 8 hits, all safe — the authoritative per-line call-site inventory (which is which) is owned by `water-ops-architecture-contract`. Any NEW `innerHTML` sink of user input without `esc()` = fail, do not ship |
| 5.2 | `grep -niE "password|secret|token|apikey|api_key" index*.html` | **Zero hits.** The file is fully readable by anyone with the URL — no secrets/credentials ever belong in it. (Note: unescaped `|` pipes are required — with `-E`, a backslash-escaped `\|` becomes a LITERAL pipe and the check silently never matches) |
| 5.3 | `grep -nE "https?://" index*.html` | No NEW external origins. As of 2026-08-10 the only expected hit is the deprecated Telegram share link (`https://t.me/share/url` in the tgBtn handler); `sw.js` and `manifest.webmanifest` must have none. Any other external origin (CDN, analytics, font host, API) = fail — confidentiality rule, get owner sign-off first (`water-ops-change-control`) |

## How to verify a formatting change by hand-computation

The formatting functions are small and nearly pure — you can compute their output on paper faster than you can build tooling. Procedure:

1. **Pick boundary inputs**: empty string, `0`, a value needing rounding (`7.25`), a value needing padding (`7.5` for Tank Level), a value with more decimals than the format (`1.234`), and a non-numeric string.
2. **Trace each through the function's actual code** (read it — do not work from memory of what it "should" do), writing down the expected output string including units and spacing.
3. **Enter each input in the browser** and compare the preview line — then Copy, paste, and compare the clipboard line too (the duplication trap applies to Quality Sampling).
4. Any mismatch between your paper answer and the app is either your trace error or a real regression — resolve which by evaluating the expression in the DevTools console (e.g. `(7.25).toFixed(1)`).

**Fully worked example** — `lFmtLevel` with input `"7.5"`:

```
lFmtLevel("7.5")
  parseFloat("7.5")        -> 7.5
  (7.5).toFixed(1)          -> "7.5"
  "7.5".split('.')          -> ["7", "5"]
  "7".padStart(2,'0')       -> "07"
  return "07" + "." + "5"   -> "07.5"
Output line: "Tank Level = 07.5 ft"
```

Enter `7.5` in EQ Log → Tank Level; preview must show `Tank Level = 07.5 ft`; copied text must show the identical line. Also confirm the boundaries: empty input → the function returns null → the line is omitted; `100.26` → `toFixed(1)` → `"100.3"` → padStart no-op → `Tank Level = 100.3 ft`. (Pick non-tie inputs for fixtures: `(100.05).toFixed(1)` is `"100.0"`, not `"100.1"` — the Tier 3.1 float caution in action.)

## If/when a test suite is introduced (candidate — nothing here exists today)

Everything in this section is **candidate/open**: unbuilt, undecided, and it changes project structure, so it requires owner sign-off via `water-ops-change-control` before any of it is started.

The honest path for a no-build single-file project:

- **Natural first targets**: the formatting functions — `sFmtValue`, `lFmtPh`, `lFmtLevel`, `lFmtTot`, `fFmt1`, `fFmt2` — and the block builders (`lBuild`, `fBuild`, `sbBuild`, plus the sample-screen line building). They are nearly pure (a few read DOM/state directly; that is the main obstacle) and their fixtures are exactly the Tier 2/3 tables above.
- **Option A — extract to a testable module**: move the pure functions into a shared script or module imported by both the page and a test runner. Highest value (could also END the sRender/getPlainText duplication by making both call one builder), but it breaks the load-bearing "single file, no build" invariant — the biggest structural change, the strongest sign-off requirement.
- **Option B — self-test HTML page**: a separate `selftest.html` that inlines/duplicates the functions (or loads index.html in an iframe) and asserts fixture outputs, printing PASS/FAIL in the page. No tooling, no build, runnable on a phone — but duplicated logic can itself drift, recreating the very trap it is meant to catch.
- Either way, the fixture set is already written: the exact-output blocks in `water-ops-run-and-operate` plus the Tier 3 boundary table here.

Until an owner-approved suite exists, **this checklist is the test suite**. Do not claim tests passed unless you actually performed the manual steps.

## Provenance and maintenance

All claims verified against the app file on 2026-08-10 (locally named `index[23].html`, deploys as `index.html`; the `index*.html` glob below matches either). Re-verify before trusting, one line each:

- No test/CI tooling appeared: `ls package.json Makefile .github 2>/dev/null` (expect nothing)
- Duplication trap still exists: `grep -n "getPlainText\|function sRender" index*.html` (two independent sample-screen line builders)
- innerHTML sink count/safety: `grep -n "innerHTML" index*.html` (2026-08-10: 8 hits, all safe per Tier 5.1; inventory in `water-ops-architecture-contract`)
- No secrets: `grep -niE "password|secret|token|apikey|api_key" index*.html` (expect zero)
- External origins: `grep -nE "https?://" index*.html` (2026-08-10: only the t.me share link; recheck sw.js and manifest.webmanifest too)
- Formatting functions unchanged: `grep -n "sFmtValue\|lFmtPh\|lFmtLevel\|lFmtTot\|fFmt1\|fFmt2" index*.html`
- Sludge header still hardcoded: `grep -n "Sludge Profile Lamella B" index*.html`
- Copy-button label quirk: `grep -n "Copy entry" index*.html` (in `flashCopied`)
- SW cache name to bump: `grep -n "waterops-v" sw.js 2>/dev/null || echo "sw.js absent locally (lives in repo)"` (2026-08-10: `waterops-v9`; sw.js not in local dir — see pwa-and-mobile-playbook)
