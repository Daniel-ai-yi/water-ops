---
name: water-ops-failure-archaeology
description: >
  Water Ops project history chronicle: what was tried, what was reverted, what
  was deliberately settled, and what is still a live risk. Load this when
  asking "has this been tried before?", "why is X this way?", or "why does
  this field list look asymmetric?"; BEFORE reverting, redesigning, or
  "fixing" anything that looks odd (EQ Sample Tank B's missing pH field,
  output headers that differ from screen titles, the index[23].html filename,
  Telegram code, service-worker cache-name bumps); or when mining git history
  of the Daniel-ai-yi/water-ops repo. Do NOT load for current-bug triage (use
  water-ops-debugging-playbook) or for current architecture rationale (use
  water-ops-architecture-contract).
---

# Water Ops — Failure Archaeology

The evolution chronicle of the Water Ops app: every settled battle, its
evidence, and its status. Read this before re-fighting anything. All facts
verified against the canonical git repo and the current app file as of
**2026-08-10**.

**What Water Ops is (one line):** a single-file PWA (progressive web app —
installable offline-capable web page) used by water-treatment plant operators
to format field readings into copyable log text. Domain terms (EQ tank,
Lamella, Post Anoxic, Parshall flume, sludge blanket, totalizer, HMI, Myron L)
are glossed in `water-treatment-domain-reference`.

**Canonical repo:** https://github.com/Daniel-ai-yi/water-ops (public,
anonymous clone works). 18 commits, single `main` branch, 2026-07-06 →
2026-08-05 EDT. HEAD is `a57840c`, and its `index.html` is **byte-identical**
to the current local working file — the repo is current; there is no drift.

**Predecessor era (one line, out of scope):** a single-tool prototype predates
this repo (June 2026); its repository was explicitly disowned by the owner for
this project (2026-08-10). Cite nothing from it.

## When NOT to use this skill

| You need... | Go to |
|---|---|
| To triage a bug happening right now | `water-ops-debugging-playbook` |
| Why the CURRENT architecture is the way it is (invariants, load-bearing decisions) | `water-ops-architecture-contract` |
| The sanctioned-work list / roadmap stages | `water-ops-platform-campaign` |
| SW lifecycle mechanics and deploy/update procedure | `water-ops-pwa-and-mobile-playbook` |
| Historical context: has this been tried, why was it undone, is this deliberate | **stay here** |

## Commit ledger (all 18 commits, main branch — verified 2026-08-10)

Most commits are titled "Add files via upload": the app is edited elsewhere,
downloaded, and uploaded through the GitHub web UI, so commit messages carry
almost no information — this ledger is the readable history.

| Hash | Date (EDT) | What it actually did |
|---|---|---|
| `60b34ed` | 07-06 | Create `.gitkeep` (repo born) |
| `1a9beca` | 07-06 | First upload: Water Ops ALREADY multi-tool — 647-line `index.html` with 4 screens (home, EQ Sampling, EQ Log, Parshall Flume; NO sludge yet), manifest "Water Ops", `sw.js` cache `waterops-v1`, 3 icons |
| `a3998d0` | 07-06 | +11/−1: added the Telegram button (`tgBtn`, `.tg` CSS, `t.me/share/url` handler); Copy button label "Copy entry" → "Copy" |
| `a6b9fd2` | 07-20 | Fluoride field added to the sampling field catalog and to eqB's field list |
| `79b454d` | 07-22 | EQ Log tank picker radio(A/B) → `<select>` adding Post Anoxic A; PA field group (pH at HMI, pH with Myron L); no cache bump |
| `4291111` | 07-22 | Standalone cache bump `waterops-v1` → `v2` (sw.js only) |
| `8186b73` | 07-22 | `paA` ("Post Anoxic A") tank added to the sampling templates (plus a missing-comma fix on `lamB`) |
| `490bdbc` | 07-23 | Sludge Blanket screen added (5th screen, +124 lines in index.html); no cache bump |
| `da0fef8` | 07-23 | Standalone cache bump `v2` → `v3` (sw.js only) |
| `7991386` | 07-23 | Dosing CO2 gained "in Auto" option with "(lo - hi)" range output; totalizer unit label got id `l-tot-unit` (gal vs cu ft switching); cache bump `v3` → `v4` in the same commit |
| `d6afb7f` | 07-26 | "Initial plan" — GitHub Copilot coding agent's empty plan commit (no file changes) |
| `5c9bc82` | 07-26 | Copilot agent: "feat: add pH input field to EQ Sample Tank B" — added `'ph'` to eqB's field list (1-line change) |
| `01fb047` | 07-26 | Merge pull request #1 (the Copilot change lands on main; no cache bump) |
| `6074323` | 07-28 | **REVERT of PR #1**: `'ph'` removed from eqB; cache bump `v4` → `v5` |
| `b77839c` | 08-03 | `'turbidity'` added to eqB instead; cache bump `v5` → `v6` |
| `9db12dc` | 08-03 | Sampling output headers "EQ Sample Lamella A/B" → "Lamella A/B"; cache bump `v6` → `v7` |
| `6b47c87` | 08-04 | Tool renamed "EQ Sampling" → "Quality Sampling" everywhere INCLUDING output headers → "Quality Sample Tank A/B"; cache bump `v7` → `v8` |
| `a57840c` | 08-05 (HEAD) | Output headers REVERTED to "EQ Sample Tank A/B" while the screen title stays "Quality Sampling"; cache bump `v8` → `v9` |

