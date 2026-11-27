# FireCraft WASM Package Runtime, Security & WebOS Delivery Handoff

**Prepared:** 2026-11-27  
**Repository:** aliasfoxkde/ArtCraft-WebOS  
**Inspected main SHA:** d08d2d11f6c546cf94f786f8f409b846a0549c60  
**Related FireCraft repository:** aliasfoxkde/FireCraft-CraftAnything-Experiment  
**Inspected FireCraft main SHA:** 284e00a53001da9bcb93d6de9e2c7f46abcdf2b3  
**Latest FireCraft release observed:** v0.3.2  
**Current WebOS FireCraft registry pin observed:** 0.2.3

This is a WebOS-only implementation handoff. FireCraft remains a separate editor project and repository. Do not copy FireCraft source code into this repository, and do not move the WebOS shell, App Store, package manager, or CI workflows into FireCraft.

---

## 1. Operating decision and repository boundary

ArtCraft WebOS is the application host and operating environment. It should be able to download a complete FireCraft browser/WASM package directly from a GitHub Release, validate it, install it locally, execute it in a restricted app context, mediate file/storage/host services, update it, and roll it back without requiring a permanent FireCraft website deployment for each installed instance.

### Responsibility split

| Capability | FireCraft repository owns | WebOS repository owns |
|---|---|---|
| Editor and document model | Yes | No |
| Standalone browser/WASM build | Builds and publishes | Downloads package |
| Host API client | Implements typed Rust/WASM client | — |
| Host API server/broker | — | Validates and enforces capabilities |
| Package manifest/artifact compatibility | Publishes and documents | Validates and consumes |
| App catalog and App Store UX | Supplies metadata | Owns |
| Download/install/cache/update/downgrade | Publishes immutable releases | Owns |
| App sandbox and host permissions | Declares requested permissions | Enforces |
| Per-app data storage and file picker | Requests typed scoped operations | Owns persistent storage and mediation |
| Desktop OS shell / windows / taskbar | No | Owns |

Do not make WebOS a FireCraft submodule, subdirectory, or build dependency. Coordination is through a versioned package manifest, host API contract, compatibility fixtures and independent pull requests.

### Keep the existing web model working

WebOS must continue to support external remote web apps, hosted third-party apps, external sites that cannot be embedded, and bundled first-party WebOS applications. Add a distinct **verified WASM package** application type. Do not turn every registry URL into a package install or break existing external app behavior.

---

## 2. Verified current state

### 2.1 WebOS baseline

The inspected main commit is d08d2d11f6c546cf94f786f8f409b846a0549c60.

The most recent main CI run inspected was GitHub Actions run 45096495247 (2026-11-23). Its CI job completed successfully. The decoded job log records:
- 95 Vitest test files passed;
- 1,475 Vitest tests passed;
- 31/31 end-to-end sweep phases green;
- zero page/console errors attributed by the sweep.

The WebOS ESLint configuration is strict and zero-warning. It includes ESLint recommended, React Hooks, jsx-a11y recommended, and a curated project ruleset. The V8 coverage configuration has four ratchets: 98.6% statements, 94.85% branches, 98.8% functions and 99.85% lines. The CI workflow also runs contrast and documentation gates, Aegis, ShellCheck, build and headless-Chromium sweep.

This is a strong baseline, but it is not proof that every release path runs the same checks. The exact coverage percentages reported in the 2026-11-02 progress snapshot are historical; the 2026-11-23 job proves that the then-current configured floors passed, not that those historical measured figures were re-measured.

### 2.2 FireCraft package available now

The latest FireCraft release observed is v0.3.2, published on 2026-11-27. It contains the complete browser package asset:

**firecraft-v0.3.2-wasm32-web.zip**

The release also publishes PACKAGE-MANIFEST.json, RELEASE-MANIFEST.json, SHA256SUMS.txt, and SOURCE-SHA.txt, plus native Windows, macOS and Linux packages.

WebOS currently pins FireCraft to 0.2.3 in three places:
1. webos/src/os/registry.js;
2. packaging/fetch-apps.sh APPS_LIST;
3. packaging/pwa-inject.sh FireCraft version table.

