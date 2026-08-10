---
name: water-ops-build-and-env
description: Load when looking for Water Ops build/install/test commands, a package.json, npm scripts, CI config, bundler/transpiler setup, or when wondering how to compile, bundle, package, or deploy this project. Answer in short - there is no build system, on purpose. Read this before running npm install or searching for tooling that does not exist.
---

# Water Ops: build system and environment

**There is NO build system. This is deliberate.** No npm, no package.json, no
node_modules, no bundler, no transpiler, no test suite, no CI/CD, no linters,
no lockfiles. If you feel the urge to run `npm install` or hunt for a build
script, STOP — your assumptions about this repo are wrong. The app is one
hand-written vanilla HTML/CSS/JS file (`index.html`) plus static assets.

Why (owner directives, current as of 2026-08-10): single-file simplicity,
offline-friendly operation in the field, and dependency-free by security
policy — this is confidential NALCO/Ecolab plant data; security is a must and
no third-party code is allowed.

## What "environment" means here

1. **A browser.** Targets as of 2026-08-10: iOS Safari, Android Chrome, and
   desktop Chrome/Edge — all three, not mobile-only.
2. **Optionally a static server** for secure-context testing (service worker
   and clipboard need HTTPS or localhost):
   `python3 -m http.server 8000` — or any static file server.
3. **A static host** for deploy. The canonical repo is
   https://github.com/Daniel-ai-yi/water-ops (public); the live deployed
   origin is GitHub Pages at `https://daniel-ai-yi.github.io/water-ops/`
   (owner-confirmed and curl-verified serving `waterops-v9`, 2026-08-10).
   A move to Azure hosting is planned — see `water-ops-platform-campaign`.

Nothing else. No runtime, no server-side code, no database.

## The deployable artifact (exactly 6 files)

`index.html`, `sw.js`, `manifest.webmanifest`, `icon-192.png`,
`icon-512.png`, `apple-touch-icon-180.png`.

"Deploy" = copy these 6 files to the static host root. That is the entire
pipeline. Note: `sw.js` is currently absent from the local working folder —
do not forget it; its canonical content lives in
`water-ops-pwa-and-mobile-playbook`.

## Traps

- Assuming a bundler exists and "the source" is elsewhere — `index.html` IS the source.
- Adding a dependency "just for dev" — violates the no-third-party-code security rule.
- Introducing a build step that breaks deploy-by-copy.
- Adding npm test scaffolding nobody will run — validation is manual (`water-ops-validation-and-qa`).
- Committing tooling configs (.eslintrc, tsconfig, etc.) that imply tooling exists.

## Future (candidate only, not commitment)

IF the platform campaign adds a backend/auth (SharePoint submission, login),
tooling decisions get revisited then — with owner sign-off, not before.

## When NOT to use this skill

- How to serve/run/test locally → `water-ops-run-and-operate`
- Deploy/update mechanics, SW cache discipline → `water-ops-pwa-and-mobile-playbook`
- What changes are allowed and how → `water-ops-change-control`

## Provenance and maintenance

- Verified 2026-08-10 against the working folder and the canonical repo
  (github.com/Daniel-ai-yi/water-ops): the repo's HEAD tracks exactly the
  6 deployable files + .gitkeep (no package.json or configs anywhere), and
  its HEAD index.html is byte-identical to the local working file.
- Re-verify: `ls` the project root — no package.json should appear.
- If tooling is EVER introduced (with owner sign-off), rewrite this skill
  first; a stale "there is no build" claim is worse than none.
