# Quality Gates

The full program record lives in
[planning/QUALITY-PROGRAM.md](../planning/QUALITY-PROGRAM.md); this page is
the working summary.

## The gates

| Gate | Where | Rule |
|---|---|---|
| Lint | `webos/eslint.config.js` | strictest ruleset, `--max-warnings 0` |
| Coverage | `webos/vitest.config.js` ratchet floors | statements/branches/functions/lines floors **only go up** — never lower one to pass; fix the change |
| Theme contrast | `webos/tests/contrast-audit.test.js` | token-pair contrast checks including wallpaper-extreme flattening; this is not a complete criterion-level WCAG 2.1 AAA conformance audit |
| Docs coverage | `webos/tests/docs-coverage.test.js` | ARCHITECTURE.md Module index must name every file under `src/` |
| Docs references | `webos/tests/docs-references.test.js` | every backticked repo path in root docs must exist |
| Registry version consistency | `webos/tests/registry-versions.test.js` | 13/13 catalog rows match the duplicated fetch/PWA version literals; this checks internal consistency, not freshness against the upstream release or package digest |
| E2E sweep | `packaging/e2e/webos-sweep.mjs` | 31 phases in the latest inspected CI run; FireCraft live-site probes may skip when the public host is unreachable, so they do not replace deterministic local-package integration tests |
| Shellcheck + Aegis | CI | shell scripts linted; scanner pinned to `aliasfoxkde/aegis` (baseline-gated) |
| CodeQL | `.github/workflows/codeql.yml` | pinned JavaScript/TypeScript security-extended scan on pull requests, main pushes, and weekly |

Application and repository gates run in `.github/workflows/ci.yml`; CodeQL runs separately. The active GitHub Safeguards ruleset requires both CodeQL results and the full WebOS CI check before merge. The latest inspected successful main run (2026-11-23, SHA `d08d2d1`) passed 95 Vitest files / 1,475 tests and a 31/31 sweep.

The package shortcut `npm run ci` currently runs lint, coverage and build only. Hosted CI additionally runs Aegis, ShellCheck, docs/contrast gates and the browser sweep. The tag release workflow currently reruns only part of that set. Until H12 is implemented, do not describe `npm run ci` alone as the complete release gate. GitHub
Code Quality is a separate billed feature and was not enabled as part of this
setup; the PR had no Code Quality analysis result.

## Principles

- **Ratchets only go up.** Lowering a floor or deleting a test to go green
  is a stop-the-line violation.
- **A measured metric must be reproducible.** Anything timing- or
  order-dependent is a broken gate.
- **Every fix ships with a regression test** naming the failure it prevents.


See the [FireCraft WASM Package Runtime Handoff](../planning/FIRECRAFT-WASM-PACKAGE-AND-RUNTIME-HANDOFF-2026-11-27.md) for current cross-repository integration/security findings, and the [2026-11-27 quality addendum](../planning/QUALITY-PROGRAM.md#2026-11-27-re-audit--ci-equivalence-react-warnings-and-firecraft-package-trust) for evidence and release-gate gaps.