## The chronicle

Format: **Symptom/decision → Root cause → Evidence → Status.**
Status legend: **settled** (do not re-fight), **still-live risk** (unresolved,
handle with care), **sanctioned work** (removal/change is approved).

### E1. The PR #1 revert — EQ Sample Tank B has NO pH field, on purpose

- **Symptom/decision:** eqB's sampling field list looks asymmetric — eqA has
  `ph`, eqB does not. A GitHub Copilot coding agent noticed the same thing and
  "fixed" it: PR #1 added `'ph'` to eqB. Two days later the owner reverted it.
- **Root cause:** the `ST` tank templates encode **plant facts** — which
  readings are actually taken at each tank — not code symmetry. pH is not
  sampled at EQ Sample Tank B. The AI agent's plausible-looking addition was
  wrong. What Tank B actually needed came later: turbidity.
- **Evidence:** `5c9bc82` (2026-07-26, author `copilot-swe-agent[bot]`) added
  `'ph'` to eqB; merged in `01fb047`; `6074323` (2026-07-28) removed it;
  `b77839c` (2026-08-03) added `'turbidity'` to eqB instead. Current eqB
  field list (HEAD, byte-identical to local file):
  `['tss','turbidity','phosphate','sulfate','h2o2','alkalinity','cod','fluoride']` — no `ph`.
- **Status:** **settled.** Never add, remove, or reorder fields in the `ST`
  templates to make tanks look consistent. Field lists change only on
  operator/plant instruction. This is the project's canonical example of an
  AI-generated change that was reverted — check this chronicle before
  "completing" an apparent omission.

### E2. Output-header tuning — output text is contract text, not the UI title

- **Symptom/decision:** the sampling screen is titled "Quality Sampling" but
  its output headers say "EQ Sample Tank A/B". This mismatch is deliberate,
  reached through three commits of tuning including one revert.
- **Root cause:** the output header lines go into the plant's log channel and
  must match operator/log conventions; the UI title is for humans picking a
  tool. When the 08-04 rename changed the output headers along with the title,
  the headers were rolled back the next day — but the UI title kept the new
  name.
- **Evidence:** `9db12dc` (08-03) "EQ Sample Lamella A/B" → "Lamella A/B";
  `6b47c87` (08-04) renamed the tool "EQ Sampling" → "Quality Sampling"
  including headers → "Quality Sample Tank A/B"; `a57840c` (08-05, HEAD)
  reverted headers to "EQ Sample Tank A/B" while `<h1>` and the menu card
  still say "Quality Sampling".
- **Status:** **settled.** Output header/format text is operator-facing
  contract text (full spec owned by `water-ops-run-and-operate`); never change
  it as a side effect of UI renames, and never "align" UI titles and output
  headers without owner sign-off.

### E3. Feature timeline — the app was born multi-tool and grew tool by tool