webos/tests/registry-versions.test.js asserts these internal copies agree. It does **not** prove the pinned version is the latest approved FireCraft release. The registry and release adoption workflow therefore have a stale-version gap.

### 2.3 Existing FireCraft host integration

The current WebOS host has special FireCraft integration in webos/src/shell/EmbedFrame.jsx:
- ready causes WebOS to restore an autosaved document using openDocument;
- applied triggers a debounced exportDocument and stores bytes at /Apps/firecraft/autosave.fw.png;
- exportPng is routed to the host save/export UX;
- postToApp() in webos/src/os/bridge.js correlates request/response messages with a request ID.

The generic WebOS file-storage bridge in webos/src/os/bridge.js already has a useful security foundation: app-scoped /Apps/<app-id>/ paths, path traversal rejection, a 64 MiB file-save bound, typed list/open/save/delete operations, request IDs and timeouts.

However, the inspected current FireCraft experiment main source does not contain the historic ready/applied/openDocument/exportDocument/exportPng bridge implementation. A current versioned FireCraft host client must be implemented in the FireCraft repository, with shared contract fixtures and an independent WebOS host implementation.

---

## 3. P1 security finding: iframe navigation and autosave disclosure

<badge color="danger">P1 — fix before local package execution</badge>

### 3.1 Source-level finding

In webos/src/shell/EmbedFrame.jsx, the event handler checks:
- event.source equals frame.current?.contentWindow;
- fd.source equals 'firecraft'.

It then accepts FireCraft control events such as ready without verifying event.origin. The generic webos:file-* bridge calls handleBridge(), which does validate the origin; the special FireCraft branch returns before that generic validation.

The ready event invokes firecraftRestore(), which reads the saved document from WebOS storage. In webos/src/os/bridge.js, postToApp() posts the request to the iframe using target origin *, and its reply listener checks source marker and request ID but not the reply origin.

### 3.2 Potential attack path

1. A legitimate FireCraft iframe loads.
2. The frame navigates to an attacker-controlled origin; the iframe WindowProxy identity can remain the same.
3. A page at that origin sends a forged message with source: 'firecraft' and type: 'ready'.
4. WebOS trusts the message and calls firecraftRestore().
5. WebOS reads the local autosaved document and sends it to the iframe using wildcard target origin.
6. The newly loaded page can potentially receive the document bytes.

This is a source/configuration finding, not a live exploit test. Do not describe it as dynamically exploited. The code path is sufficient to require a fix and a hostile-navigation regression test.

### 3.3 Required correction

For the existing remote hosted FireCraft path:
- Validate both the exact iframe WindowProxy and the expected origin on every inbound event.
- Derive the allowed origin from a trusted, validated registry entry; do not trust a URL supplied by an untrusted app message.
- Bind every host-to-app message to the configured exact target origin; do not use * for data-bearing requests.
- Validate event.origin as well as source marker and request ID on every response.
- If the iframe URL/origin changes, a navigation occurs, or the app is removed, invalidate pending requests and revoke all app capabilities.
- Make autosave/restore/export failures visible to the user through a status or notification path; do not silently pretend a save succeeded.

For the future locally cached, sandboxed package:
- Do not give the package same-origin access to the WebOS shell or other app storage.
- Prototype a sandboxed iframe with scripts enabled but without allow-same-origin, and mediate access through a dedicated MessageChannel.
- Use a one-time per-launch nonce for capability bootstrap, bind the MessagePort to the exact frame/window lifecycle, and close it on navigation, reload, uninstall or permission revocation.
- Do not use message source strings as authentication. They are protocol labels, not credentials.
- Do not use wildcard messaging for document bytes, file contents, user preferences or other private data. If an initial opaque-origin bootstrap must use a wildcard target, send only a one-time handshake/capability transfer, never the document or persistent secret; prove the handshake cannot be replayed after navigation.
- Test whether the chosen runtime can load the wasm-bindgen JS/WASM files, use WebGL2, receive keyboard/pointer events, download/export, and operate under the selected sandbox flags across supported browsers.

