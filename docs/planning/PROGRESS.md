# Progress — ArtCraft WebOS

Live status board. Update this (and [SESSIONS.md](SESSIONS.md)) at the end
of every working session. Plan-of-record for open work:
[HANDOFF.md](HANDOFF.md).

## Snapshot (runtime baseline 2026-11-02; repository and CI verified 2026-11-02)

| Dimension | State |
|---|---|
| Released / deployed version | **3.25.0** ([v3.25.0](https://github.com/aliasfoxkde/ArtCraft-WebOS/releases/tag/v3.25.0)) |
| Production | https://artcraft-webos.pages.dev — served bundle hash matches local `webos/dist`, `sw.js` cache `webos-v3.25.0` |
| Craft apps | all thirteen on their current upstream web release (2026-11-02): cadcraft 0.13.0, deckcraft 0.11.1, designcraft 0.13.0, effectcraft 0.17.0, firecraft 0.2.3, gridcraft 0.12.1, lightcraft 0.14.0, pdfcraft 0.14.0, photocraft 0.16.0, **solidcraft 0.30.1** (tangency-labs), soundcraft 0.13.1, vectorcraft 0.17.0, wordcraft 0.11.0 — eight of them served from the R2 `craftweb` route |
| FireCraft | https://firecraft-web.pages.dev — **v0.2.3**, served from the upstream release zip (wasm `firecraft-web-d13f1a83e8a1a76f`) on the standard `fetch-apps.sh` pipeline; bridge autosave/open-with/exportPng→Photos + export-to-Files (2026-10-21) |
| Tests | 1475 specs / 95 files, `npm run ci` green |
| Coverage | 98.88% stmts / 94.86% branches / 98.95% funcs / **99.91% lines**; floors 98.6/94.6/98.8/99.85 in `webos/vitest.config.js` (95 files, measured − tolerance) |
| Lint | strictest ruleset, zero warnings |
| Accessibility | WCAG 2.1 AAA token gates incl. wallpaper-extreme flattening + desktop scrim pills |
| Docs coverage | ARCHITECTURE.md Module index ⇄ src/ enforced by `tests/docs-coverage.test.js` — now in GitForge CI pipeline |
| Registry versions | 13/13 craft-app rows match the deployed bundle (verified by asset + wasm hash); pinned by `tests/registry-versions.test.js`. The shell SW cache version is pinned to `src/version.js` by `tests/version.test.js` (added after the v3.25.0 SW-bump miss) |
| CI | GitHub Actions: lint, docs gate, WCAG audit, coverage ratchet, build, E2E sweep, **Shellcheck**, **Aegis**, and **CodeQL**. GitHub Safeguards now requires both CodeQL results and the full WebOS CI check before merge. GitForge mirrors the application gates. Historical GitForge trigger issue remains in R2. |
| Aegis | scanner pinned to `aliasfoxkde/aegis` **v0.6.4** (checksum-verified) via `packaging/aegis-scan.sh`; baseline regenerated with that binary (1809 findings / 111 files / 22 categories); the gate fails only on findings outside the baseline |

## Done (evidence in HISTORY.md / SESSIONS.md)

- [x] Quality program phases A–G (audit → plan → lint → AAA → coverage →
      cleanup/triage → docs gate → release) — complete at v3.14.0.
- [x] FireCraft deployed + integrated as twelfth craft app — complete at
      v3.15.0.
- [x] E2E tooling committed to `packaging/e2e/`.
- [x] Docs-reference gate (`tests/docs-references.test.js`) + full-docs
      staleness pass — 18+ stale references fixed, superseded plans
      (PROGRAM-3, QUALITY-PLAN) bannered (2026-10-15).
- [x] OSS readiness merged in PR #14 (2026-11-01): public repository,
      prototype scope and product vision documented, MIT license for
      first-party shell/tooling, security-reporting guidance, clone/setup
      instructions, and pinned CodeQL workflow. GitHub CI and CodeQL passed;
      Pages was not redeployed because no runtime code changed.

## Open (ordered; atomic tasks in HANDOFF.md)

- [ ] **P15/P16** OSS-readiness refinements in
      [OSS-READINESS-PLAN.md](OSS-READINESS-PLAN.md) — dependency-review
      workflow, branch-coverage ratchet trial, P16.3 accessibility audit
      beyond contrast.
- [ ] **H1** Routine-change loop — standing procedure, apply per change.
- [ ] **H8–H12** FireCraft local WASM package runtime, protocol security, package install/update/rollback and release-gate hardening. See [FireCraft WASM Package Runtime Handoff](FIRECRAFT-WASM-PACKAGE-AND-RUNTIME-HANDOFF-2026-11-27.md); WebOS work stays in this repository, FireCraft editor work stays in its repository.
- [x] **H2** FireCraft: H2.0–H2.6 complete — v0.2.1 deployed, v3.17.0 shipped.
      H2.1 contract documented; H2.2 postToApp correlate implemented; H2.3
      autosave (applied→exportDocument→filePut, ready→fileGet→openDocument);
      H2.4 open-with via Files context menu; H2.5 exportPng→Photos and
      export-to-Files (shared SaveDialog component, 2026-10-19); H2.6 E2E probe
      open→apply→exportDocument round-trip verified.
- [ ] **H3** Upstream adoption watch — maintain app ownership/asset pins and validate new upstream releases. The existing FireCraft registry/fetch/injection pin is 0.2.3, while FireCraft-CraftAnything-Experiment has published v0.3.2 (2026-11-27). Update only through the H8 package-integrity and host-compatibility path; the current registry-versions test proves internal consistency, not upstream freshness. The icon/mark question remains separate from package versioning.
- [x] **P9** Future-work items — P9.1 and P9.5 closed with the craft-app
      upgrade (2026-10-22); P9.2 preview server pinned to 127.0.0.1;
      P9.3 the sweep asserts internal state (window registry + IndexedDB);
      P9.4 the legacy driver deprecated, its surfaces ported into the sweep.
- [x] **P10** Adopt the upstream craft-app releases — complete 2026-10-22:
      all twelve craft apps upgraded to their current upstream web release
      (cadcraft 0.3.0→0.10.0 … vectorcraft 0.7.0→0.14.0, deck/sound/effect
      joined the standard release-zip pipeline), seven oversized wasm
      bundles moved to the R2 route via `packaging/r2-publish.sh`, registry
      rows + SW cache stamps re-pinned, each deployment production-verified
      (asset hash + `application/wasm`), shell released as v3.21.0.
- [x] **H4** Quality forward path — H4.1 coverage floors met (96.5/88.5/95.8/99.6), H4.2 deps refreshed (vitest 5.0.5, coverage-v8, fake-indexeddb; re-refreshed 2026-10-21: sql.js 1.15.1, vite 6.4.5 — React 19 / Vite 8 / ESLint 10 / happy-dom 21 deferred with reasons in SESSIONS.md), H4.3 Aegis re-scan zero new findings (baseline suppresses all), H4.4 sweep synced.
- [x] **H5** H5.3 container revalidation complete in node:24-trixie-slim; H5.1 pipeline registered (push, 4e1079cd…ccc957); H5.2 GitHub Actions as primary gate (runner dispatch pending for GitForge).
- [x] **H7** Coverage determinism + CI repair (2026-10-20) — three one-second
      clock intervals (Taskbar, Sidebar clock widget, Screensaver big clock)
      were covered only when the test file outlived the timer, so the same
      commit measured 95.87% functions on CI and 96.02% locally. The ratchet
      was set from the local number and turned CI red on every push.
      `tests/clock-timers.test.jsx` drives all three timers with fake timers;
      the metric is now reproducible (two consecutive full runs agree exactly)
      and the floors sit a fixed tolerance below the measured value.
      Also in this pass: the dead `makeFirecraftExportHandlers` export was
      removed from `src/shell/Window.jsx` (it duplicated the live
      state-driven handlers and existed only for its own test), the live
      export-to-Files path got component-level tests, and the E2E sweep's
      FireCraft open-with phase was repaired (it imported a module URL the
      production build does not serve) — the sweep is green 27/27.
- [x] **H6** Releases — latest observed WebOS release is v3.25.0 (published 2026-11-03); prior release history and automation details remain in HANDOFF.md / HISTORY.md. The latest inspected main CI receipt is run 45096495247 on main SHA d08d2d11f6c546cf94f786f8f409b846a0549c60 (2026-11-23). This is not a receipt for any future package-runtime changes.

## Deliberately closed (do not reopen without new evidence)

- The 4 uncovered lines — 3 are unreachable reducer/switch `default:` arms
  (calcState.jsx:126, state.jsx:483, MarkdownView.jsx:81) and Panels.jsx:336
  is a V8 attribution quirk; documented in QUALITY-PROGRAM.md. The paths
  that were genuinely reachable (Files.jsx:304 "Open in FireCraft",
  SaveDialog.jsx:50 backdrop cancel, Window.jsx:81-87 export-to-Files) are
  covered as of 2026-10-20. Ratchets only ever go up.
- Istanbul conditional-JSX attribution — covering those lines requires
  invoking the arrow functions (click the buttons), not refactoring JSX.
- Cloudflare compute on the rendering path — verified absent; keep it that
  way (static Pages + thin proxy worker only).

## Risk register

| ID | Description | Impact | Mitigation | Status |
|---|---|---|---|---|
| R1 | H3.1 blocked on FireCraft upstream shipping an official app icon (aliasfoxkde/FireCraft#2) | WebOS uses a code-drawn placeholder for FireCraft; official mark unavailable until upstream ships one | Swap script ready; upstream issue filed with exact requirements | ACCEPTED — placeholder glyph is the permanent design (2026-10-19) |
| R2 | Coverage ratchet was set from a non-reproducible local measurement (2026-10-19) | Every push from 594e579 to 7a6e12c had a red CI; v3.18.0 shipped with a failing gate and the release notes/progress board wrongly claimed CI was green | Root cause found (timer-dependent coverage); `tests/clock-timers.test.jsx` pins the metric; floors now sit a fixed tolerance below a measured value two consecutive runs agree on | RESOLVED 2026-10-20 |
| R3 | The Aegis CI step added 2026-10-20 never scanned anything: nonexistent `setup-python` SHA, wrong tool (the PyPI `aegis-cli` is an unrelated project), nonexistent `--format github`, and a baseline built by a different aegis version (8 categories vs 22) whose fingerprints matched nothing | The security gate was reported as wired but was in fact dead — the repo was unscanned in CI while the progress board claimed a scan ran | `packaging/aegis-scan.sh` pins the real scanner (aliasfoxkde/aegis v0.6.4) with a verified checksum; the baseline was regenerated with that binary; a negative control (injected AWS key) proves the gate fails on new findings | RESOLVED 2026-10-21 |


## Audit addendum — 2026-11-27

Repository source inspected at main SHA d08d2d11f6c546cf94f786f8f409b846a0549c60. Latest inspected successful main CI run: 45096495247, dated 2026-11-23. It reported 95 test files / 1,475 Vitest tests passing and a 31/31 browser sweep. The configured four-metric coverage floors passed. This is historical CI evidence, not a fresh run on any future runtime changes.

### Current cross-project findings

- WebOS FireCraft registry/fetch/PWA injection pins are all 0.2.3. FireCraft's separate repository published v0.3.2 on 2026-11-27. The current registry-versions test verifies that the three WebOS literals agree; it does not check upstream release freshness or a package digest.
- FireCraft-specific iframe message handling validates event.source but does not validate event.origin before accepting ready/applied/export control messages. The generic bridge does validate origin. The host-to-app postToApp path uses wildcard target origin. A frame navigation could potentially spoof ready, trigger autosave restore, and receive local document bytes. Fix and add a hostile-navigation regression test before local package execution.
- The current release workflow reruns lint, coverage and build but does not rerun all PR CI gates (Aegis, ShellCheck, docs/contrast and browser sweep). A tag release must use the same reusable exact-SHA quality workflow.
- webos-sweep includes FireCraft live-deployment checks that can skip when the external deployment is unreachable. Keep those probes separate from a required deterministic local-package integration fixture.
- CI logs showed nested-button validateDOMNesting and act-environment warnings. Files tiles contain outer button controls with inner action buttons. Fix the invalid interactive nesting and make unexpected warnings fail tests.
- npm run ci currently runs lint, coverage and build, not every hosted CI gate. Provide explicit quick/full modes and fail closed for release validation.

The implementation status of these findings is tracked in H8–H12 of HANDOFF.md and the linked FireCraft WASM Package Runtime Handoff. Do not mark the package migration complete merely because the registry version was changed.