- **Symptom/decision:** growth pattern for context (not a failure): the repo
  starts (2026-07-06) with home + EQ Sampling + EQ Log + Parshall Flume in a
  647-line file — no Sludge Blanket. Then: fluoride added to eqB (`a6b9fd2`,
  07-20); EQ Log tank picker became a select and gained Post Anoxic A with its
  own field group — pH at HMI, pH with Myron L, gal totalizer (`79b454d`,
  07-22); `paA` added to sampling (`8186b73`, 07-22); Sludge Blanket screen
  added (`490bdbc`, 07-23, +124 lines); Dosing CO2 "in Auto" option with
  range output and the `l-tot-unit` unit-switching id (`7991386`, 07-23).
- **Root cause:** tools and fields are added as operators need them; the
  single-file architecture absorbs each addition as a new screen div + build/
  render/clear functions (recipe owned by `water-ops-new-tool-recipe`).
- **Evidence:** hashes above, all verified by diff on 2026-08-10.
- **Status:** **settled** context. Expect more tools over time (owner,
  2026-08-10).

### E4. Cache-bump discipline — nine cache names in one month

- **Symptom/decision:** the service worker (SW — the script that makes the
  app work offline by serving cached files) cache name went `waterops-v1` →
  `waterops-v9` across the repo's one-month history. Owner confirmed
  2026-08-10: "updating cache was a problem."
- **Root cause:** the SW is cache-first, so installed users keep getting the
  old cached app until a byte-different `sw.js` with a NEW cache name
  installs, activates (`skipWaiting`/`clients.claim`), and deletes old caches.
  Bumping the cache name is the app's only update mechanism. The history
  shows the discipline being LEARNED: early feature commits (`79b454d`,
  `490bdbc`) shipped without a bump and standalone sw.js-only bumps followed
  (`4291111`, `da0fef8`) — i.e. the deployed update didn't show up until the
  bump was pushed separately. From `7991386` (07-23) onward, every deploy
  commit pairs the index.html change with its cache bump. Note the merged
  PR #1 (`01fb047`) carried no bump — one more reason its change never
  mattered in the field before being reverted.
- **Evidence:** bump commits, each verified by diff: `1a9beca` v1 (initial),
  `4291111` v1→v2, `da0fef8` v2→v3, `7991386` v3→v4, `6074323` v4→v5,
  `b77839c` v5→v6, `9db12dc` v6→v7, `6b47c87` v7→v8, `a57840c` v8→v9 (HEAD).
- **Status:** **still-live risk.** Every deploy MUST bump the cache name in
  the same change-set (change-control rule). The repo does not reveal where
  (or whether) the app is currently live-hosted — verify the deployed
  origin's `sw.js` cache name before and after any deploy rather than
  assuming. Full SW mechanics: `water-ops-pwa-and-mobile-playbook`.

### E5. The upload workflow and the `index[23].html` name (this skill OWNS this explanation)

- **Symptom/decision:** the canonical local working file is literally named
  **`index[23].html`**. Nearly every commit is "Add files via upload".
- **Root cause:** the working loop is download-then-upload: the single-file
  app is edited elsewhere (e.g. an AI chat), downloaded to the machine —
  where the browser's duplicate-download numbering produced the `[23]`
  suffix — then uploaded through the GitHub web UI. When deployed or
  committed the file is just `index.html`; every other skill calls it
  `index.html`, and this entry is the one place the quirk is explained.
- **Evidence:** 14 of 18 commits are titled "Add files via upload";
  HEAD (`a57840c`) `index.html` is byte-identical to the local
  `index[23].html` (verified by byte-compare, 2026-08-10).
- **Status:** **still-live risk, but bounded.** The repo DOES track the app
  (no history gap as of 2026-08-10 — do not claim otherwise), but the
  workflow has costs: commit messages are uninformative (diffs are the only
  record of intent), there are no branches or reviewable PRs (except the one
  Copilot PR, E1), and nothing guarantees the next download gets committed.
  After any change session, confirm the repo HEAD still matches the working
  file before trusting either as canonical.

### E6. Telegram — added at birth, deprecated, removal sanctioned