**Exit:** hostile cross-origin frame navigation cannot trigger restore/export or receive app data; wrong-origin, forged-ready, stale-port, reload, timeout and permission-revocation tests pass.

---

## 4. Target package architecture

\`\`\`
FireCraft GitHub Release
  firecraft-vX.Y.Z-wasm32-web.zip
  PACKAGE-MANIFEST.json / WEBOS-PACKAGE.json
  SHA256SUMS.txt / RELEASE-MANIFEST.json / SOURCE-SHA.txt
              |
              v
WebOS App Catalog
  app type = verified-wasm
  app id / repo / tag / channel / host API range
              |
              v
Package Resolver
  stable / nightly / beta
  validate allowlisted repo, tag, ABI, manifest, digest, attestation
              |
              v
Package Installer
  download directly from immutable GitHub Release asset
  inspect inventory / enforce size limits / reject path traversal
  store immutable version by package digest
              |
              v
WebOS Package Cache
  current + previous known-good
  user pin / channel / per-app feature state
              |
              v
Isolated App Runtime
  sandboxed frame + MessageChannel capability broker
  no same-origin shell access
              |
              v
WebOS Host APIs
  per-app storage / file picker / notifications / clipboard grants
  lifecycle / export-save dialog / window actions
              |
              v
Automatic health check / switch / rollback
\`\`\`

The package loader must not compile or rebuild FireCraft. It downloads a release artifact produced and validated by FireCraft CI.

### 4.1 App source types

| Source type | Meaning | Runtime |
|---|---|---|
| remote-url | Third-party or independently hosted web app | Existing embed/browser behavior |
| verified-wasm | Versioned package from an allowlisted GitHub Release | Local WebOS sandbox |
| bundled-webos | App implemented and released with WebOS | Existing WebOS component/runtime |

Keep legacy remote FireCraft serving as a temporary fallback only until the local package is proven and the stable catalog pointer has moved. Do not remove the old path before rollback and package validation are demonstrated.

### 4.2 Package manifest contract

A WebOS package manifest must include:

| Field | Purpose |
|---|---|
| schema_version | Manifest schema version |
| app_id | Stable WebOS application ID, e.g. firecraft |
| app_version | Semantic version of FireCraft |
| channel | stable / beta / nightly |
| repository / release_tag | Allowlisted upstream source and immutable release identity |
| asset_name | Complete browser ZIP asset name |
| asset_sha256 / asset_size | Integrity and size limit |
| source_sha / tree_sha | Exact source provenance |
| workflow_identity / attestation | Build provenance and trusted workflow |
| entrypoint | HTML entrypoint within the package |
| files[] | Every expected package file, path, MIME type, size and hash |
| wasm_file / glue_file | WASM binary and wasm-bindgen JS glue references |
| host_api_min / host_api_max | Supported host API range |
| min_webos_version | Minimum compatible WebOS shell |
| capabilities[] | Requested host API capabilities with default grants |
| document_schema_read / document_schema_write | Format compatibility window |
| feature_flags | Flag schema and disabled-by-default Experimental features |
| package_limit | Compressed/uncompressed size bounds |

The manifest may be a dedicated file inside the WebOS package ZIP or a separately signed release asset, but the installer must validate that its file inventory matches the actual archive. Do not treat an outer RELEASE-MANIFEST.json as sufficient proof of the internal browser bundle entrypoint and host ABI unless it explicitly contains those fields.

### 4.3 Channel manifests

Publish small signed or attested channel pointers:
- stable.json
- nightly.json
- later beta.json

A channel manifest points to an immutable tag, exact asset name, package version, digest, manifest, minimum WebOS version and source SHA. It must be monotonically versioned/generation-numbered and validated against the trusted repository allowlist. A channel pointer can move; a release asset must not.

WebOS resolves a channel pointer, then downloads the immutable GitHub Release asset. Do not install from a moving latest/download URL.

---

## 5. Runtime isolation and host capability broker

### 5.1 Isolation requirements

The package contains executable code. A hash proves which bytes were downloaded, not that those bytes are safe. WebOS must continue to treat the package as untrusted application code.

Required:
- execution in an isolated frame/sandbox, not as a script loaded into the WebOS document;
- no direct access to the WebOS JavaScript global, IndexedDB stores, DOM, localStorage or service-worker scope;
- CSP appropriate to local package execution, WASM and WebGL2;
- restrictive network policy (default no network; capability/allowlist required for optional network features);
- app-specific storage root and capability checks on every operation;
- per-app file-open/save operations mediated by a visible host picker or app sandbox path;
- lifecycle and request budgets, timeouts, cancellation and shutdown cleanup;
- no arbitrary host command execution, arbitrary path access or shell access;
- error boundary and crash isolation per app window.

Prototype the iframe sandbox before selecting the final package loading mechanism. A blob URL, cached service-worker path, OPFS URL, or same-origin virtual route has different origin/CSP behavior. Validate the chosen mechanism in Chromium, Firefox and WebKit before fixing the ADR. Do not add allow-same-origin merely to make localStorage or fetch work; expose only the required host services.

### 5.2 Capabilities

Use a manifest-declared, user-visible capability set. Example FireCraft starting profile:

| Capability | Default | Purpose |
|---|---|---|
| app.storage.read | Allow within app root | Read FireCraft app preferences/recovery files |
| app.storage.write | Allow within app root | Write FireCraft recovery/project files |
| files.open | Ask/host picker | User chooses a file to open/import |
| files.save | Ask / host save dialog | Save/export to a user-selected destination |
| clipboard.read | Ask / gesture-bound | Paste from system clipboard |
| clipboard.write | Ask / gesture-bound | Copy/export clipboard data |
| notifications.create | Deny until needed | User-visible operation/status notices |
| window.lifecycle | Allow minimum | Ready, suspend, resume, shutdown |
| network.fetch | Deny by default | Optional future network-dependent feature |

File list/open/save/delete from the generic app bridge must remain scoped to /Apps/firecraft/. User-selected files outside that directory must be returned as a one-time granted blob/handle; never widen the app storage root.

### 5.3 Message protocol

Adopt one versioned protocol; do not preserve a second unauthenticated file API beside the generic storage channel.

Required properties:
- handshake: package identity, runtime identity, session nonce, protocol version and capability negotiation;
- request: request ID, message type, payload schema, capability name, deadline/timeout;
- response: matching request ID, ok/error code, bounded payload;
- lifecycle: host.init, app.ready, app.suspend, app.resume, app.shutdown;
- document operations: open, save, export and recovery snapshot;
- file APIs: list app-private files, open app-private file, save app-private file, delete app-private file;
- host APIs: open file picker, save dialog, notification, clipboard request;
- cancellation and channel close;
- bounded message/file sizes and explicit error handling.

Autosave behavior:
1. FireCraft emits a versioned dirty/document-changed signal.
2. WebOS debounces and requests an export/recovery snapshot.
3. The app returns a typed result and schema version.
4. WebOS stores it to the app-scoped recovery slot and confirms durable write.
5. On launch, WebOS offers/replays a known-good recovery snapshot after protocol handshake.
6. Every failed save is visible and retryable. Never silently swallow autosave errors or report “saved” before the storage write resolves.

---

## 6. Download, install, update and rollback

### 6.1 Installation lifecycle

1. User selects FireCraft in the WebOS app catalog.
2. Resolve selected channel (stable by default).
3. Fetch and validate the channel manifest.
4. Require the repository and asset to match the FireCraft allowlist.
5. Download the complete WASM ZIP directly from the immutable GitHub Release.
6. Verify signature/attestation where available, then SHA-256 and archive inventory.
7. Reject traversal paths, duplicate/case-colliding entries, symlinks or unexpected file types, decompression bombs, file-count excess and size-limit violations.
8. Verify host API compatibility and package schema.
9. Store under an immutable content-addressed version/digest.
10. Launch in the sandbox and complete boot/health check.
11. Atomically switch the catalog active version only after successful boot.
12. Retain previous known-good package and state.
13. Notify user of installed version/channel/source SHA.

Use resumable/timeout-aware downloads. If API rate limit or GitHub outage occurs, preserve the existing install. Do not replace a working package with an incomplete download.

### 6.2 Version cache

The cache must preserve:
- active version;
- previous known-good version;
- user-pinned version (if pinned);
- selected channel and last valid channel manifest generation;
- package hash/provenance receipts;
- app data and recovery files (separate from executable package).

Never delete user data when uninstalling or downgrading an executable. App data deletion must be a separate explicit action.

Cache implementation should choose between CacheStorage, OPFS and IndexedDB based on package size, browser quotas, streaming support and lifecycle semantics. Keep the storage backend behind an interface so that the package manager can change it without rewriting the app catalog.

### 6.3 Rollback

Rollback is an executable operation, not just a version selector:
1. Mark failed package unavailable for automatic promotion.
2. Stop further channel pointer updates.
3. Stop/suspend faulty app process/frame.
4. Switch active version to previous known-good package.
5. Re-negotiate host API and capability policy.
6. Restore app data only if the app schema is compatible; never blindly roll back a newer document schema.
7. Run startup and host-service smoke tests.
8. Record exact failed package digest, source SHA, error class and rollback reason.
9. Notify opted-in users and keep the failed package/evidence quarantined for diagnosis.

A channel manifest update must not delete prior immutable versions. WebOS should support user-pinned versions and “Switch back to Stable” even when offline, provided the cached stable artifact remains available.

### 6.4 Nightly and Experimental

- Stable channel is the default.
- Nightly channel requires explicit user opt-in and clear warnings.
- Beta can be added later if a distinct user cohort exists.
- Experimental feature flags remain disabled by default in every channel.
- Joining nightly does not turn on all experimental flags.
- WebOS stores feature choices per app/version and presents a per-feature disable action.
- A remotely disabled feature must never overwrite the user's local saved preference without an explanation; distinguish user opt-in from a remote kill switch.

---

## 7. Current WebOS quality and CI audit

### 7.1 Preserve the strong existing gates

Keep zero-warning ESLint and React Hooks / jsx-a11y checks, the four-metric V8 coverage ratchet, contrast audit, docs coverage and reference tests, checksum-pinned Aegis baseline, ShellCheck, CodeQL and the full headless Chromium app/shell sweep.

Do not lower thresholds to get a new runtime feature green. Add new package/runtime tests and raise floors only from repeatable measurements.

### 7.2 Fix the release workflow quality bypass

Current WebOS PR/push CI runs Aegis, ShellCheck, docs/contrast gates, coverage, build and E2E. The current tag release workflow reruns only lint, coverage and build before GitHub release / Cloudflare Pages deployment.

Refactor:
- extract a reusable exact-SHA quality workflow;
- invoke it from PR CI and tag release jobs;
- make tag publication depend on the same required checks;
- add merge_group to any required merge-queue checks;
- split release build, GitHub publication and Pages deployment into separate least-privilege jobs;
- do not grant contents: write or Cloudflare deploy credentials to the test job;
- publish only artifacts produced by the verified source SHA; do not rebuild a different unverified artifact in the publish job.

Release should fail if the quality workflow did not execute, was skipped, or ran against a different SHA.

### 7.3 Make local/hosted verification consistent

Today npm run ci runs lint, coverage and build. Hosted GitHub CI also runs Aegis, ShellCheck, contrast/docs and E2E. Provide:
- npm run ci:quick — quick lint and targeted tests;
- npm run ci:full — every required local gate including Aegis, ShellCheck, contrast/docs and browser sweep where dependencies exist;
- a machine-readable skip summary for unavailable Chrome/browser tooling;
- release mode which fails closed if any required tool is missing.

A developer may choose a faster loop. The release controller may not claim the full gate passed when it did not run.

### 7.4 Treat test warnings as defects

The 2026-11-23 CI log includes:
- React validateDOMNesting warning: button cannot appear as a descendant of button;
- React Testing Library act(...) environment warnings;
- aborted external fetches in third-party embed tests;
- background async work attempting to start after an app/frame has been destroyed.

Actions:
- Fix nested interactive controls in Files tiles.
- Correct Testing Library environment setup or wrap async updates with act.
- Mock third-party fetch paths in required unit/component tests; add separate clearly labeled live probes.
- Abort/await pending background tasks and close message ports/listeners/timers when frames unmount.
- Add a test-level console capture policy that fails on unexpected React warnings, uncaught errors and console errors.

Do not blanket suppress console output. Any expected warning needs a narrow test-specific assertion and reason.

### 7.5 Add package-manager-specific test suites

Unit tests:
- catalog descriptor parsing and allowlist;
- SemVer/channel resolution;
- manifest schema validation;
- digest/provenance mismatch;
- archive traversal, duplicate paths, MIME/type and size limits;
- package cache key, quota/eviction and atomic pointer switch;
- offline stable resolution and pinning;
- rollback and data preservation;
- capability grant/revoke.

Browser integration:
- real local package boot in sandbox;
- no host global, IndexedDB, DOM or storage access from app frame;
- MessageChannel handshake and timeout/cancel/reload behavior;
- hostile navigation and forged ready messages;
- same-origin and opaque-origin handling;
- save/open/export and autosave/recovery;
- large file transfer and bounded payloads;
- service worker/cache eviction and offline rollback;
- update from current to next, then forced rollback.

E2E sweep:
- keep all existing 31 phases;
- make new FireCraft package phases use a deterministic local fixture built in the same CI job;
- test deployed FireCraft live site separately as an optional external health check;
- a skipped external probe must not be counted as a passed required local integration test.

---

## 8. WebOS implementation phases and exit gates

### H8 — Secure FireCraft iframe and protocol hardening

- [ ] **H8.1** Fix FireCraft special-message origin validation and host-to-frame wildcard messaging for the existing remote app path.
- [ ] **H8.2** Add hostile navigation, forged ready, wrong origin, wrong request ID, stale channel, timeout and data non-disclosure tests.
- [ ] **H8.3** Add cancellation/revocation on iframe load, navigation, reload, unmount, uninstall and permission changes.
- [ ] **H8.4** Define the versioned unified app host protocol: handshake, request IDs, capability names, storage/file/lifecycle/export, cancellation, limits and error codes.
- [ ] **H8.5** Keep /Apps/<app-id>/ storage scoping and path traversal protections; add capability grant/deny tests.
- [ ] **H8.6** Make autosave failures visible and retryable; add concurrent-window and frame-close tests.

**Exit:** no cross-origin frame can cause autosave restore or receive document bytes; all message trust decisions have tests and the protocol version is documented.

### H9 — Package format, catalog and release trust

- [ ] **H9.1** Define WebOS WASM package manifest and channel manifest schemas; publish fixture JSON schemas.
- [ ] **H9.2** Add allowlisted GitHub source/release/asset resolver and immutable release tag validation.
- [ ] **H9.3** Verify ZIP inventory, SHA-256, attestation/provenance, source SHA, archive path safety and WebOS host API compatibility.
- [ ] **H9.4** Add package-type registry descriptor while preserving existing remote-url and bundled-web app types.
- [ ] **H9.5** Eliminate duplicated FireCraft version tables. The registry-version test must validate generated/catalog data and latest approved version policy, not only compare three manually duplicated values.
- [ ] **H9.6** Update FireCraft from pinned 0.2.3 to approved v0.3.2 package only after package compatibility and host integration tests pass.
- [ ] **H9.7** Add package integrity receipts, SBOM/provenance references and cache manifest.

**Exit:** WebOS can resolve and verify a known-good FireCraft package without trusting a hosted mutable URL; the catalog accurately identifies the package source/version.

### H10 — Sandboxed local WASM runtime

- [ ] **H10.1** Prototype sandboxed iframe loading with scripts enabled but no same-origin access; compare blob/CacheStorage/OPFS-backed loading strategies.
- [ ] **H10.2** Test WASM, wasm-bindgen JS glue, static assets, WebGL2, keyboard/pointer, clipboard mediation, downloads and CSP in Chromium, Firefox and WebKit.
- [ ] **H10.3** Implement one-time MessageChannel capability handshake and per-launch session identity; close/revoke on lifecycle changes.
- [ ] **H10.4** Implement host capability broker with user-visible prompts and least-privilege defaults.
- [ ] **H10.5** Support per-app data, open/save/export, autosave/recovery, notifications and shutdown/resume.
- [ ] **H10.6** Ensure app failure cannot crash the WebOS shell or access other app/user data.

**Exit:** local FireCraft package runs inside the WebOS sandbox with typed host APIs and no same-origin shell access; standalone FireCraft and remote apps still work.

### H11 — Package install, update, cache and rollback

- [ ] **H11.1** Implement stable/nightly channel resolver with signed/attested pointers to immutable release tags.
- [ ] **H11.2** Download directly from GitHub Releases, verify digest/provenance and install into content-addressed immutable storage.
- [ ] **H11.3** Keep current and previous known-good versions; atomically switch after startup health check.
- [ ] **H11.4** Add pin/downgrade, return-to-stable, offline rollback and per-app update history.
- [ ] **H11.5** Keep user documents, recovery files and settings independent from package replacement.
- [ ] **H11.6** Add explicit nightly opt-in and per-feature Experimental defaults off.
- [ ] **H11.7** Add update notifications, release notes, version/source provenance and rollback reporting.
- [ ] **H11.8** Test corrupt package, wrong digest, interrupted download, GitHub outage, API rate limit, wrong host ABI, failed startup, quota exhaustion and offline recovery.

**Exit:** WebOS can upgrade FireCraft to a verified new artifact and roll back to last-known-good offline without losing user data.

### H12 — CI, release and operational hardening

- [ ] **H12.1** Extract shared reusable quality workflow; make release rerun the same exact-SHA checks as PR CI.
- [ ] **H12.2** Add merge_group handling and required check identity; verify merge-queue candidate behavior.
- [ ] **H12.3** Split test/build/release/Pages deployment permissions into least-privilege jobs.
- [ ] **H12.4** Add console-warning failure policy and eliminate nested-button/act/task-cleanup warnings.
- [ ] **H12.5** Replace required external FireCraft sweep dependence with a local package fixture; retain optional live-site probes separately.
- [ ] **H12.6** Add package runtime security and lifecycle scenarios to the browser sweep; maintain existing shell/app regression phases.
- [ ] **H12.7** Add incident/rollback metrics, release receipts, package adoption and failure telemetry with explicit privacy constraints.
- [ ] **H12.8** Update release guide, user app-channel guide, package API docs, security model, WebOS app author guide and history.

**Exit:** a tag cannot publish unless the same required tests, scans, build and E2E evidence used for PR integration pass on the release source SHA.

---

## 9. Cross-repository acceptance tests

These scenarios require both repositories but must be implemented and validated independently.

1. **Manifest agreement:** FireCraft release manifest and WebOS package resolver agree on app ID, version, tag, asset, digest, host API range and capabilities.
2. **Install:** WebOS downloads FireCraft ZIP from GitHub Release directly, verifies it and launches the app without needing firecraft-web.pages.dev.
3. **Standalone:** the same FireCraft artifact starts normally in a standard browser without WebOS.
4. **Host handshake:** WebOS and FireCraft negotiate protocol version, capabilities and session; incompatible versions fail with a clear message.
5. **File open/save:** user chooses a local file through host picker; FireCraft edits and saves/export through WebOS; app cannot read outside its granted root.
6. **Autosave/restore:** changes are durably stored, then restored only to the correct FireCraft instance and schema; errors are surfaced.
7. **Origin attack:** navigate the iframe to an attacker page, send forged ready and request IDs, and verify no saved document bytes are disclosed.
8. **Isolation:** FireCraft cannot read WebOS shell globals, other app folders, shell IndexedDB records or localStorage.
9. **Upgrade:** WebOS moves from FireCraft v0.3.2 to a later immutable release after verification, while keeping the previous version.
10. **Rollback:** deliberately fail startup/health check; WebOS returns to last-known-good, restores the app pointer and preserves user files.
11. **Offline:** with network disconnected, current app launches and previous cached version can be restored.
12. **Nightly:** explicit channel enrollment installs nightly; Experimental flags remain off until individually enabled.
13. **Release provenance:** every running build has verifiable source SHA, artifact digest and release identity.

---

## 10. First-run checklist for WebOS implementation agent

- [ ] Read root AGENTS.md, WebOS docs/README.md, docs/planning/HANDOFF.md, PHASES.md, PROGRESS.md, QUALITY-PROGRAM.md, architecture docs, and current release workflow.
- [ ] Inspect git status, current branch, latest release/tag and CI status. Preserve all current work.
- [ ] Record a fresh baseline with npm ci, lint, coverage, build, Aegis, ShellCheck, contrast/docs gates and browser sweep on one exact SHA.
- [ ] Fix the P1 origin/navigation/autosave issue before enabling local package execution.
- [ ] Add regression tests before changing the protocol.
- [ ] Write the ADR and protocol schema before implementing the runtime.
- [ ] Prototype sandbox and MessageChannel behavior across supported browsers.
- [ ] Implement package resolver/cache only after the sandbox/protocol has a passing local fixture.
- [ ] Update catalog to v0.3.2 only after package, host API and compatibility verification; do not only change version strings.
- [ ] Refactor release workflow so the full CI gates cannot be bypassed by a tag push.
- [ ] Add a failure-injected upgrade/downgrade/rollback E2E.
- [ ] Update this handoff, WebOS task index, progress snapshot, quality program, user/developer docs and release receipts.
- [ ] Open a PR to WebOS main from this dedicated branch after the implementation and evidence are complete. Do not make any FireCraft changes in this WebOS PR.

## 11. References and repository files

### WebOS source / workflow
- webos/src/shell/EmbedFrame.jsx — FireCraft autosave, restore and iframe message dispatch.
- webos/src/os/bridge.js — generic app storage bridge and host-to-app request correlation.
- webos/src/os/registry.js — external and craft app catalog, including FireCraft version/source.
- packaging/fetch-apps.sh — upstream release ZIP fetcher and current hardcoded app version list.
- packaging/pwa-inject.sh — PWA injection and version list.
- webos/tests/registry-versions.test.js — internal version consistency test.
- webos/tests/os-bridge.test.js and webos/tests/embed-frame-autosave.test.jsx — existing bridge/autosave tests.
- packaging/e2e/webos-sweep.mjs — 31-phase browser sweep; currently includes live FireCraft deployment-dependent steps.
- .github/workflows/ci.yml, .github/workflows/release.yml, webos/package.json, webos/vitest.config.js, webos/eslint.config.js.

### FireCraft integration references
- FireCraft v0.3.2 GitHub Release: https://github.com/aliasfoxkde/FireCraft-CraftAnything-Experiment/releases/tag/v0.3.2
- FireCraft host integration roadmap: https://github.com/aliasfoxkde/FireCraft-CraftAnything-Experiment/blob/docs/quality-and-webos-roadmap-2026-11-27/docs/QUALITY-AND-WEBOS-ROADMAP-2026-11-27.md
- FireCraft proposed Rust/WASM protocol contract: https://github.com/aliasfoxkde/FireCraft-CraftAnything-Experiment/blob/docs/quality-and-webos-roadmap-2026-11-27/docs/WEBOS-HOST-INTEGRATION.md
- Current FireCraft WASM package asset: firecraft-v0.3.2-wasm32-web.zip.

**Completion rule:** do not call this a “local FireCraft app” merely because the ZIP was cached. It is complete only when the package is verified, sandboxed, capability-limited, host-compatible, updateable, downgradeable, and tested against hostile navigation and offline failure.