- **Symptom/decision:** the app formats entries the way it does because it
  originally fed a Telegram bot that parsed them into the plant's logging
  channel. The Telegram share button was added the day the repo was born and
  still exists at HEAD. The bot is retired/retiring (owner, 2026-08-10).
- **Root cause:** the delivery channel is changing — future direction is
  direct submission to SharePoint or another database.
- **Evidence:** `a3998d0` (2026-07-06) added `tgBtn`, `.tg` CSS, and the
  `window.open('https://t.me/share/url?url=&text=' + encodeURIComponent(text))`
  handler (that commit also changed the Copy button's initial label
  "Copy entry" → "Copy" — the surviving label inconsistency is owned by
  `water-ops-debugging-playbook`). All still present at HEAD.
- **Status:** **sanctioned work.** Removing the Telegram code is WANTED — do
  not "preserve for compatibility." Scope and staging:
  `water-ops-platform-campaign`.

## Rules distilled from the chronicle

1. `ST` field lists are plant facts — never "fix" their asymmetry (E1).
2. Treat AI-suggested completions of apparent omissions with suspicion; the
   one merged AI PR in this repo's history was reverted (E1).
3. Output header/format text is contract text; UI renames must not touch it
   without sign-off (E2).
4. Every deploy bumps the SW cache name in the same change-set — the history
   shows what happens when it lags (E4).
5. Do not assert where the app is live-hosted; verify the deployed origin's
   `sw.js` cache name directly (E4).
6. The repo is canonical and current (HEAD = local file as of 2026-08-10);
   re-verify that identity after any change session (E5).
7. Telegram code is dead weight, not legacy to protect (E6).

## How to extend this chronicle

When history grows, add entries using the same read-only commands this skill
was built with. Never run mutating git commands against the project repo from
a skill session.

```bash
# Fresh read-only clone (do NOT reuse any session temp path)
git clone https://github.com/Daniel-ai-yi/water-ops /path/to/clone

# Full ledger with dates (messages are mostly "Add files via upload" — rely on diffs)
git -C /path/to/clone log --format='%h %ad %s' --date=iso

# What a commit changed (stat, then targeted diff)
git -C /path/to/clone show --stat <hash>
git -C /path/to/clone diff <hashA> <hashB> -- index.html
git -C /path/to/clone diff <hashA> <hashB> -- sw.js

# A file exactly as it was at a commit
git -C /path/to/clone show <hash>:index.html
git -C /path/to/clone show <hash>:sw.js
git -C /path/to/clone show <hash>:manifest.webmanifest

# Check repo HEAD still matches the local working file
git -C /path/to/clone show HEAD:index.html | cmp - "/path/to/index[23].html"
```

For each new entry: state the symptom or decision, dig for the root cause
(diff + owner context), cite the commit hash or exact file fact as evidence,
and assign a status (settled / still-live risk / sanctioned work). Date-stamp
anything volatile.

## Provenance and maintenance

All claims verified 2026-08-10 against a clone of
https://github.com/Daniel-ai-yi/water-ops and the local working file.
Re-verify with (clone fresh first, as above):

- Ledger shape: `git -C /path/to/clone log --oneline | wc -l` — expect 18 as of HEAD `a57840c` (2026-08-05); more means this chronicle needs new entries.
- E1 revert: `git -C /path/to/clone diff 01fb047 6074323 -- index.html` — shows `'ph'` removed from eqB.
- E1 current state: `grep -n "eqB" index.html` in the working file — field list has `turbidity`, no `ph`.
- E2 header split: `git -C /path/to/clone diff 6b47c87 a57840c -- index.html` — headers back to "EQ Sample Tank A/B"; `grep -n "Quality Sampling" index.html` — screen title unchanged.
- E4 cache name at HEAD: `git -C /path/to/clone show HEAD:sw.js | grep CACHE` — `waterops-v9` as of 2026-08-10; a higher number means deploys happened since.
- E5 repo currency: the byte-compare command in the section above — if it reports a difference, the repo or the local file has moved and "no drift" no longer holds.
- Deployed origin: unverified as of 2026-08-10 — never cite a live URL or its SW state without checking it at that moment.
