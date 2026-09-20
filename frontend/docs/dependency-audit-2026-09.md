# Frontend Dependency Audit and Upgrade Plan (September 2026)

- **Tree audited**: upstream `kubeflow/pipelines` master `791a19c8` (2026-09-17), fast-forwarded into this fork's `master` on 2026-09-19.
- **Audit date**: 2026-09-19. Registry facts below were read from npm on that date; the verification passes re-read them on 2026-09-20.
- **Scope**: `frontend/` (React client), `frontend/server/` (Node/Express BFF), `frontend/mock-backend/`, `test/frontend-integration-test/`, and the CI/toolchain that builds them.
- **Companion**: `react-18-19-upgrade-checklist.md` in this directory is upstream's record of the first-generation React migration that this audit builds on.

---

## 0. Executive summary

1. **The React upgrade this fork set out to do has already been completed upstream.** When this fork's `master` was last synced (`ecfe94eb`, 2025-07-14) the frontend was React 16.12 on Create React App, Jest + Enzyme, Material-UI v3, react-router-dom v4, react-query v3, react-flow-renderer v9, react-vis, TypeScript 3.8, Storybook 6 and Node 22. Between 2026-02-05 and 2026-09-15 upstream migrated every one of those: React 19.2, Vite 8 + Vitest 4, Testing Library 16 (Enzyme removed), MUI v5 + Emotion, react-router v8, TanStack Query v5, @xyflow/react 12, recharts 3, TypeScript 6, ESLint 10 flat config, Storybook 10, Node 24. The fork was 1,062 commits behind (299 touching `frontend/`).
2. **`master` and this branch now carry that code.** `origin/master` was fast-forwarded (`ecfe94eb..791a19c8`) and pushed; the fork had no frontend commits of its own, so nothing was lost and the fork has zero divergence from upstream.
3. **The synced tree is healthy in this environment** (Node 24.14.0): typecheck, lint (0 warnings), Prettier, the React-19 peer gate, 139 UI test files / 2,181 tests, 27 server test files / 1,025 tests, the production Vite build and the Storybook build all pass. Details in §2.
4. **What remains to reach "latest supported"** is a second-generation upgrade, much smaller than the first. Of 106 direct dependencies in the three main manifests, 34 are current, 25 need a minor or patch, 19 are a major behind, 23 should be deleted and 5 replaced (§4). The items that need real work, in priority order:
   - **Node 24.14.0 → 24.21.0 and npm 11.19.1** (P1, small): the pin has skipped three Node security releases and blocks jsdom 30 and npm 12 (§5.9). Node 24 stays the target until at least Q2 2027.
   - **MUI 5.18 → 9.4** (P1, the largest item; v5 is unsupported upstream). One hop is feasible; the verified breakage is one removed icon alias (`ExecutionNode.tsx:21`), ~20 slot-prop sites, two theme override keys that silently stop applying, two test assertions and 28 Emotion-hash snapshots. Upstream tracks this in issue #14142 with a stepwise v6 → v7 → v9 plan that must be reconciled first (§5.1, §5.13).
   - **Server** (P1): five majors (`@kubernetes/client-node` 2, `tar-stream` 3, `supertest` 7, `google-auth-library` 11, `form-data` removal), the deprecated `crypto-js`, and one finding that is not a version at all: two `overrides` pin Express 5 to the Express-4 router, so 13 wildcard routes and one `req.params[0]` read have never run under real Express 5 syntax (§5.8). Clears all 7 open `npm audit` findings.
   - **Test toolchain** (P1): Vitest 4 → 5 and jsdom 24 → 30 (blocked by the Node pin), plus two defects that exist today: a dead `environmentMatchGlobs` key and Storybook imports through a redundant `tsconfig` hack (§5.3).
   - **Tailwind 3.0 → 4.3** (P2) via `@tailwindcss/vite`, which also deletes the prebuild step whose absence fails tests, at the cost of a browser-floor decision (§5.2).
   - **Replacements** (P2): unmaintained `dagre` → `@dagrejs/dagre`, `react-virtualized` → `react-window`; a latent ts-proto 1.x/2.x drift in the checked-in protobuf code (§5.6, §5.7, §5.12).
   - **Hygiene** (S): 13 dead packages including a placeholder `fs`, stale `@types/*`, `@types/node` two majors behind the runtime (§5.7).
   - **Process and floor decisions** (P1, small): no browser-support policy exists and the build still targets es2015, which Tailwind 4, MUI 9 and the bundle work all wait on (§5.14); the Dependabot `mui` group has never matched anything, so peer-coupled bumps arrive as un-installable single PRs (§5.16); of the 7 open `npm audit` findings only the high has an in-range fix, the two `minio` transitives need an accepted-risk record (§5.17).
   - **Not yet**: TypeScript 7 (typescript-eslint's peer range blocks it; add a Dependabot ignore), npm 12 (needs the Node bump), Node 26 (LTS from 2026-10-28), Storybook 11 (alpha), React Compiler (6 suppressions to fix first) (§6, deferred).
5. **Why your earlier attempts broke on tests, plots and the router**, and what is different now: the fork's 2025 stack had three hard React blockers (Enzyme, react-vis, react-router-dom 4) that had to move together with React. Upstream removed each first and bumped React last. The remaining risk is churn, not blockers: Emotion-hash and SVG snapshots (one ROC-curve snapshot is 4,325 lines), class-instance test coupling, and one-shot mocks under StrictMode (§5.10).
6. **Your earlier attempt branches** (`react-18-upgrade`, `react-v7-upgrade`, `cursor/*`) are superseded in every respect and should be closed (§3). The fork should stay a mirror and land all of this upstream first (§5.13, §7).

**How this audit was produced.** Eleven research agents (npm registry, official changelogs and migration guides, the code) produced the cluster findings in §5; each was then checked by two adversarial verifiers with separate lenses (registry facts, codebase) that re-derived every version, date, peer range, file path and count; refuted claims were corrected before they reached this document, and the most consequential catches are named inline (a hard build break in MUI 9, a visual regression in Tailwind 4, a handler that breaks under real Express 5, a lockfile check that does not guard the Node pin). A completeness pass then covered packages and cross-cutting concerns no cluster owned (§5.11 to §5.17). Everything in §2 was measured by running the tools on this tree.

---

## 1. Where the fork was, where upstream is

| Area | Fork `master` at `ecfe94eb` (2025-07-14) | Upstream `master` at `791a19c8` (2026-09-17) | Landed by |
|---|---|---|---|
| React | 16.12 (`ReactDOM.render`) | 19.2.8 (`createRoot`, StrictMode in dev/test) | #13070 (18), #13153 (19), #13229 (StrictMode) |
| Build | Create React App 5 + craco | Vite 8.0 (rolldown) | #12754 (2026-02-09) |
| Unit tests | Jest (react-scripts) + Enzyme + react-test-renderer + snapshot-diff | Vitest 4.1 + Testing Library 16 + jest-dom 7 + user-event 14 | #12754, #12856, #13019, #13075 |
| UI kit | @material-ui/core 3.9 | @mui/material 5.18 + Emotion 11 | #12925 (2026-03-02) |
| Router | react-router-dom 4.3 | react-router 8.4 (declarative `HashRouter`) | #14417 (2026-09-15) |
| Data fetching | react-query 3 | @tanstack/react-query 5.102 | #12946, #13089 |
| DAG canvas | react-flow-renderer 9 | @xyflow/react 12.11 | #12945 |
| Charts | react-vis 1.11, react-svg-line-chart, d3 5 | recharts 3.7 (d3 removed) | #12829 (2026-02-18), #13626 |
| Code editor | brace + react-ace 7 | ace-builds 1.44 + react-ace 15 | dependency sweeps |
| Styling | typestyle 2 + Tailwind 3 | typestyle 2.4 + Tailwind 3.0.11 (unchanged approach) | — |
| TypeScript | 3.8 | 6.0.3 | #14089 (2026-08-20) |
| Lint / format | tslint (server), eslint-config-react-app, Prettier 1.19 | ESLint 10 flat config + typescript-eslint 8 + react-hooks 7, Prettier 3.8 | #12997, #13833, #12943 |
| Storybook | 6.3 (webpack, CRA preset) | 10.6 (`@storybook/react-vite`) | #12940 |
| API clients | swagger-codegen (Java jar) | OpenAPI Generator (Docker) | #12817 |
| Protobuf | google-protobuf + grpc-web + protobufjs 6 | protobufjs 8 + ts-proto 2 (grpc-web/MLMD removed) | #13986 |
| Node / npm | 22.14 | 24.14.0 pinned (`engine-strict`), npm 11.17.0 via `packageManager` | #13037, #13553 |
| Server | Express 4, @kubernetes/client-node 0.12, Jest 25 + ts-jest, tslint | Express 5.2 (but pinned to the Express-4 router via overrides, see §5.8), @kubernetes/client-node 1.4, Vitest 4, ESLint, ESM | #12045 (Sep 2025), #12756 (2026-02-05), #12997 |
| CI guards | none frontend-specific | In CI: `check-react-peers` (React 19 gate), `check-lockfile-drift`, mock-backend typecheck, generated-client drift check. Manual scripts only (not wired into any workflow): coverage baseline/compare, visual compare (Playwright + pixelmatch), ui-smoke-test | #12881, #13026, #13019, #12754 |

Dependency delta in `frontend/package.json` between the two commits: **45 packages removed, 30 added, 39 version-changed** (full lists in Appendix A).

Upstream's own record of the React track is `frontend/docs/react-18-19-upgrade-checklist.md` (15 numbered steps, all checked off; last updated 2026-04-07). Its ordering is worth keeping as a template for the work below: prerequisite cleanup → CI peer gate → ecosystem dependencies first → core bump → stabilization → StrictMode → docs.

---

## 2. Baseline health of the synced tree (measured 2026-09-19)

Environment: Node 24.14.0 (from `frontend/.nvmrc`), npm 11.9.0 (bundled; CI installs the pinned 11.17.0), `npm ci` clean, `postinstall` installed `server/` and `mock-backend/`.

| Check (npm script) | Result | Notes |
|---|---|---|
| `typecheck` (`tsc --noEmit`) | pass | 16 s |
| `typecheck:mock-backend` | pass | |
| `lint` (`lint:ui` + `lint:server`, `--max-warnings=0`) | pass | ESLint JSON output confirms 0 messages; the `react-hooks/*` React Compiler rules are enabled at `warn` and currently produce none |
| `format:check` (Prettier 3.8) | pass | |
| `check:react-peers` (target React 19) | PASS | 791 lockfile entries checked; allowlist empty for 17/18/19 |
| `test:ui` (Vitest 4.1.11, jsdom, StrictMode on) | 139 files / 2,181 tests pass | 137 files / 2,094 tests in the main run; 2 files (87 tests) initially failed only because I invoked `vitest` directly and skipped the `pretest` hook that generates `src/build/tailwind.output.css`; they pass once it exists. Residual noise: 2 `act()` warnings from `TensorboardViewer` in `ViewerContainer.test.tsx` |
| `server` tests (Vitest) | 27 files / 1,025 pass, 2 todo | 45 s |
| `vite build` | pass | 14.5 s. Output: one JS chunk 2,978.84 kB (gzip 832.36 kB) + CSS 28.18 kB. No code splitting (see §6, "Nice to have"). Upstream's March 2026 baseline was 4,499 kB / 1,004 kB gzip |
| `storybook build` | pass | 7 stories, 7.2 MB static output |
| `npm audit` (frontend) | 0 vulnerabilities | |
| `npm audit` (frontend/server) | 7 (1 high, 6 moderate) | high: `js-yaml` 4.x pulled in by `@kubernetes/client-node@1.4.0`; moderate: `uuid` via `gaxios`/`google-auth-library@9`, `stream-json` via `minio@8`. All are cleared by the server bumps in §6 |

Interpretation: there is nothing broken to fix before starting the next round. Tests, plots and the router, the three areas that broke in your earlier attempts, are covered by the passing suite (ROC curve, confusion matrix, paged table, HTML/Markdown viewers and Tensorboard all have tests; `Router.test.tsx` and the run/recurring-run router tests exercise the v8 declarative router).

---

## 3. Your earlier upgrade branches (superseded)

All six branches were cut from mid-2025 `master` and predate upstream's migration. None can be merged as-is (they modify the CRA/Jest/Enzyme stack that no longer exists), and every goal they pursued is now on `master` at a newer version.

| Branch | Base | Commits | What it attempted | Status vs. `master` today |
|---|---|---|---|---|
| `react-18-upgrade` | 2025-05-28 | 3 | React 17, MUI 4, recharts 2 + victory, Storybook 7, TS 4.9 (still Enzyme/CRA) | superseded: React 19.2, MUI 5, recharts 3, Storybook 10, TS 6 |
| `react-v7-upgrade` | 2025-07-20 | 2 (260 files) | React 18, MUI 5 + `@mui/styles`, react-router-dom 6, reactflow 11, recharts 2 + victory, RTL 13 (still CRA/craco) | superseded: react-router 8, @xyflow/react 12, Vite |
| `cursor/migrate-frontend-tests-to-react-testing-library-1984` | 2025-06-26 | 20 (170 files) | Enzyme → RTL on React 17 | superseded: upstream removed Enzyme in #12754 and finished the RTL/act cleanup in Apr–Jun 2026 |
| `cursor/fix-lint-errors-in-test-files-a2b1` | 2025-06-26 | 12 (136 files) | MUI v5 codemod, TS 4.9, lint fixes | superseded |
| `cursor/fix-typescript-implicit-any-errors-52f1` / `-d73c` | 2025-06-25 | 2 each | TS 4.9 strictness fixes, MUI 4 | superseded |

Recommendation: close/delete these branches after this audit is accepted, so nobody rebases them onto the new `master`. The `snyk-*` branches are dependency bumps against the old lockfile and are equally obsolete.

---

## 4. Dependency inventory vs. latest (npm registry, 2026-09-19)

Verdicts come from §5 (research, then verified). "Gap" compares the locked version with the `latest` dist-tag. `keep` means current or intentionally held.

### 4.1 `frontend/package.json`

| Package | Section | Locked | Latest | Gap | Verdict | Note |
|---|---|---|---|---|---|---|
| `@emotion/react` | dep | 11.14.0 | 11.14.0 | current | keep |  |
| `@emotion/styled` | dep | 11.14.1 | 11.14.1 | current | keep |  |
| `@eslint/js` | dev | 10.0.1 | 10.0.1 | current | keep |  |
| `@mui/icons-material` | dep | 5.18.0 | 9.4.0 | major (+4) | bump-major | 9.4.0 single hop |
| `@mui/material` | dep | 5.18.0 | 9.4.0 | major (+4) | bump-major | 9.4.0 single hop |
| `@storybook/addon-links` | dev | 10.6.0 | 10.6.0 | current | keep |  |
| `@storybook/react-vite` | dev | 10.6.0 | 10.6.0 | current | keep |  |
| `@tanstack/react-query` | dep | 5.102.8 | 5.103.1 | minor | bump-minor |  |
| `@testing-library/dom` | dev | 10.4.1 | 10.4.2 | patch | bump-minor |  |
| `@testing-library/jest-dom` | dev | 7.0.1 | 7.0.1 | current | keep |  |
| `@testing-library/react` | dev | 16.3.3 | 16.3.3 | current | keep |  |
| `@testing-library/user-event` | dev | 14.6.7 | 14.6.7 | current | keep |  |
| `@types/d3-dsv` | dev | 3.0.7 | 3.0.7 | current | keep |  |
| `@types/dagre` | dev | 0.7.54 | 0.7.54 | current | remove |  |
| `@types/express` | dev | 5.0.6 | 5.0.6 | current | keep |  |
| `@types/js-yaml` | dev | 3.12.3 | 4.0.9 | major (+1) | remove |  |
| `@types/lodash` | dev | 4.14.119 | 4.17.25 | minor | bump-minor |  |
| `@types/lodash.groupby` | dep | 4.6.9 | 4.6.9 | current | remove |  |
| `@types/node` | dev | 20.19.28 | 26.6.2 | major (+6) | bump-major | ^24 (not 26) |
| `@types/pako` | dep | 3.0.0 | 3.0.0 | current | remove | npm-deprecated |
| `@types/react` | dev | 19.2.14 | 19.3.0 | minor | bump-minor |  |
| `@types/react-dom` | dev | 19.2.4 | 19.3.0 | minor | bump-minor |  |
| `@types/react-virtualized` | dev | 9.22.3 | 9.22.3 | current | remove |  |
| `@typescript-eslint/eslint-plugin` | dev | 8.65.0 | 8.70.0 | minor | bump-minor |  |
| `@typescript-eslint/parser` | dev | 8.65.0 | 8.70.0 | minor | bump-minor |  |
| `@vitejs/plugin-react` | dev | 5.2.0 | 6.1.1 | major (+1) | bump-major |  |
| `@vitest/coverage-v8` | dev | 4.1.11 | 5.0.1 | major (+1) | bump-major |  |
| `@xyflow/react` | dep | 12.11.6 | 12.11.6 | current | keep |  |
| `ace-builds` | dep | 1.44.0 | 1.44.0 | current | keep |  |
| `autoprefixer` | dev | 10.4.19 | 10.6.1 | minor | remove | (with Tailwind 4) |
| `browserslist` | dev | 4.28.8 | 4.29.0 | minor | remove | (keep package.json key until Tailwind 4) |
| `d3-dsv` | dep | 3.0.1 | 3.0.1 | current | keep |  |
| `dagre` | dep | 0.8.5 | 0.8.5 | current | replace | @dagrejs/dagre ^3.1.1 + @dagrejs/graphlib ^4.0.5 |
| `eslint` | dev | 10.8.0 | 10.11.0 | minor | bump-minor |  |
| `eslint-plugin-import-x` | dev | 4.17.1 | 4.17.1 | current | keep |  |
| `eslint-plugin-react-hooks` | dev | 7.1.1 | 7.1.1 | current | keep |  |
| `fs` | dev | 0.0.1-security | 0.0.1-security | current | remove |  |
| `globals` | dev | 16.5.0 | 17.12.0 | major (+1) | bump-major |  |
| `http-proxy-middleware` | dep | 4.2.0 | 4.2.0 | current | remove | (client copy only) |
| `immer` | dep | 11.1.18 | 11.1.18 | current | keep |  |
| `js-yaml` | dep | 5.4.1 | 5.4.2 | patch | bump-minor |  |
| `jsdom` | dev | 24.1.3 | 30.1.0 | major (+6) | bump-major | 30.1.0 after Node >=24.15 |
| `lodash` | dep | 4.18.1 | 4.18.1 | current | keep | security floor >=4.18.1 |
| `lodash.debounce` | dep | 4.0.8 | 4.0.8 | current | remove |  |
| `lodash.flatten` | dep | 4.4.0 | 4.4.0 | current | remove |  |
| `lodash.groupby` | dep | 4.6.0 | 4.6.0 | current | remove |  |
| `lodash.isfunction` | dep | 3.0.9 | 3.0.9 | current | remove |  |
| `markdown-to-jsx` | dep | 9.10.2 | 9.10.3 | patch | bump-minor |  |
| `pako` | dep | 3.0.1 | 3.0.2 | patch | bump-minor |  |
| `pixelmatch` | dev | 5.3.0 | 7.2.0 | major (+2) | bump-major |  |
| `playwright` | dev | 1.58.0 | 1.63.0 | minor | bump-minor |  |
| `pngjs` | dev | 7.0.0 | 7.0.0 | current | keep |  |
| `postcss` | dev | 8.5.23 | 8.5.28 | patch | remove | (with Tailwind 4) |
| `prettier` | dev | 3.8.1 | 3.9.8 | minor | bump-minor |  |
| `proto3-json-serializer` | dep | 4.0.2 | 4.0.2 | current | remove |  |
| `protobufjs` | dep | 8.8.0 | 8.8.0 | current | keep |  |
| `re-resizable` | dep | 6.11.2 | 6.11.2 | current | keep |  |
| `react` | dep | 19.2.8 | 19.3.0 | minor | bump-minor |  |
| `react-ace` | dep | 15.0.0 | 15.0.0 | current | keep |  |
| `react-dom` | dep | 19.2.8 | 19.3.0 | minor | bump-minor |  |
| `react-dropzone` | dep | 20.1.1 | 20.1.2 | patch | bump-minor |  |
| `react-router` | dep | 8.4.0 | 8.4.0 | current | keep |  |
| `react-textarea-autosize` | dep | 8.5.9 | 8.5.9 | current | remove |  |
| `react-virtualized` | dep | 9.22.6 | 9.22.6 | current | replace | react-window ^2.3.1 |
| `recharts` | dep | 3.7.0 | 3.10.1 | minor | bump-minor |  |
| `rollup-plugin-visualizer` | dev | 5.14.0 | 7.1.1 | major (+2) | bump-major |  |
| `runtypes` | dep | 7.0.5 | 7.0.5 | current | keep |  |
| `semver` | dev | 7.7.4 | 7.8.5 | minor | bump-minor |  |
| `storybook` | dev | 10.6.0 | 10.6.0 | current | keep |  |
| `tailwindcss` | dev | 3.0.11 | 4.3.3 | major (+1) | bump-major | 4.3.3 via @tailwindcss/vite |
| `ts-proto` | dep | 2.12.1 | 2.12.4 | patch | bump-minor | move to devDependencies; regenerate src/generated |
| `tsx` | dev | 4.23.12 | 4.23.13 | patch | bump-minor |  |
| `typescript` | dev | 6.0.3 | 7.0.2 | major (+1) | keep | blocked by typescript-eslint peer <6.1 |
| `typestyle` | dep | 2.4.0 | 2.4.0 | current | keep |  |
| `url` | dep | 0.11.4 | 0.11.4 | current | remove |  |
| `vite` | dev | 8.0.16 | 8.3.0 | minor | bump-minor |  |
| `vitest` | dev | 4.1.11 | 5.0.1 | major (+1) | bump-major |  |
| `yaml` | dev | 2.8.3 | 2.9.1 | minor | bump-minor |  |

### 4.2 `frontend/server/package.json`

| Package | Section | Locked | Latest | Gap | Verdict | Note |
|---|---|---|---|---|---|---|
| `@aws-sdk/credential-providers` | dep | 3.980.0 | 3.1136.0 | minor | bump-minor |  |
| `@kubernetes/client-node` | dep | 1.4.0 | 2.0.0 | major (+1) | bump-major |  |
| `@types/crypto-js` | dev | 3.1.47 | 4.2.2 | major (+1) | remove |  |
| `@types/express` | dev | 5.0.6 | 5.0.6 | current | keep |  |
| `@types/gunzip-maybe` | dev | 1.4.3 | 1.4.3 | current | keep |  |
| `@types/node` | dev | 22.19.7 | 26.6.2 | major (+4) | bump-major | ^24 (not 26) |
| `@types/supertest` | dev | 2.0.16 | 7.2.1 | major (+5) | bump-major |  |
| `@types/tar` | dev | 4.0.5 | 7.0.87 | major (+3) | remove | npm-deprecated |
| `@types/tar-stream` | dev | 1.6.7 | 3.1.4 | major (+2) | remove | (tar-stream 3.2.1 bundles types) |
| `@vitest/coverage-v8` | dev | 4.1.11 | 5.0.1 | major (+1) | bump-major |  |
| `axios` | dep | 1.18.0 | 1.20.0 | minor | remove |  |
| `crypto-js` | dep | 4.2.0 | 4.2.0 | current | replace | node:crypto; npm-deprecated |
| `express` | dep | 5.2.1 | 5.2.1 | current | keep |  |
| `form-data` | dep | 2.5.6 | 4.0.6 | major (+2) | remove | (after supertest 7) |
| `google-auth-library` | dep | 9.15.1 | 11.1.0 | major (+2) | bump-major |  |
| `gunzip-maybe` | dep | 1.4.2 | 1.4.2 | current | replace | node:zlib createGunzip()/createInflate() selected by the existing detectCompression() peek in minio-helper.ts |
| `http-proxy-middleware` | dep | 4.2.0 | 4.2.0 | current | keep |  |
| `js-yaml` | dep | 5.4.1 | 5.4.2 | patch | bump-minor |  |
| `lodash` | dep | 4.18.1 | 4.18.1 | current | remove |  |
| `minio` | dep | 8.0.7 | 8.0.7 | current | keep |  |
| `peek-stream` | dep | 1.1.3 | 1.1.3 | current | replace | small local Transform on node:stream (or the existing detectCompression pattern generalized) |
| `prettier` | dev | 3.8.1 | 3.9.8 | minor | bump-minor |  |
| `supertest` | dev | 4.0.2 | 7.2.2 | major (+3) | bump-major |  |
| `tar-stream` | dep | 2.2.0 | 3.2.1 | major (+1) | bump-major |  |
| `typescript` | dev | 6.0.3 | 7.0.2 | major (+1) | keep | optional TS 7 pilot |
| `vitest` | dev | 4.1.11 | 5.0.1 | major (+1) | bump-major |  |

### 4.3 `frontend/mock-backend/package.json`

| Package | Section | Locked | Latest | Gap | Verdict | Note |
|---|---|---|---|---|---|---|
| `@types/express` | dep | 5.0.6 | 5.0.6 | current | keep |  |
| `express` | dep | 5.2.1 | 5.2.1 | current | keep |  |

### 4.4 `test/frontend-integration-test/package.json` and `frontend/scripts/ui-smoke-test/package.json`

| Package | Locked | Latest | Verdict | Note |
|---|---|---|---|---|
| `webdriverio`, `@wdio/cli`, `@wdio/config`, `@wdio/local-runner`, `@wdio/mocha-framework`, `@wdio/utils` | 9.31.6 | 9.31.9 | bump-minor | no v10 exists |
| `@wdio/junit-reporter`, `@wdio/spec-reporter` | 9.31.2 | 9.31.2 | keep | |
| `mocha` | 10.8.2 | 12.0.2 | keep | `@wdio/mocha-framework` 9.x depends on `^10.8.2` (11 was tried and reverted upstream) |
| `@puppeteer/browsers` (+ `$@puppeteer/browsers` override) | 3.2.2 exact | 3.2.2 | keep | forces a patched major into `@wdio/utils` (`^2.2.0`, two unpatched `extract-zip` advisories); lifted only by WebdriverIO v10 |
| `serialize-javascript` (override under mocha) | 7.0.5 | 7.1.1 | bump-minor | |
| `wait-port` | 1.1.0 | 1.1.0 | keep | |
| `selenium/standalone-chromium` (Docker) | 152.0 | 152.0 | keep | consider the fully qualified tag |
| `playwright` (ui-smoke-test) | 1.58.1 | 1.63.0 | bump-minor | keep in step with `frontend` |
| `sharp`, `looks-same` (ui-smoke-test) | 0.35.4 / 10.0.1 | current | keep | `engines` says `>=18`; real floor is 20.9 |

**Totals for the three primary manifests (106 direct dependencies)**: keep 34, bump-minor 25, remove 23, bump-major 19, replace 5.

## 5. Cluster findings

Each cluster below was researched by one agent (npm registry, official changelogs and migration guides, and the code itself), then checked by two adversarial verifiers, one re-deriving every registry fact and one re-deriving every codebase claim with its own grep. Where the verifiers refuted something, the corrected fact is what appears here; the most consequential catches are called out inline. Effort is S/M/L/XL in engineer-days (S ≤ 1, M 2 to 5, L 5 to 10); priority P0 to P3.

| § | Cluster | Locked → latest | Effort | Risk | Priority | Verification |
|---|---|---|---|---|---|---|
| 5.1 | MUI | 5.18.0 → 9.4.0 | M | medium | P1 | done, 2 passes; 1 hard break found |
| 5.2 | Tailwind | 3.0.11 → 4.3.3 | M | medium | P2 | done, 2 passes; 1 visual regression found |
| 5.3 | Test/build toolchain | Vitest 4.1 → 5.0, jsdom 24 → 30 | M | medium | P1 | done, 2 passes |
| 5.4 | TypeScript/ESLint | TS 6.0.3 (7.0.2 blocked) | M | low | P2 | done, 2 passes |
| 5.5 | React core, router | 19.2.8 → 19.3.0; router current | S | low | P2 | done, 2 passes |
| 5.6 | Graph, plots | recharts 3.7 → 3.10; dagre → @dagrejs | M | medium | P2 | done, 2 passes |
| 5.7 | Remaining client libs | 1 replace, 13 delete, 7 bump | M | medium | P2 | done, 2 passes |
| 5.8 | Server, mock backend | 5 majors + Express un-fork | M | medium | P1 | done, 2 passes; `req.params[0]` found |
| 5.9 | Runtime, CI, e2e tooling | Node 24.14 → 24.21, npm 11.19 | S | low | P1 | done, 2 passes |
| 5.10 | Tests / plots / router health | | | | | done, 1 pass |
| 5.11 | TanStack Query (completeness pass) | 5.102.8 → 5.103.1 | S | low | P3 | done, 1 pass |
| 5.12 | Generated-code toolchains (completeness pass) | OpenAPI Generator 7.19 → 7.25; ts-proto output 1.x → 2.x | M | medium | P2 | done, 2 passes |
| 5.13 | Upstream coordination (completeness pass) | | M | medium | P1 | done, 1 pass |
| 5.14 | Browser-support floor and build targets (completeness pass) | es2015 → Vite baseline | S | low | P1 | done, 1 pass; 3 numbers corrected |
| 5.15 | Bundle size, code splitting, sourcemaps (completeness pass) | | S | low | P2 | measured directly; research agent blocked by a model safeguard |
| 5.16 | Dependabot grouping and ignore rules (completeness pass) | | S | low | P1 | done, 1 pass; mechanism confirmed from source |
| 5.17 | Security advisories in the lockfiles (completeness pass) | | S | low | P1 | done, 1 pass |

### 5.1 MUI: `@mui/material` + `@mui/icons-material` 5.18.0 → 9.4.0

| | |
|---|---|
| Latest | 9.4.0 (2026-08-27). Dist-tags: `latest-v7` 7.3.11 (LTS, security + regressions only), `latest-v6` 6.5.0, `latest-v5` 5.18.0. **There is no v8**; MUI skipped it to realign with MUI X. v5 and v6 are listed as no longer supported. |
| React / TS support | React ^17 \|\| ^18 \|\| ^19; TypeScript ≥ 4.9 documented (repo is on 6.0.3 and typechecks MUI 5 today with `skipLibCheck`; TS 6 support for 9.4's `slotProps` generics is unstated and must be proven by `npm run typecheck` on the branch). Emotion 11.14.x stays the engine; Pigment CSS is an optional peer, not required. Browser floor Chrome 117 / Firefox 121 / Safari 17 / Edge 121. |
| Footprint | 77 files import `@mui/*`; 63 barrel import statements from `@mui/material` (Button 29, Tooltip 23, CircularProgress 12, Dialog 10, MenuItem 9, and so on); 8 subpath imports (`TextField`, `styles`, `Snackbar`, `Select`, `Switch`, `SvgIcon`, `Button`, `colors`), all still present in v9's explicit exports map; 64 per-icon default imports across 29 files plus one barrel icon import (`SideNav.tsx:35`). |
| Effort / risk / priority | M / medium / **P1** (largest remaining item; v5 is unsupported) |
| Verification | done (54 registry claims: all versions, dates, dist-tags, peers, exports and guide facts confirmed; codebase pass corrected the usage inventory and found one hard break the research had missed, all folded in below). |

**One hop is feasible.** The codebase has none of the v6-only or v7-only breakage (no `Hidden`, no `Grid`/`Grid2`, no deep imports beyond one segment, no `@mui/lab`, no system props, no `theme.palette.mode`, no `@mui/styles`, no `component=` on button-like components, no `MuiTouchRipple`/`ListItemIcon`/`sizeMedium`). Go 5.18 → 9.4 directly, applying the v6, v7 and v9 checklists in one PR, with 7.3.11 (LTS) as the fallback pin if v9 exposes a blocking regression.

**Hard break (missed by the research, caught by verification)**

- `components/graph/ExecutionNode.tsx:21` imports `@mui/icons-material/RemoveCircleOutline`, one of the 23 legacy `*Outline` aliases v9 removed. The file does not exist in 9.4.0, so `npm install @mui/icons-material@9` fails typecheck, Vite and every test that imports `ExecutionNode` until it is renamed to `RemoveCircleOutlined`. It renders the unknown-execution-state fallback (line 110), which has no test; add one.

**What v9 removes that this code uses**

1. `TextField` `InputProps` / `inputProps` / `InputLabelProps` → `slotProps.input` / `slotProps.htmlInput` / `slotProps.inputLabel`. 20 sites in 8 files: 13 `InputProps` (10 on the local `<Input>` atom, 3 directly on `<TextField>` in `CustomTable.tsx:464`, `NewRunParameters.tsx:150`, `NewRunParametersV2.tsx:361`), 2 `inputProps` (`Trigger.tsx:310` on `<Input>`; `NewRunV2.tsx:889` nested inside `InputProps` with a `data-testid`, must become `slotProps.htmlInput` or the test id is lost), 5 `InputLabelProps` all on `<Input>` (`CustomTable.tsx:320`, `Trigger.tsx:207,218,248,259`). The official `deprecations/text-field-props` codemod reaches the 3 direct `<TextField>` sites; the 17 `<Input>`-atom sites (`src/atoms/Input.tsx` spreads props into `TextField`) are manual. **Do not touch** `UploadPipelineDialog.tsx:209` and `NewPipelineVersion.tsx:418`: their `inputProps={{ tabIndex: -1 }}` belongs to the local `DropzoneArea` atom, not MUI. `TextField` `inputRef` (7 sites in `NewExperiment.tsx`, `NewPipelineVersion.tsx`) is retained.
2. `Dialog` `PaperProps` → `slotProps.paper`: `PipelinesDialog.tsx:127`, `PipelinesDialogV2.tsx:137`, `NewRun.tsx:332,402`, `NewRunV2.tsx:911,1022`. Codemod `deprecations/dialog-props` covers these. The 10 `Dialog classes={{ paper }}` sites are unaffected.
3. `Checkbox`/`Switch` `inputProps` → `slotProps.input`: 3 sites. `viewers/RuntimeArtifactComparison.tsx:463` (Checkbox, `aria-hidden`, asserted by its test), `pages/FrontendFeatures.tsx:82` (Switch, `aria-label`), and `NewRunParametersV2.tsx:137` (Checkbox, `aria-label 'Set custom pipeline root.'`, which `NewRunParametersV2.test.tsx:853` queries by label; the label must survive). `Select` `inputProps` (6 sites, including the multiple-select at `RuntimeArtifactComparison.tsx:451`) is not deprecated and stays.
4. Button CSS classes `textPrimary`/`textSecondary`/`textInherit` are gone. `src/Css.tsx` overrides `MuiButton.styleOverrides.textPrimary/textSecondary`; in v9 those keys are silently ignored and every primary text button loses its border and margin. Move them to `styleOverrides.root.variants` keyed on `{ variant: 'text', color: 'primary' }`. **No test renders inside `ThemeProvider`** (only `src/index.tsx` does), so this regression is invisible to the entire Vitest suite, snapshots included; `scripts/visual-compare.mjs` is the only safety net for every theme override. `SideNav.test.tsx:103-106` asserts the old class names and must change.
5. `TableSortLabel` `iconDirectionDesc` class is gone; `CustomTable.test.tsx:209` asserts it (there is no `classes=` override on `TableSortLabel`; `CustomTable.tsx:463` is the rows-per-page `TextField`).
6. Emotion class hashes change on any MUI major: 28 of 44 snapshot files contain `css-*` hashes and will churn. Regenerate only after the visual diff (`npm run visual:baseline` before, `visual:current` + `visual:diff` after).
7. Behaviour changes to re-verify, no code removal: `<TextField select>` renders its label as a `<div>` (`CustomTable.tsx:459`, `Trigger.tsx:161,323` via `<Input select>`; `Trigger.test.tsx:40` queries the combobox by that label name and must be re-run); `MenuItem` throws outside a `Menu`/`Select` and `Tab` outside `Tabs` (all users are nested); `Tabs` roving tabindex (`RuntimeArtifactComparison.tsx:242-254`); `TablePagination` uses `Intl.NumberFormat` (values < 1000 here); ButtonBase Enter/Space now bubbles (`NativeArtifactLineage.tsx:150,404`); Backdrop no longer sets `aria-hidden` (`PlotCard.test.tsx:59` clicks `[class*="MuiBackdrop-root"]`, still rendered). v9 also replaced `process.env.NODE_ENV === 'test'` checks with feature detection / user-agent sniffing for jsdom and happy-dom, which the guide warns "might lead to unintended CI changes"; triage under the Vitest + jsdom + StrictMode setup. The v6 ripple change makes `fireEvent.click` on Button/Checkbox/Switch/Tabs log `act()` warnings unless awaited (43 test files use `fireEvent.click`).
8. The `optimizeDeps.exclude: ['@mui/material/colors']` line in `vite.config.mts` was a workaround for v5's non-exports-map dual package (PR #13628, blank screen on `npm start`). v9 exports `./colors` explicitly. Simplest fix: stop importing the subpath in `components/graph/SubDagLayer.tsx:21`, delete the exclusion, and smoke-test `npm start`.
9. `tsconfig.json` excludes `**/?*test.*`, so `npm run typecheck` never sees test files; v9 typing fallout inside tests surfaces only at runtime.

**Steps**

1. Capture baselines on the current tree: `npm run visual:baseline`, `npm run coverage:baseline`, one green `npm run test:ui`.
2. Bump `@mui/material` and `@mui/icons-material` to `^9.4.0` together (Dependabot already groups `@mui/*`). Keep Emotion as is. Do not add `@mui/material-pigment-css`. No `react-is` override is needed on React 19.
3. Rename the `RemoveCircleOutline` import (hard break) and `SideNav.tsx:35` to a per-icon import.
4. Run `npx @mui/codemod@latest deprecations/all src` (text-field, dialog, checkbox, switch, button-classes, table-sort-label-classes, tooltip, snackbar, select-classes, input-props). The v6/v7 codemods (`v6.0.0/all`, `v7.0.0/all`, `v9.0.0/system-props`) are expected no-ops here; run them for completeness. Format afterwards.
5. Hand-migrate the 17 `<Input>` atom sites (item 1), the theme variants (item 4), the two test assertions (items 4 and 5), the `colors` subpath (item 8).
6. `npm run typecheck` and `npm run lint:ui`; expect `slotProps` typing fallout in the ~45 files that mix typestyle and MUI.
7. Run the suite **without** `-u`; triage failures into removed-API leftovers, v9 DOM/a11y changes (item 7), and ripple `act()` noise (wrap in `await act` or switch to `userEvent`, which 14 files already use).
8. Regenerate the 28 snapshot files, then `npm run visual:current && npm run visual:diff` on SideNav, dialogs, CustomTable sort headers, Trigger, NewRun forms, tooltips; this is the only check that covers the theme override migration.
9. Update `frontend/README.md` lines 7 and 8 ("Vite 7" is already stale, "MUI v5" → "MUI v9").

**Decisions for you**

- Single hop to 9.4 (recommended, code evidence supports it) vs. staging through 7.3.11 to harvest runtime deprecation warnings first (7.x still ships the removed icon alias, so the hard break only appears at v9 either way).
- Whether to give the `<Input>` atom an explicit `slotProps` API so future MUI codemods can see it, or just migrate its 17 callers in place.
- Snapshot policy: 28 files are effectively Emotion-hash snapshots; this is the moment to replace them with role/text assertions so they stop churning on every MUI minor.
- Whether to render tests inside `ThemeProvider` (a `renderWithTheme` helper) so theme overrides are at least exercised by the suite.

### 5.2 Tailwind CSS 3.0.11 → 4.3.3 (or remove it)

| | |
|---|---|
| Latest | 4.3.3 (2026-07-16); a 4.3.4/4.4.0 is imminent (unreleased changelog is non-empty). `v3-lts` dist-tag = 3.4.19 (2025-12-10, a single fix release). 3.0.11 is from 2022-01-05. |
| How it is wired today | Out-of-band prebuild: `npx tailwindcss build -i src/tailwind.css -o src/build/tailwind.output.css`, triggered by five `pre*` hooks (`prestart`, `prebuild`, `pretest`, `pretest:ui`, `pretest:ui:coverage`). The generated file is imported by `src/index.tsx`, `src/TestUtils.ts` (which 55 of 136 UI test files import), `.storybook/preview.ts` and 3 stories, with an ESLint `no-unresolved` ignore, a coverage exclude and a `.gitignore` entry to paper over it. **This is the hiccup that fails tests whenever vitest is run without the hook**, exactly what happened in my baseline run (§2). |
| Real usage | Small: 6 components (`graph/ArtifactNode`, `graph/ExecutionNode`, `graph/SubDagNode`, `navigators/PipelineVersionCard`, `tabs/RuntimeNodeDetailsV2`, `tabs/StaticNodeDetailsV2`), the SideNav helper string `flex flex-row flex-shrink-0` (8 renders, 127 snapshot occurrences), two `z-20` wrappers in `pages/PipelineDetailsV2.tsx:79` and `pages/RunDetailsV2.tsx:573`, one story. About 68 className sites, ~110 distinct utilities, 10 live custom `mui-*` palette tokens (12 referenced, 2 only inside a commented block; the config defines 32). typestyle (70 files) and MUI (68 files) are the dominant styling systems. |
| Hidden dependency | Tailwind's preflight is the app's **only** global CSS reset: `src/CSSReset.tsx` exists but its import is commented out at `src/index.tsx:17`, and `CssBaseline` is used nowhere. Removing Tailwind is therefore a whole-app visual change, not a 10-file port. |
| Effort / risk / priority | M / medium / P2 |
| Verification | done. Registry facts all confirmed. Codebase verifier corrected: the config is `vite.config.mts:59` (not `vite.config.ts:57`); two extra `z-20` consumers; StaticNodeDetailsV2 is 8/8 Tailwind sites; Storybook must **not** get a second `tailwindcss()` (builder-vite merges `vite.config.mts`); and one real regression the research had ruled out (below). |

**Options**

- **A. Upgrade to 4.3.3 with `@tailwindcss/vite` (recommended).** Deletes the prebuild step, the generated file and every special case around it; drops `postcss`, `autoprefixer` and `browserslist` as direct devDependencies (Tailwind 4 prefixes and nests via its built-in Lightning CSS; `postcss` stays in the tree as Vite's own dependency). Cost: raises the browser floor to Safari 16.4 / Chrome 111 / Firefox 128, which conflicts with `build.target: 'es2015'` in `vite.config.mts:59` and the `browserslist` key. Vite 8's own default target (`baseline-widely-available` = Chrome 111 / Edge 111 / Firefox 114 / Safari 16.4 / iOS 16.4) already equals Tailwind's floor, so the repo's es2015 setting is the CRA-era outlier, not Tailwind. 1 to 2 engineer-days including the visual pass.
- **B. Bump to 3.4.19.** Lockfile-only for `tailwindcss`/`postcss`/`autoprefixer` (their ranges already admit the latest), un-pin `browserslist`. Zero source change, keeps every special case. Half a day. Only sensible as a stop-gap if the browser-floor decision stalls.
- **C. Remove Tailwind.** Port ~68 sites to typestyle/MUI `sx`, restore a reset (`CSSReset.tsx` is a 2010-era Meyer reset, not equivalent; or add MUI `CssBaseline`), full-app visual regression pass. 3 to 5 days, highest visual risk. Defer; revisit after MUI 9.

**What breaks under Option A (all verified against the code)**

1. `@tailwind base/components/utilities` → `@import "tailwindcss"`; the `tailwindcss build` CLI no longer exists (moved to `@tailwindcss/cli`). Delete the five `pre*` scripts and `build:tailwind`.
2. Removed utilities: `flex-shrink-0` → `shrink-0` (`SideNav.tsx:41`, dead duplicate at `Css.tsx:385`), `flex-grow` → `grow` (`ArtifactNode.tsx:40`). Regenerate `SideNav.test.tsx.snap` and `Router.test.tsx.snap` (127 occurrences).
3. Default ring changed from 3px blue-500 at 50% to 1px currentColor: `focus:ring` at `ArtifactNode.tsx:36`, `ExecutionNode.tsx:48`. Set `--default-ring-color` (to `rgb(59 130 246 / 0.5)`, the value the generated CSS has today) and width in `@theme`.
4. **Default border colour gray-200 → currentColor.** `SubDagNode.tsx:69` (the expand button) has `border-2` with no rest-state colour and relies on the v3 default; under v4 every SubDAG expand button gets a dark 2px border. Add `border-gray-200` or `--default-border-color`. (The research finding had asserted this change "does not bite"; the codebase verifier refuted it.)
5. Preflight: `button { cursor: default }`. The three Tailwind graph nodes are raw `<button>` elements with no cursor utility (`ArtifactNode.tsx:34`, `ExecutionNode.tsx:46`, `SubDagNode.tsx:44`; also `ROCCurve.tsx:377`, `RuntimeArtifactComparison.tsx:482/494/524`, `SubDagLayer.tsx:131`). MUI buttons set their own cursor and are shielded. Add the upgrade guide's `@layer base { button:not(:disabled), [role="button"]:not(:disabled) { cursor: pointer } }`.
6. Default gray/blue palette moved to oklch: slight shade shifts in `SubDagNode.tsx` and `PipelineVersionCard.tsx`. The custom `mui-*` hex tokens are unaffected if ported verbatim to `@theme`.
7. `hover:` variants are wrapped in `@media (hover: hover)` (7 sites in the three graph nodes): no hover effects on touch devices unless `@custom-variant hover (&:hover)` is added. Recommend accepting the new behaviour.
8. `tailwind.config.js` is no longer auto-loaded: convert to `@theme` (the `spacing` extension is redundant in v4; `w-136` keeps working), drop `darkMode: 'media'` (v4 default) and the dead `variants.extend` block. Prune the ~21 unused colour tokens rather than porting all 32.
9. Automatic source detection scans the whole project (server, mock-backend, generated code) and the v3 JIT already emits ~25 false-positive utilities from words in `src/generated/**`. Use `@import "tailwindcss" source(none); @source "../src"; @source "../index.html";`.
10. Lightning CSS is Vite 8's default CSS minifier and `build.cssTarget` inherits `build.target` (es2015). Vite documents that `cssTarget` overrides `css.lightningcss.targets` during minification, so feeding v4 output (`@property`, `color-mix()`, oklch, `@layer`) through an es2015 target is untested here. Raise `cssTarget` (or drop `build.target` and inherit Vite's default) in the same PR; `tailwindcss({ optimize: false })` is the documented escape hatch if the plugin's own optimise pass conflicts (GitHub discussion #19530).
11. Consumers of the generated file: repoint `src/index.tsx:18`, `.storybook/preview.ts:3` and the 3 stories to `src/tailwind.css`; **drop** the import from `src/TestUtils.ts:20` (no test asserts Tailwind computed styles, and `test.css: true` in `vitest.config.mts` has no other consumer). Remove the ESLint ignore (`eslint.config.cjs:86-91`), the coverage exclude, `.gitignore:39`, the dead export at `Css.tsx:384-386` and the stale comment at `Css.tsx:21-25`.
12. Storybook: `@storybook/builder-vite` loads and merges `vite.config.mts`, so `tailwindcss()` added there is inherited; do not add it again in `.storybook/main.ts` (the existing `src` alias in `viteFinal` is redundant for the same reason).
13. Docker: `npm run build` no longer triggers a prebuild; `@tailwindcss/oxide` ships a `linux-x64-gnu` binary for the `node:24-slim` build stage (the alpine runtime stage never installs frontend devDependencies, so musl is irrelevant).

**Steps (Option A)**: its own PR after MUI 9, not bundled. Baselines first (`visual:baseline`, green `test:ui`). Decide and record the browser floor (`cssTarget`). `npm i -D tailwindcss@^4.3.3 @tailwindcss/vite@^4.3.3 && npm rm postcss autoprefixer browserslist`. Run `npx @tailwindcss/upgrade` on a clean branch (it depends on native `tree-sitter` bindings; confirm it runs on Node 24 first) and review its diff; it does not touch the npm scripts, ESLint/coverage/.gitignore special cases, Vite configs or snapshots. Apply items 3 to 5 by hand. Wire the plugin in `vite.config.mts` only. Repoint imports (item 11). Delete the scaffolding. `npm run lint && npm run typecheck && npm run test:ui -- -u` (SideNav/Router snapshots), `vite build` (check `@layer theme, base, components, utilities` and `--color-mui-*` in the output, no Lightning CSS warnings), `npm run build:storybook`, then `visual:current` + `visual:diff` on graph nodes, PipelineVersionCard and any raw buttons.

### 5.3 Test and build toolchain: Vitest 4 → 5, jsdom 24 → 30, plugin-react 5 → 6

| Package | Locked | Latest | Verdict |
|---|---|---|---|
| `vitest` (frontend and server) | 4.1.11 | 5.0.1 (2026-09-15) | bump-major |
| `@vitest/coverage-v8` (both) | 4.1.11 | 5.0.1 | lockstep with vitest (exact peer) |
| `jsdom` | 24.1.3 (2024-08-25) | 30.1.0 (2026-09-17) | bump-major, **six majors**, blocked by the Node pin (below) |
| `@vitejs/plugin-react` | 5.2.0 | 6.1.1 | bump-major, mechanical (Babel removed; repo calls `react()` with no options) |
| `vite` | 8.0.16 | 8.3.0 | bump-minor (server lock already resolves 8.2.2); 8.1.0 carries one breaking change, `server.hmr` renamed to `server.ws`, which this config does not set |
| `rollup-plugin-visualizer` | 5.14.0 | 7.1.1 | bump-major, mechanical (ESM-only, Node ≥ 22; 7.1.1 fixed types for Vite-8-without-rollup users, which is this repo) |
| `pixelmatch` | 5.3.0 | 7.2.0 | bump-major, mechanical (ESM default export unchanged) but diff counts change: recapture `.visual/baseline` |
| `playwright` | 1.58.0 | 1.63.0 | bump-minor; also `scripts/ui-smoke-test`; rerun `npx playwright install chromium` |
| `@testing-library/react` 16.3.3, `jest-dom` 7.0.1, `user-event` 14.6.7, `storybook`/`@storybook/react-vite`/`addon-links` 10.6.0 | current | current | keep. Storybook 11 exists only as `11.0.0-alpha.1`; nothing in its migration notes blocks this repo. |
| `@testing-library/dom`, `tsx` | 10.4.1 / 4.23.12 | 10.4.2 / 4.23.13 | patch |

Effort / risk / priority: M / medium / **P1**. Verification: done (registry 58 claims / 48 confirmed; codebase 53 / 41; corrections folded in); related facts confirmed by neighbouring clusters: Vitest 5's optional peer `@types/node ^22 || >=24` makes the current `@types/node` 20 a hard blocker, so §5.9's type bump must precede Phase 1; jsdom 25 to 29 install on Node 24.14.0, only 30 needs 24.15.

**Two defects that exist today, independent of any bump**

1. `vitest.config.mts` still sets `test.environmentMatchGlobs`. Vitest 4 removed that option (the string does not occur anywhere in the installed package), and the line was dead from the day it was added (#12817 landed on Vitest 4.0.17), so the three `scripts/**` tests have always run under jsdom instead of Node; they pass only because the Node modules they load (`fs`, `os`, `path`, `child_process`, `playwright`) remain reachable. Fix now: delete the key and add `// @vitest-environment node` to `scripts/generate_openapi_typescript_fetch.test.js`, `scripts/ui-smoke-test/capture-screenshots.test.js`, `scripts/ui-smoke-test/seed-data.test.js` (or use `test.projects`).
2. `.storybook/preview.ts` and the 7 stories import types from the transitive renderer `@storybook/react`; the three `tsconfig.json` `paths` entries that once made that resolve are redundant since `moduleResolution: bundler` (#14089) because the hoisted package resolves on its own. Storybook 9+ wants imports from the framework package `@storybook/react-vite` (which re-exports the renderer). One `sed`, then delete the three `paths` entries.

**Vitest 5 changes that touch this repo** (from its migration guide, checked against the code)

- `clearMocks` now defaults to `true`. 100 lines reference `.mock.calls` across UI and server tests; 170 clear/reset/restore calls already exist across 67 files. Either accept the default and fix order-dependent tests, or set `clearMocks: false` in both configs. Recommendation: accept it; run the suite once with `clearMocks: true` on Vitest 4 beforehand to see what leaks. Note that test files are neither linted (ESLint ignores `*.test.*`) nor type-checked (both tsconfigs exclude tests), so Vitest 5 type changes surface only at runtime.
- Unawaited `.resolves`/`.rejects` now fail: 149 sites scanned, 0 unawaited (5 in `server/minio-helper*.test.ts` are stored then awaited, which is fine).
- `vi.mock`/`vi.hoisted` must be top-level: 0 violations.
- Fake timers also mock `Temporal`: 40 call sites in 9 files, no `Temporal` usage.
- Reports move under `.vitest/`: add it to `.gitignore`.
- `server.deps.inline` is marked deprecated but still works; the `@xyflow/react`/`@xyflow/system` entry (added in #12945 without a recorded reason) may no longer be needed. Try removing it and run the 6 xyflow test files.
- Node floor `^22.12 || ^24 || ≥26`, Vite peer `^6.4 || ^7 || ^8`: both satisfied.

**jsdom 24 → 30**

- **Hard prerequisite**: jsdom 30.x declares `engines: ^22.22.2 || ^24.15.0 || >=26.0.0`. The repo pins Node **24.14.0** exactly with `engine-strict=true`, so `npm ci` refuses jsdom 30 until the Node patch bump in §5.9 lands. 29.1.1 is the ceiling on 24.14.0.
- Behaviour changes between 24 and 30 that can move test results: 28 overhauled resource loading; 27 switched the selector engine (nwsapi → `@asamuzakjp/dom-selector`), derives the UA stylesheet from the HTML Standard (changes `getComputedStyle`), makes `element.click()` dispatch a `PointerEvent`, and turned Window data properties into accessors; 29 rewrote CSSOM; 30 added `CSS.escape`/`CSS.supports` and converts computed lengths to px. Expect a snapshot-review pass (44 snapshot files, 243 snapshot assertions) and re-checks of `toBeVisible`/`toHaveStyle`/role queries.
- The shims in `src/vitest.setup.ts` stay: jsdom HEAD still has no `DOMMatrixReadOnly`, `ResizeObserver`, `DataTransfer` or `Worker`, and still leaves `MouseEvent.view` null (jsdom#3935, closed not-planned). On jsdom 24 the `MouseEvent.view` patch already runs its `defineProperty` path (the direct assignment throws on the getter-only prototype), so 27's accessor change does not alter it; the xyflow test files (4 of the 13 importers are tests) exercise it. `URL.createObjectURL` and `Worker` have no first-party callers; the shims exist for `ace-builds`, reached from 4 source files.
- `happy-dom` 20.14.5 was evaluated as an alternative: faster, implements `DOMMatrixReadOnly`/`ResizeObserver`/`DataTransfer`, but lacks `Worker` and `URL.createObjectURL`, serialises differently (would churn all 44 snapshot files), and has documented `user-event` gaps (file upload). Only 20.x is acceptable (CVE-2025-61927 in ≤ 19). Do not switch without a spike branch that runs the full suite under `environment: 'happy-dom'` and counts failures.

**Order**: (1) fix the two defects above, (2) Node patch bump + `@types/node` ^24 (§5.9), (3) Vitest 5 + coverage-v8 5 in frontend and server together (nothing in CI enforces frontend/server lockstep, but the root `test:ci` runs the server tests, so keep them aligned by hand), confirm `coverage/vitest/coverage-summary.json` still feeds `scripts/coverage-baseline.mjs`, (4) jsdom 30 with a deliberate snapshot review, (5) plugin-react 6 / visualizer 7 / pixelmatch 7 (recapture visual baseline), (6) Vite 8.3, playwright 1.63, tsx, dom minors, tighten `storybook` range from `^10.2.0` to `^10.6.0`. Also extend `TOOLCHAIN_PACKAGES` in `scripts/check-lockfile-drift.mjs` with jsdom, `@vitest/coverage-v8`, vite, `@vitejs/plugin-react` and the Testing Library packages so silent lock drift in test-affecting packages fails PRs.

### 5.4 TypeScript 6.0.3 vs 7.0.2, ESLint and formatting

| Package | Locked | Latest | Verdict |
|---|---|---|---|
| `typescript` (frontend, server) | 6.0.3 | 7.0.2 (2026-07-08) | **keep 6.0.3 in frontend**; optional pilot of 7 in `frontend/server` |
| `@typescript-eslint/parser` + `eslint-plugin` | 8.65.0 (exact pin) | 8.70.0 | bump together |
| `eslint` | 10.8.0 | 10.11.0 | bump-minor (no breaking changes; three recommended rules tightened, none of the patterns occur) |
| `prettier` (frontend and server) | 3.8.1 | 3.9.8 | bump-minor in both manifests, expect a mechanical reformat commit. Nothing in CI enforces the two copies to match: `check-lockfile-drift.mjs` diffs one lockfile against its own base ref and runs only in `frontend/`; the reason to keep them aligned is that root `format:check` formats `server/**/*.ts` with the root prettier while `npm run format` inside `server/` uses the server copy (the #13026 mismatch) |
| `globals` | 16.5.0 | 17.12.0 | bump-major, no effect (`no-undef` is off) |
| `semver`, `yaml` | 7.7.4 / 2.8.3 | 7.8.5 / 2.9.1 | bump-minor; re-run `check:react-peers` (semver prerelease-range semantics changed); `yaml` 3.0 (prerelease) will drop the default import used in `mock-backend/fixed-data.ts:16` |
| `@eslint/js`, `eslint-plugin-import-x`, `eslint-plugin-react-hooks` | current | current | keep |
| `browserslist` (direct devDep, exact pin) | 4.28.8 | 4.29.0 | remove the direct pin (nothing imports it; goes away with Tailwind 4 anyway) |
| `tsconfig.json` `include: ["src", "stories"]` | | | `stories` directory does not exist; stale entry, harmless |

Effort / risk / priority: M / low / P2. Verification: done (registry 63 claims / 56 confirmed; codebase 43 / 34; corrections folded in).

**Why TypeScript 7 is not adoptable yet.** 7.0.2 is the native (Go) compiler. It ships no JavaScript API (the package exports only `./lib/version.cjs`, `./package.json` and `unstable/*`; `bin` has only `tsc`, no `tsserver`; engines `node >=16.20`), and every `@typescript-eslint/*` 8.70.0 package declares `peer typescript >=4.8.4 <6.1.0` and `require()`s the TS API at load time. With TS 7 installed, `npm run lint:ui` and `lint:server` fail outright (typescript-eslint #12445, filed against 7.0.1-rc with the same exports shape; #12518/#12720 closed "not planned"; the open tracking issue #10940 cites ESLint's lack of async parsers and the need to bridge AST/type information from the Go process, and makes no commitment about landing in 8.x vs. a new major. TS 7.1, which is expected to bring a new API, is planned for 2026-11-24.) 6.0.3 is the terminal JS-based release; there is no 6.1.

The good news: all four tsconfigs (`frontend`, `frontend/tsconfig.test.json`, `frontend/server`, `frontend/mock-backend`) parse on 6.0.3 with **zero** TS 6.0 deprecation diagnostics, so nothing TS 7 removes (`baseUrl`, `moduleResolution node10/classic`, `target es5`, `module amd/umd/system`, `outFile`, `esModuleInterop=false`) is in use, and the code-level syntax TS 6.0 deprecates (`module Foo {}` namespaces, `import ... assert`, `/// <reference no-default-lib>`) has 0 occurrences. (`rootDir` defaulting to the tsconfig directory and `types` defaulting to `[]` are already TS 6.0 behaviour.) When typescript-eslint supports 7.x, the move is a version bump. `frontend/server` has no typescript-eslint dependency in its own build (`tsc --project .`), so it is a low-risk TS 7 pilot (its 71-file program typechecks in about 3 s today; the frontend program is 428 files, 231 of them checked-in generated API clients that the `exclude` list does not keep out because they are imported, and typechecks in about 13 s, so TS 7's speed is not a pressing motive). **Add a Dependabot `ignore` for `typescript >= 7`**: `.github/dependabot.yml` has no ignore rules, so it will propose 7.0.2 as a grouped major PR that fails `npm ci` on the typescript-eslint peer range.

**Steps**: bump `@typescript-eslint/*` 8.70.0, `eslint` 10.11.0, `globals` 17 (and consider `globals.es2025` + `ecmaVersion: 'latest'`); lint is 0/0 today so promote the two React Compiler rules actually downgraded in `eslint.config.cjs` (`set-state-in-effect`, `preserve-manual-memoization`) back to the recommended `error` (the `incompatible-library`/`unsupported-syntax` lines restate the recommended `warn` and can go); bump `prettier` 3.9.8 in both manifests and commit `npm run format` separately (3.9.0 changes non-null assertion consistency, union condensing, JSDoc placement; 3.9.7 fixed a template-literal idempotency regression from 3.9.6, so do not stop at 3.9.6; three `?.x!` sites in `PipelineDetails.tsx:594`, `NewRunV2.tsx:548,942` to eyeball); bump `semver`/`yaml` (the semver prerelease-range change has zero applicable sites: no React peer range in the lockfile uses a prerelease tag; note that `check:react-peers:18` already fails on the current tree because `react-dom` 19.2.8 and `react-router` 8 exclude React 18, so delete that script); drop the direct `browserslist` devDependency but **keep the `browserslist` key in `package.json` until Tailwind 4 lands**: autoprefixer reads it during `build:tailwind`. Re-run `npm run test:ci`. Also worth knowing: 136 test files and every `.mts`/`.mjs`/`.cjs` config are outside both ESLint and `tsc` scope, and `tsconfig.prod.json`/`tsconfig.test.json` are referenced by nothing.

### 5.5 React core 19.2.8 → 19.3.0 and react-router 8.4.0

| Package | Locked | Latest | Verdict |
|---|---|---|---|
| `react`, `react-dom` | 19.2.8 | 19.3.0 (2026-09-09) | bump-minor, **in lockstep** (`react-dom@19.3.0` peers `react ^19.3.0`); pulls `scheduler` 0.28 |
| `@types/react`, `@types/react-dom` | 19.2.14 / 19.2.4 | 19.3.0 | bump-minor, in lockstep (`@types/react-dom@19.3.0` peers `@types/react ^19.3.0`); additive types only |
| `react-router` | 8.4.0 | 8.4.0 (2026-09-15) | **current**. `react-router-dom` is correctly absent (removed in v8). Peer `react >=19.2.7`, engines `node >=22.22`. |
| `babel-plugin-react-compiler` | not installed | 1.0.0 | investigate later (see below) |

Effort / risk / priority: S / low / P2. Verification: done (registry pass 46 claims / 42 confirmed; codebase pass 67 claims / 61 reproduced exactly; corrections folded in).

**React 19.3** is additive (ViewTransition, `addTransitionType`, Fragment refs, `react-dom` `browser()`, Trusted Types) with no deprecations or removals. All 47 `react`/`react-dom` peer declarations in the lockfile accept 19.3.0; the peer gate stays green with an empty allowlist. One behaviour change to re-test: resize-event updates are now batched to the next frame, and `SideNav.tsx:247` calls `setState` from a `resize` listener (`SideNav.test.tsx` covers it). Type diff is additions only; there are 0 zero-argument `useRef()` calls, 0 `defaultProps` on function components, 0 `propTypes`, 0 legacy lifecycles, 0 `findDOMNode`/`ReactDOM.render`. The one bare `JSX.Element` (`atoms/ErrorBoundary.test.tsx:22`) is in a test file excluded from `tsc`. `.github/dependabot.yml` already covers npm with a `mui` group; a matching `react` group (`react`, `react-dom`, `@types/react`, `@types/react-dom`) is worthwhile hygiene, though not strictly required: Dependabot's lockfile-only PR #14154 carried `react` along with `react-dom` because the ranges admit both. Note that `@types/react` has sat at 19.2.14 since March with no Dependabot PR despite four newer 19.2.x patches.

**Router**: nothing to do. Declarative mode (`HashRouter` in `index.tsx`, `Routes`/`Route`/`Navigate`/`matchPath` in `Router.tsx`) is fully supported in v8; the two trailing-`?` optional segments (`/pipelines/details/:pid/version/:vid?`, `/pipelines/details/:pid?`) are documented v8 syntax handled by `compilePath`, and `Router.test.tsx` (20 tests) covers them, the encoded-`%2F` pipeline ids, and the legacy `/executions` redirects. 51 files import from `react-router`; 0 from `react-router-dom`; 0 `UNSAFE_*` usages. react-router 8 is ESM-only (`"type": "module"`, no CJS build), which is fine for Vite/Vitest but means nothing CommonJS (the server, `tsconfig.test.json` with `module: commonjs`) may import it. Three test files import `RouterProvider` from the package root rather than the documented `react-router/dom`; both work. Keep the `RoutePageElement` `matchPath` + `decodeURIComponent` shim (works around v8 `useParams` decoding `%2F`) and add a regression test against `useParams` so a router-side fix can retire it. Ten production call sites in 6 files build links from the two optional-segment patterns via `.replace()`; `RunList.tsx:259` strips the literal `:pid?` token, so any change to that syntax must update it too.

**React Compiler**: not now. The lint side is ready (`eslint-plugin-react-hooks` 7 compiler rules are enabled and produce 0 findings in CI), but with inline suppressions ignored there are 8 diagnostics in 5 files (4 `exhaustive-deps`, 3 `preserve-manual-memoization` in `NewRunV2.tsx:318-335`, 1 `refs` in `lib/KubeflowClient.tsx:75` reading a ref during render), each a compiler bailout to fix first. The class components (66 files / 17,490 lines counting the `Page` and `Viewer` subclasses; 39 files / 8,605 lines extend `React.Component` directly) would be skipped entirely, so app-wide gains are limited. The wiring also depends on the `@vitejs/plugin-react` decision: 5.x takes `react({ babel: { plugins } })`, 6.x removed Babel (use `@rolldown/plugin-babel` + `reactCompilerPreset()` or the experimental oxc compiler). Pilot later in `compilationMode: 'annotation'` on `pages/functional_components/*` and `hooks/*`.

**Modernisation backlog (not required for version support)**: 66 class-component files, 17,490 lines (the 39 listed below extend `React.Component` directly; the other 27 are the 20 pages extending `Page` and 7 viewers extending `Viewer`, which follow once those bases are replaced). Smallest first: `icons/*` (6 files), `atoms/BusyButton`, `components/CollapseButton`, `Metric`, `SidePanel`, `StaticNodeDetails`, `lib/BuildInfo`, `lib/KubeflowClient` provider, `pages/RunListsRouter`, `pages/Page` base; mid-size `Banner`, `Toolbar`, `ExperimentList`, `NewRunParameters`, `PlotCard`, `LogViewer`, `CompareTable`, `MD2Tabs`, `PipelineVersionList`, `ResourceSelector`, `RecurringRunsManager`; large `CustomTable` (788), `SideNav` (683), `ArtifactDetails` (598), `RunList` (593), `Trigger` (530), `Router.tsx` `RoutedPage` (480), `Graph` (423), `RecurringRunList` (407), `NewRunParametersV2` (404), `UploadPipelineDialog` (362). The two class error boundaries (`atoms/ErrorBoundary`, `Graph.tsx` `GraphErrorBoundary`) must stay classes; `NavigationErrorBoundary` is already a function wrapper around the first.

### 5.6 Graph and plots: recharts, dagre, @xyflow/react

| Package | Locked | Latest | Verdict |
|---|---|---|---|
| `recharts` | 3.7.0 | 3.10.1 (2026-07-25) | bump-minor; nothing `ROCCurve.tsx` uses was removed or deprecated (checked against 3.10.1 types), but ~510 `recharts-*` nodes across 3 snapshot files will churn |
| `react-redux` (transitive, pinned by `overrides.recharts`) | 9.2.0 exact | 9.3.0 | relax the override to `^9.2.0` or delete it; it was a React 17 → 19 peer workaround (#12829, #13082) and now only blocks 9.3.0 |
| `react-is` (recharts peer) | resolves hoisted 16.13.1 | 19.3.0 | add `react-is@^19` as a direct dependency; recharts' README requires it to match `react`, and 16.13.1 does not recognise React 19's element symbol (latent: `isFragment` returns false) |
| `dagre` + `@types/dagre` | 0.8.5 (2019-12-03) / 0.7.54 | 0.8.5 | **replace** with `@dagrejs/dagre` ^3.1.1 (2026-08-08; own types, ESM+CJS, no lodash) and `@dagrejs/graphlib` ^4.0.5; remove `@types/dagre` |
| `@xyflow/react` | 12.11.6 | 12.11.6 (2026-09-01) | current; no v13 exists yet, but `@xyflow/system` has a `1.0.0-next.3` prerelease line (Aug 2026), the usual precursor to a React Flow major. Watch it. |
| `d3-dsv` + `@types/d3-dsv` | 3.0.1 / 3.0.7 | current | keep (one `csvParseRows` call in `lib/OutputArtifactLoader.ts`) |

Effort / risk / priority: M / medium / P2. Verification: done (registry 44 claims / 38 confirmed; codebase 43 / 35; every version, date and peer range reproduced; corrections folded in).

**"Plots break" explained.** recharts is imported by exactly one production file, `components/viewers/ROCCurve.tsx` (a 474-line class component extending `Viewer`, using `LineChart`, `Line`, `XAxis`, `YAxis`, `CartesianGrid`, `ReferenceArea`, `ReferenceLine`, `ResponsiveContainer`, `Tooltip`). Its test renders recharts for real (no mock), stubs `ResizeObserver` per test (a shared `mockResizeObserver` helper already exists in `TestUtils.ts:196-239` and is used by six page tests), and keeps a **4,325-line SVG snapshot**; `ViewerContainer.test.tsx.snap` (144 recharts nodes) and `CompareV1.test.tsx.snap` (42) also encode recharts markup. Every React or recharts bump changes SVG attributes and fails these snapshots (the same files churned in the React 18, React 19 and StrictMode PRs, and again in the MLMD removal #13986 and the router v8 PR #14417, so the churn is not tied to React alone). `src/testUtils/muiSnapshot.ts:75-87` renumbers recharts ids to keep them stable across runs; re-check it if 3.8 to 3.10 changed the id scheme. That is the symptom you saw; the fix is to shrink the snapshot surface (one full-DOM snapshot for the single-series case, targeted assertions such as path count / stroke colours / tick count for the rest) so upgrades stop rubber-stamping 4,000-line diffs. Every other viewer (ConfusionMatrix = table, HTMLViewer = iframe, MarkdownViewer = markdown-to-jsx, PagedTable = MUI Table, Tensorboard, VisualizationCreator = react-ace) has tests and needs no chart library. Legacy `react-vis`/`react-svg-line-chart`/`d3` are fully gone; the only remnant is a `.rv-xy-plot` selector in `scripts/ui-smoke-test/capture-screenshots.js:274,279`.

**dagre replacement** (12 importing files: 6 source `components/Graph.tsx`, `lib/GraphTypes.ts`, `lib/StaticGraphParser.ts`, `lib/WorkflowParser.ts`, `lib/v2/StaticFlow.ts`, `pages/PipelineDetails.tsx` and 6 tests; plus `pages/RunDetails.tsx:123-124` and `pages/PipelineDetails.tsx:657`, which use `dagre.graphlib.Graph` as a type **without importing dagre**, relying on `@types/dagre`'s `export as namespace dagre` UMD global. `@dagrejs/dagre` declares no such namespace, so removing `@types/dagre` breaks both files until explicit type imports are added; `pages/PipelineDetailsV1.tsx:35,85-86` consumes `DagreGraph` and is a compile-time dependent too). Three real API deltas in `@dagrejs/dagre` 3.x: (1) the `graphlib` export has no `json`, so `StaticGraphParser.ts:299` (`transitiveReduction`) and its test must import `{ json }` from `@dagrejs/graphlib`; (2) `Graph` generics are `<GraphLabel, NodeLabel, EdgeLabel>` instead of `@types/dagre`'s `Graph<NodeData>`, so `new graphlib.Graph<GraphNodeData>()` in `WorkflowParser.ts:51`, `StaticGraphParser.ts:216` and `DagreGraph` in `GraphTypes.ts:53` would silently type the graph label; (3) node `x`/`y` and edge `points` are optional (`width`/`height` on `NodeLabel` are **required**), so `Graph.tsx:157-170,180,206,267,382` and `v2/StaticFlow.ts:434-438` need narrowing after `layout()`, while the input type `GraphNodeInput` (`GraphTypes.ts:46-50`, optional width/height) cannot simply be intersected with `NodeLabel`; keep a separate input type and a post-layout type. All current `setNode` calls already pass width and height. All three import styles in use (namespace, default, named) remain valid. Layout coordinates may differ slightly from 0.8.5 (`Graph.test.tsx.snap` embeds 11 layouts with 77 coordinate lines); spike `layout()` on a `json.read` graph before committing (`@dagrejs/dagre` inlines its own `Graph` class and never imports `@dagrejs/graphlib` at runtime, so class identity is a question). Ask upstream whether the v1 dagre renderer (`Graph.tsx`, `PipelineDetailsV1`, v1 `RunDetails`) is slated for removal; if so the migration shrinks to `StaticFlow.buildGraphLayout` and `PipelineDetails.tsx`.

**xyflow**: keep 12.11.6. The `server.deps.inline` entry in `vitest.config.mts` and the `DOMMatrixReadOnly` / `MouseEvent.view` shims in `vitest.setup.ts` are still required (`@xyflow/system` references `DOMMatrixReadOnly`; d3-drag reads `event.view`). The two shims are already documented in-repo; only the `server.deps.inline` line lacks a comment. `PipelineDetailsV2.test.tsx:233-235` and `RunDetailsV2.test.tsx:1496-1531` work around the same `event.view` issue by using `fireEvent` instead of `user-event`. `scripts/ui-smoke-test/capture-screenshots.js` also hard-codes the xyflow internal class `.react-flow__node` (5 lines).

**Steps**: recharts 3.10.1 in an isolated PR with visual review of the three snapshot files (expect `reselect` to move in the lockfile too); relax the `react-redux` override; add `react-is` matching the installed React version; then the dagre swap (`sed` the 12 import specifiers, explicit imports in `RunDetails.tsx`/`PipelineDetails.tsx:657`, `json` from `@dagrejs/graphlib`, retype `GraphTypes.ts` with separate input/laid-out types, narrow post-layout fields), run `Graph`, `StaticGraphParser`, `WorkflowParser`, `StaticFlow`, `DynamicFlow`, `PipelineDetails*`, `RunDetails` tests and add a unit test for `buildGraphLayout`; delete the `.rv-xy-plot` selectors. Consider a global `ResizeObserver` stub in `vitest.setup.ts` so `ResponsiveContainer` can render in any test.

### 5.7 Remaining client libraries: replace one, delete thirteen, bump seven

Effort / risk / priority: M / medium / P2 (the hygiene part is S / low and can go first). Verification: done (registry 45 claim groups / 34 confirmed; codebase 46 / 25; every current/latest version, date, peer and deprecation reproduced; corrections folded in).

**Replace**

- **`react-virtualized` 9.22.6 → `react-window` ^2.3.1.** react-virtualized's last publish and last commit are both 2025-01-20 and its README steers users to react-window. It is used in exactly one component, `components/LogViewer.tsx` (`List`, `AutoSizer`, a custom symmetric `overscanIndicesGetter` with `overscanRowCount=400` so off-screen log lines stay selectable, an `overflow: auto !important` + `containerStyle` hack for horizontal scrolling of `white-space: pre` lines, `scrollToRow` for follow-new-logs). react-window v2 covers all three natively (symmetric `overscanCount`, root `overflow-y: auto` with absolutely positioned full-width rows, native `onScroll` + `scrollToRow({ index, align: 'end' })`, self-measuring so `AutoSizer` goes away). Rewrite is ~40 lines in the component, but `LogViewer.test.tsx` needs restructuring, not just new expectations: 17 of its 19 tests construct `new LogViewer(...)` and call the private `_rowRenderer`, which the rewrite deletes; only one snapshot contains react-virtualized markup. Reuse the existing `mockResizeObserver()` helper from `TestUtils.ts` for the two tests that render the list. Verify horizontal scroll and 400-row selection in a real browser. The `react-virtualized` → `dist/commonjs` alias in `vite.config.mts` was added for 9.21.0's broken ESM import, which 9.22.6 fixed; delete the alias with the library. Alternatives considered: `@tanstack/react-virtual` (headless, more code), `react-virtuoso` (pixel-based overscan, horizontal scrolling unverified).

**Delete (no source changes; verify with `typecheck`, `lint`, `vite build`)**

| Package | Why it is dead |
|---|---|
| `fs` 0.0.1-security (devDep) | npm placeholder package squatting the builtin's name; contains no code. First appears in the KFP 2.0.0-alpha.2 manifest (2022), almost certainly an accidental `npm install fs`. |
| `url` 0.11.4 | Added 2026-03-01 (#12940) for swagger-codegen clients that imported `url.parse`; those clients were regenerated with OpenAPI Generator three days later (#12817). 0 imports, 0 modules in today's build sourcemap. Confirm no "url externalized" warning on one `vite build`. |
| `http-proxy-middleware` (client manifest) | CRA-era leftover for the `proxy` field. Dev proxying is `server.proxy` in `vite.config.mts`. 0 client imports. The server keeps its own copy. |
| `@types/express` (client manifest devDep) | no consumer in `src/`, `scripts/`, the Vite/Vitest/ESLint configs or `.storybook/`; `server/` and `mock-backend/` each declare their own copy (found by the completeness pass). |
| `react-textarea-autosize` | 0 imports anywhere (bumped in #13025 without a call site). |
| `lodash.debounce`, `lodash.flatten`, `lodash.groupby`, `lodash.isfunction` | 0 imports (the code imports from `lodash`); the last `lodash.groupby` import went with MLMD in #13986. |
| `@types/lodash.groupby` | types for an unused package, and listed under production `dependencies`, dragging `@types/lodash` into the prod tree. |
| `@types/pako` 3.0.0 | npm-deprecated stub ("pako provides its own type definitions"); contains no `.d.ts`. |
| `@types/js-yaml` 3.12.3 | js-yaml 5 ships its own `dist/js-yaml.d.ts`; 3.12.3 declares `safeLoad`/`safeDump`, which no longer exist. Latest 4.0.9 is for js-yaml 4 only. Delete, do not bump. |
| `proto3-json-serializer` | 0 references anywhere in `frontend/`; nests a duplicate `protobufjs@7`. |
| `src/__mocks__/typestyle.js` | dead Jest-era manual mock (no `vi.mock('typestyle')` anywhere, snapshots carry real typestyle hashes); delete. Same PR: fix the stale CRA-era comment in `scripts/start-proxy-and-server.sh` ("proxy field") and the "react-dropzone v14" comment in `atoms/DropzoneArea.tsx:50`. |

**Bump**: `js-yaml` 5.4.2 (8 production files + 15 tests in `src`, 4 more in `server/`, so bump both manifests), `pako` 3.0.2, `markdown-to-jsx` 9.10.3, `react-dropzone` 20.1.2, `ts-proto` 2.12.4 (and move it to `devDependencies`; it is a protoc plugin used only by `scripts/pipelinespec.sh` and `scripts/k8splatformspec.sh`), `autoprefixer` 10.6.1 / `postcss` 8.5.28 (only until Tailwind 4 removes them). Type drift: `@types/lodash` is locked to a 2018 build (4.14.119) through the unbounded `>=4.14.117` range, move to `^4.17.25` (the `chain(...)` in `lib/CompareUtils.ts:43-48` is the one place generics may tighten); `@types/node` `^20` → `^24` to match the Node 24 runtime (do not jump to 26; `NodeJS.Timeout` is used in `Tensorboard.tsx:90` and `RunDetails.tsx:191,1072` and still exists in 24). The latter is a **hard prerequisite for Vitest 5**, whose optional peer is `@types/node ^22 || >=24`. Removing `proto3-json-serializer` also takes `@types/node` out of the production dependency graph (its nested `protobufjs` 7 is the only prod package that depends on it).

**Keep**: `typestyle` 2.4.0 (70 files; last release 2022-08, framework-agnostic, no React coupling, coexists with Emotion via `StyledEngineProvider injectFirst`; it pins `csstype` 3.0.10 exactly, forcing eight nested `csstype` copies, types-only; replacing it is a separate XL project to couple with the MUI plan, not this sweep), `react-ace` 15 + `ace-builds` 1.44 (3 files: `Editor.tsx`, `VisualizationCreator.tsx`, `pages/PipelineDetails.tsx` with `mode-yaml`), `re-resizable` 6.11.2, `immer` 11 (recharts nests its own `immer` 10.2 and both copies are in the bundle), `runtypes` 7, `d3-dsv` 3, `protobufjs` 8.8 (see the ts-proto hazard below), `lodash` **≥ 4.18.1 as a security floor** (4.17.23 and 4.18.0 fixed GHSA-f23m-r3pf-42rh and GHSA-r5fr-rjxr-66jc; 4.18.0 itself is deprecated as a "Bad release", as is `lodash-es` 4.18.0). A `lodash-es` switch only removes the duplicate whole-library copy: `dagre`/`graphlib` require 240 `lodash/*` per-method modules, so lodash leaves the bundle only when dagre is replaced (§5.6); `chain()` in `CompareUtils.ts` must be rewritten first.

**Latent hazard**: `src/generated/**` is ts-proto **1.x** output (`import _m0 from 'protobufjs/minimal'`, `import Long from 'long'`; `long` is not even declared in `package.json`, it resolves only because protobufjs hoists it; last regenerated in #12839 on 2026-03-13) while the installed plugin is ts-proto **2.12.x**, whose output imports `@bufbuild/protobuf/wire` instead. Any `npm run build:pipeline-spec` or `build:platform-spec:kubernetes-platform` silently swaps the runtime protobuf library to an undeclared dependency. The drift came from Dependabot bump #14290 (2026-09-06). CI already runs `scripts/check-spec-generation.sh` with the 2.12.x plugin and typechecks the fresh output in a temp directory, which proves the 2.x toolchain compiles, but it never diffs against `src/generated`, so the drift is invisible to CI. Decide: (a) regenerate with 2.12.4, add `@bufbuild/protobuf@^2.15` as a runtime dependency, drop `protobufjs` and `long` (recommended; ts-proto 2.x defaults to `forceLong=number`), or (b) pin `ts-proto@1.x` and declare `long` (1.x ended at 1.181.2 in August 2024, so this pins a dead line).

### 5.8 Frontend server (`frontend/server`) and mock backend

Effort / risk / priority: M / medium / **P1** (this is where the open `npm audit` findings live). Verification: done (registry 61 claim groups / 53 confirmed; codebase 35 / 24; corrections folded in).

| Package | Locked | Latest | Verdict |
|---|---|---|---|
| `express`, `http-proxy-middleware`, `minio`, `js-yaml`, `@types/express` | 5.2.1 / 4.2.0 / 8.0.7 / 5.4.1 / 5.0.6 | current or patch (Express 5.x has had no release since 2025-12-01; the 4.x line still ships, 4.22.3 on 2026-09-14) | keep (see the router override below) |
| `@kubernetes/client-node` | 1.4.0 | 2.0.0 (2026-08-12) | bump-major: node-fetch → undici transport, Node 18/20/23 dropped, js-yaml 5 (dedupes with the server's copy and clears the **high** audit finding). All calls in `k8s-helper.ts` already use the 1.x object-parameter API, so it should be source-compatible; the transport change (proxy env, custom CA, keep-alive) is only visible in-cluster. |
| `tar-stream` + `@types/tar-stream` | 2.2.0 / 1.6.7 | 3.2.1 (bundled types exist only from 3.2.1, so pin `^3.2.1`; the tree already carries a nested `tar-stream` 3.1.7 under `tar-fs` via the k8s client, so the bump dedupes two majors) | bump-major with one real code change: v3 is streamx-based and `write(data)` takes no callback, so `minio-helper.ts:1204` (`extract.write(chunk, callback)`) would never call back and stall the tarball preview. Rewrite to `if (!extract.write(chunk)) extract.once('drain', cb) else cb()`; re-test `pipeline(pack, gzip, res)` in `handlers/artifacts.ts` and the directory-download integration tests. Remove `@types/tar-stream`. |
| `supertest` + `@types/supertest` | 4.0.2 (2019, **registry-deprecated**, as is its `superagent` 3.8.3; every `npm ci` in `server/` prints the notice) / 2.0.16 | 7.2.2 / 7.2.1 | bump-major; the API surface used in the 10 supertest files (`.expect` 288, `.get` 261, `.set` 45, `.end` 32, `.post` 23, `.send`/`.query` 12 each, `.parse` 6, `.delete`, `.buffer`, `.type`) is unchanged in superagent 10 (`form-data ^4.0.5`, `formidable 3`). Removes the only consumer of `form-data` 2.x and the legacy `formidable 1`/`readable-stream 2` copies. One type change: `app.test.ts:309,408,511,631` declare `requests.SuperTest<requests.Test>`, which no longer matches what `requests()` returns in `@types/supertest` 7 (use `requests.Agent` or `ReturnType<typeof requests>`); it only surfaces in editors because test files are outside the server `tsc` scope. |
| `google-auth-library` | 9.15.1 (`legacy-14` line) | 11.1.0 (2026-09-16) | bump-major: 10 moved to gaxios 7 (node-fetch v3, `Headers`, dual ESM/CJS), 11.0 requires Node ≥ 22, 11.1 silently moves to `gcp-metadata` 9 / `google-logging-utils` 2. `gcs-helper.ts:127-131` assumes `responseType: 'stream'` returns a Node `Readable`; add a `Readable.fromWeb` guard (line 124 already does this for the anonymous path). Unit tests mock `GoogleAuth` entirely, so verify against a real bucket. Clears the moderate `uuid`/`gaxios` audit findings. |
| `form-data` (direct dep) + two `form-data@…` overrides | 2.5.6 | 4.0.6 | remove after supertest 7: never imported; added for CVE-2025-7783; the only 2.x consumer is `superagent@3.8.3` under supertest 4. |
| `crypto-js` + `@types/crypto-js` | 4.2.0 / 3.1.47 | deprecated ("Active development of CryptoJS has been discontinued") | **replace** with `node:crypto`. One call: `k8s-helper.ts:83` SHA1 of a logdir → `createHash('sha1').update(logdir).digest('hex')` produces byte-identical hex, so existing Viewer CR names are stable. Add a unit test pinning one known hash. |
| `axios`, `lodash` (direct deps) | 1.18.0 / 4.18.1 | | remove: never imported by server code (global `fetch` is used; `minio` brings its own lodash). |
| `@types/tar` | 4.0.5 | deprecated stub | remove: there is no `tar` in the tree. |
| `@aws-sdk/credential-providers` | 3.980.0 | 3.1136.0 | bump-minor (one `fromNodeProviderChain` call). |
| `@types/node` | 22.19.7 | 24.x | bump to `^24` (runtime is Node 24); `mock-backend`'s lockfile still resolves a stale transitive `@types/node` 10.11.6. |
| `vitest`/`@vitest/coverage-v8`, `typescript`, `prettier` | 4.1.11 / 6.0.3 / 3.8.1 | | move in lockstep with the frontend root (§5.3, §5.4); `server/tsconfig.json` (`module`/`moduleResolution` `node16`, `types: ['node']`) is TS 7-ready. 23 top-level `vi.mock`, 0 nested; several hundred call-history assertions (`toHaveBeenCalledWith` 122, `not.toHaveBeenCalled` 139, `toHaveBeenCalledTimes` 54, and so on) to consider under Vitest 5's `clearMocks`; 11 of the 27 test files already clear or restore mocks, so 16 files need the audit. |
| `gunzip-maybe`, `peek-stream` | 1.4.2 (2020) / 1.1.3 (2018) | unmaintained | P3: replace with `node:zlib` (the code already sniffs gzip/deflate magic bytes in `minio-helper.ts:1181-1195`) and a ~40-line local `Transform`; that also retires the `duplexify: 4.1.3` override. |
| `mock-backend`: `express` 5.2.1, `@types/express` 5.0.6 | current | | keep; `@types/express` belongs in devDependencies; its lockfile has two `http://registry.npmjs.org` resolved URLs to regenerate before npm 12. |
| `server/package.json` `engines` | absent | | add one (or rely on the root); today `engine-strict` applies only to the frontend root. |

**The most important server finding is not a version at all.** `server/package.json` overrides `router: 1.3.7` (from #12045, Sep 2025, the Express 4 → 5 bump) and `path-to-regexp: 0.1.13` (pinned at 0.1.12 in 7a3ffeb5 on 2025-10-01 and bumped in #13626). They force Express 5.2.1 to load the Express-4-era router instead of its declared `router ^2.2.0` / `path-to-regexp` 8 (verified via `require.resolve`). So the tree has never run real Express 5 route matching, and **13** wildcard route strings in `app.ts` (lines 195, 204, 235, 244, 357, 369-372, 381, 394, 408, 423, including the `${basePath}`-prefixed duplicates: `/artifacts/*`, `/artifacts/:source/:bucket/*`, `/${prefix}/runs/*reportMetrics`, `/_proxy/*`, `/${prefix}/*`) use 0.1.x syntax that path-to-regexp 8 rejects at startup ("Missing parameter name"). `handlers/artifacts.ts:918` (and `integration-tests/artifact-proxy.test.ts:322`) read `req.params[0]`, which under path-to-regexp 8 becomes `req.params.splat`, an **array of segments**, so the artifact key must be rebuilt with `.join('/')`. The root `frontend/package.json` also carries a stale `express: { path-to-regexp: 0.1.13 }` override (the root lockfile has no express); remove it with the server overrides. The `mock-backend` has no overrides and runs the real Express 5 stack. Removing the overrides means rewriting the 13 routes (`/*splat`, and a regex or explicit `endsWith(':reportMetrics')` for the `runs/*reportMetrics` case, whose semantics differ), changing the `req.params[0]` reads, and re-running the browser integration tests. Do this **first** among the server changes so failures are attributable. Test coverage exists for the gate: the root `test:ci` runs the server's Vitest suite including `integration-tests/*`, and `e2e-test-frontend.yml` already runs an in-cluster smoke test against the live server (`/k8s/pod/logs`, Tensorboard Viewer create/get/delete, i.e. the k8s client paths), but that step is `continue-on-error: true`; make it blocking for the `@kubernetes/client-node` 2.0 PR. Also stale in the overrides block: `cross-spawn`, `json-schema`, `date-and-time` (no longer in the tree); `xmlbuilder: 15.1.1` (xml2js asks `~11`) and `duplexify: 4.1.3` (peek-stream asks `^3`) are untested cross-major pins.

**Order**: dead-weight removal + `crypto-js` → `node:crypto` (S) → un-fork Express 5 (M, route rewrite + integration tests) → `@types/node` 24 → supertest 7 then delete `form-data` and its overrides → `@kubernetes/client-node` 2.0 (in-cluster smoke: pod logs, Tensorboard Viewer CRs, secret/configmap reads, proxy/CA scenarios) → `google-auth-library` 11 (real-bucket check) → `tar-stream` 3 (backpressure rewrite, multi-GB download test) → AWS SDK / js-yaml / prettier minors → Vitest 5 and TypeScript with the root. Optional P3: `gunzip-maybe`/`peek-stream` replacement.

### 5.9 Runtime, CI and integration-test tooling

Effort / risk / priority: S / low / **P1** (unblocks jsdom 30 and npm 12; closes three skipped Node security releases). Verification: done (registry 96 atomic claims / 38 of 42 entries confirmed; codebase 45 / 38; corrections folded in).

**Node**: stay on Node 24, bump the patch. The pin `24.14.0` (2026-02-24) in `frontend/.nvmrc`, `package.json` `engines` (with `engine-strict=true`) and the lockfile root has skipped three Node 24 security releases (24.14.1 on 2026-03-24, 24.17.0 on 2026-06-18, 24.18.1 on 2026-07-29, each with High CVEs). Newest Node 24 LTS is **24.21.0** (2026-09-07). Node 24 is Active LTS until 2026-10-20, then Maintenance LTS with security fixes until 2028-04-30; Node 26 (Current since 2026-05-05) becomes Active LTS on 2026-10-28 and nothing in the toolchain needs it (all packages with `engines.node` across the five lockfiles accept both 24.21.0 and 26.9.0; no removed Node 26 APIs are used). Re-evaluate Node 26 in Q1/Q2 2027. Every build, image and release consumer reads `frontend/.nvmrc` (19 references), so the bump is a three-file change: `.nvmrc`, `engines.node`, and the lockfile root. **Order matters**: with `engine-strict=true`, every npm command in `frontend/` fails with `EBADENGINE` once `engines` says 24.21.0 while the shell still runs 24.14.0 (reproduced), so install the new Node first, then edit `engines`, then refresh the lockfile with `npm install --package-lock-only --ignore-scripts` **using the pinned npm** (the image-bundled npm 11.9.0 serialises the lockfile differently and drops 36 `libc` entries). Nothing guards this: `check-lockfile-drift.mjs` only runs its sync check when a dependency field changes (`engines`/`packageManager` are not in `DEP_FIELDS`), and `npm ci` compares only inventory versions, so add `engines` and `packageManager` to `DEP_FIELDS` or a one-line `.nvmrc == engines == lock-root` check. One consumer is outside the `.nvmrc` chain: the e2e `frontend-integration-test` job runs `test/server-integration-test` on the runner's default Node (no `setup-node` step). `node:24.21.0-slim` and `-alpine` images exist. Policy question: keep the exact pin (deterministic, but it sits unpatched between manual bumps; Dependabot does not update `.nvmrc`/`engines`/`packageManager`) or move to `^24.15.0` and let `.nvmrc` carry the exact version.

**npm**: bump `packageManager` to `npm@11.19.1` in the same PR. Node 24.20+ bundles npm 11.19.0, so keeping 11.17.0 would make the Dockerfile (lines 19 and 35), `frontend.yml`, `CONTRIBUTING.md:28` and `README.md:77` **downgrade** npm. npm 12.0.2 is not adoptable yet: its engines (`^22.22.2 || ^24.15.0 || >=26`) exclude 24.14.0, and it blocks dependency install scripts by default (`esbuild`, `unrs-resolver`, the nested `protobufjs` under `proto3-json-serializer` in the frontend lock; `edgedriver`/`geckodriver`/`esbuild` in the integration-test lock, none actually needed). After the Node bump: `npm approve-scripts` for esbuild/unrs-resolver (protobufjs disappears with `proto3-json-serializer`), regenerate `mock-backend/package-lock.json` (two `http://` resolved URLs), then move to 12.

**`@types/node`**: `^24` in both `frontend` (currently ^20) and `frontend/server` (^22).

**GitHub Actions**: `actions/setup-node`, `setup-python`, `checkout`, `upload-artifact`, `docker/build-push-action` are all on their newest major (v7). One fix: `e2e-test-frontend.yml` pins Python **3.9**, EOL since 2025-10-31; `frontend.yml` already uses 3.13 (ten other workflows outside this scope also pin 3.9). Its only Python use is the `wait_for_pods` helper (`kubernetes==30.1.0`), which should run on 3.13.

**`test/frontend-integration-test` (WebdriverIO)**: `webdriverio`/`@wdio/*` 9.31.6 → 9.31.9 (no v10 exists; the "v10 wish list" only commits to Node ≥ 22). Keep `mocha ^10.8.2`: `@wdio/mocha-framework` 9.31.9 depends on `^10.8.2` (upstream tried mocha 11 in 9.31.7 and reverted in 9.31.9 on 2026-09-13); a direct bump would only install a second, unused copy. Keep the exact `@puppeteer/browsers 3.2.2` pin and its `$@puppeteer/browsers` override for the whole WebdriverIO 9.x line: `@wdio/utils` declares `^2.2.0`, whose `extract-zip` carries two unpatched High advisories (GHSA-jmr9-qjv8-65gv and GHSA-7pqw-9j4j-h8q3 / CVE-2026-19693); upstream's fix (webdriverio#15553, draft) targets v10 only. The `dependency-smoke.test.mjs` job guards the forced major. Bump the `serialize-javascript` override `7.0.5` → `7.1.1`. Change `run_test.sh` (and the README line that documents it) from `npm install` to `npm ci` so the container honours the lockfile. `selenium/standalone-chromium:152.0` is current (consider pinning the fully qualified tag).

**Visual tooling**: `playwright` 1.63 in `frontend` and `scripts/ui-smoke-test` (then `npx playwright install chromium`), `pixelmatch` 7.2 (ESM default export unchanged; recapture baselines), `ui-smoke-test` `engines` `>=18` understates its real floor (`sharp` needs ≥ 20.9).

**Governance gap**: nothing automates `.nvmrc`/`engines`/`packageManager` currency (Dependabot cannot bump `.nvmrc`; dependabot-core #14752). Assign an owner or add a scheduled check comparing `.nvmrc` with `nodejs.org/dist/index.json` for the 24.x line, targeting each Node 24 security release within a sprint.

### 5.10 The three areas that broke before: tests, plots, router

Verification: done (55 claims / 43 confirmed; all four status ratings judged justified; counts corrected below).

| Area | Status today | Evidence |
|---|---|---|
| Router | **healthy** | react-router 8.4.0 is latest; 0 `react-router-dom` or v5 leftovers (`withRouter`, `useHistory`, `<Switch>`, `<Redirect>`, `RouteComponentProps`: 0 matches); `Router.test.tsx` 20/20 including optional/encoded params and the legacy `/executions` redirects; 20 test files use `MemoryRouter`, four use the data router (`createMemoryRouter` + `RouterProvider`: `Router.test.tsx`, `URLParser.test.ts`, `NewRunSwitcher.test.tsx`, `ArtifactDetails.test.tsx`). Remaining debt: `RoutedPage` in `Router.tsx:339-464` is the last class in the routing layer and mutates `this.state.toolbarProps` in place before `setState`; the legacy execution-redirect routes can be retired after a deprecation window. |
| Unit suite | **watch** | Green (139 files / 2,181 tests in my run; 0 skipped/todo/only tests anywhere; ESLint 0/0). But it leans on: 304 manual `act()` calls in 36 test files and `flushPromisesInAct`/`invokeAndFlush` helpers in 28 files alongside 489 `waitFor` + 278 `findBy*` (two styles coexist); 58 `wrapper.instance()` calls in 4 files (`NewRun.test.tsx` 31, `PipelineDetails.test.tsx` 20, `CustomTable.test.tsx` 5, `VisualizationCreator.test.tsx` 2) plus `createRunListInstance()` in `RunList.test.tsx`, 27 files using `React.createRef<ClassComponent>`, class state read 10 times directly and 65 times through four `get*State()` helpers plus 16 `wrapper.state()` calls, 17 private-method calls through `as any` casts, 5 test subclasses of production classes; 45 `makeErrorResponseOnce` one-shot mocks in 15 files, a documented StrictMode flake source (#14306 / #14323: the first StrictMode mount consumes the mock); 31 `console.error/warn` spies in 22 files, 27 silencing; six jsdom shims in `vitest.setup.ts` (TZ pin + `Date.prototype.toLocale*` monkey-patch defaulting to `en-US` because `Utils.tsx:57,59` and `ArtifactList.tsx:239-240` format dates without a locale; `DOMMatrixReadOnly`; `MouseEvent.view`; `URL.createObjectURL`/`Worker` for ace-builds, reached through `Editor.tsx` and two direct side-effect importers; Map-backed `localStorage`); the `test` script sets `LC_ALL=en_US.UTF-8`, which is not installed in the CI container; the dead `environmentMatchGlobs` key (§5.3); tests and stories are excluded from ESLint, so hooks rules never run on them. |
| Plots | **watch** | Contained to one recharts consumer with a 4,325-line snapshot (§5.6); 8 of 11 viewer components carry 9,593 snapshot lines in total; `CompareV1.test.tsx.snap` is 8,126 lines. The responsive-container test reaches into class state (`ref.current.setState`). |
| React 19 readiness | **watch** | No legacy APIs, peer gate green, StrictMode on in dev and tests. But the class-component footprint is **66 non-test files / 17,490 lines / 69 declarations** once the 20 pages extending the abstract `Page` and the 7 viewers extending `Viewer` are counted (both are `React.Component` subclasses); 39 files / 8,605 lines extend `React.Component` directly. The `Page` base keeps the `_isMounted` / `setStateSafe` anti-pattern (120 references); lifecycles in use: `componentDidMount` 27, `componentWillUnmount` 11, `componentDidUpdate` 6, `getDerivedStateFromProps` 2 (`NewRunParameters.tsx`, `NewRunParametersV2.tsx`), `componentDidCatch` 2, `shouldComponentUpdate` 1. Converting any of `NewRun`, `RunDetails`, `PipelineDetails`, `CustomTable` to functions invalidates the instance-coupled tests wholesale. |

**Why your earlier attempts broke here and what has changed.** In the fork's 2025 state, the suite was Enzyme + react-test-renderer + snapshot-diff on Jest, plots were react-vis, and routing was react-router-dom 4: each of those was a hard React 17/18 blocker, and each upgrade attempt had to migrate the tests, the chart library and the router at the same time as React. Upstream removed all three blockers first (CRA → Vite + Vitest and Enzyme removal in #12754, react-vis → recharts in #12829, router 4 → 5 in #12754 and 5 → 8 in #14417) and only then bumped React. What remains is not blockers but **churn surfaces**: Emotion-hash and SVG snapshots, class-instance test coupling, and one-shot mocks under StrictMode.

**Recommendations (test hygiene, independent of any version)**
1. Replace the 45 `makeErrorResponseOnce` calls in mount-driven tests with `makeErrorResponse` (as #14323 did for three files) and add a grep guard against new uses.
2. Shrink the snapshot surface: one full-DOM snapshot per viewer, targeted assertions for the rest; replace the 28 Emotion-hash snapshots (§5.1) with role/text assertions during the MUI 9 PR.
3. Burn down `instance()`/`createRef`/`ref.current.state` coupling starting with `NewRun`, `PipelineDetails`, `RunDetails`, `CustomTable`, tracking the count.
4. Fix `environmentMatchGlobs` (§5.3); lint tests and stories (a test-specific ESLint override rather than a blanket ignore); make the locale explicit at the four call sites in `Utils.tsx:57,59` and `ArtifactList.tsx:239,240` so the `Date.prototype` monkey-patch and `LC_ALL` can go; replace blanket `console.error` silencing with `expectErrors()`-style assertions so new React warnings fail tests.
5. Convert `RoutedPage` to a function component with a reducer for toolbar/banner/dialog/snackbar state; standardise test routing on one helper (`createMemoryRouter` + `RouterProvider` or a small `renderWithRouter`).

### 5.11 TanStack Query (the one runtime dependency no cluster owned)

`@tanstack/react-query` 5.102.8 → 5.103.1 (2026-09-16); peer `react ^18 || ^19`; there is no v6 line. It is the only data layer: 27 files, 24 `useQuery`, 2 `useQueries`, 3 `useMutation`, 1 `useInfiniteQuery`, 23 `new QueryClient` sites in 11 files. The delta was checked by diffing the installed source against the 5.103.1 tag: one runtime guard removed in `react-query`'s `HydrationBoundary` (unused here) and 34 lines in `query-core` across five areas (partial-key matching, `isPlainObject`, `streamedQuery`, hydration, `refetchInterval` resolution), none of which touches a pattern this code uses (all key filters are shorter than or equal to the stored keys; no hydration; the six function-form `refetchInterval` sites are semantically unchanged). Lockfile-only bump: the range `^5.102.8` already admits it. `isLoading` vs `isPending` usage is v5-correct as written (conditionally enabled queries need `isLoading`). The production `QueryClient` runs on library defaults (`retry` 3 with backoff, `refetchOnWindowFocus` true, `gcTime` 5 min), which is a UX choice to document, not a bug. Optional: `@tanstack/eslint-plugin-query` (CJS entry works with the flat config; `flat/recommended` has 7 rules, one of which is a `warn` that fails `--max-warnings=0`, so triage first) and the devtools package (needs a `@tanstack/*` Dependabot group because it pins the exact react-query minor). Effort S, risk low, P3. Verification: done (59 claims, every version, date and peer range reproduced; corrections folded in).

### 5.12 Generated-code toolchains (OpenAPI Generator, ts-proto, protoc)

About 283 generated TypeScript files are checked in (276 OpenAPI client files plus 7 protobuf files) and neither generator is current or drift-checked end to end.

- **OpenAPI Generator**: `scripts/generate_openapi_typescript_fetch.js` pins the Docker image `openapitools/openapi-generator-cli:v7.19.0` (2026-01-19); latest is **v7.25.0** (2026-08-24). A template diff of `typescript-fetch` 7.19 → 7.25 shows no API-surface break for this code but a mechanical rewrite: per-model imports instead of barrel imports in the 16 API class files (7.22), a `<op>RequestOpts` method per operation (91 operations, 7.20), centralised date helpers in both shared `runtime.ts` files and 24 date-bearing models (7.25), and prototype-chain fixes that make `instanceof runtime.ResponseError` reliable under the es2015 target (about 42 regenerated files in all). One script fix is needed first: the v1 symbol normaliser in the generator script does not match the new `…V1RequestOpts` names. CI already regenerates and `git diff --exit-code`s the clients on every PR, so the bump is a regenerate-and-commit PR; note that the CI pathspec omits the two shared `openapi/runtime.ts` files (`src/generated/openapi`, `server/src/generated/openapi`), so extend it, and that the v7.19.0 pin is also hard-coded in `CONTRIBUTING.md:268`.
- **ts-proto / protobuf**: covered in §5.7 (1.x output committed, 2.12.x installed, undeclared `long`). Additional facts: ts-proto 2.x output drops `long` entirely (default `forceLong=number`) and imports `@bufbuild/protobuf/wire`; consumers use only `fromJSON()`, the `ParameterType` enum and interface types, so application impact is nil. ts-proto 2.x stamps the protoc version into every file (`annotateFilesWithVersion` defaults true), and **protoc is unpinned**: CI apt-installs Ubuntu 24.04's 3.21.12 (2022) while the backend pins 31.1. A diff-based drift check therefore needs a pinned protoc (reuse `.github/actions/protobuf`'s install, version shared with `backend/api/Dockerfile`) or `annotateFilesWithVersion=false`.
- **Plan**: PR A (protobuf): ts-proto `^2.12.4`, add `@bufbuild/protobuf ^2.15`, remove `protobufjs` and `proto3-json-serializer`, pin protoc in `frontend.yml`, regenerate, turn `scripts/check-spec-generation.sh` into a typecheck plus `diff -ru` against `src/generated`, run the `src/lib/v2` and `NewRunParametersV2` tests. PR B (OpenAPI): extend the normaliser and its test, bump the image to v7.25.0 (script and `CONTRIBUTING.md`), `npm run apis:all` (needs Docker), commit; no code uses `instanceof` on the runtime error classes, so that behaviour change needs no follow-up. Land both in a quiet window before MUI 9 and Vitest 5 so reviewers see generated-only diffs. Add ts-proto and the generator image to a "generator pins" note in `CONTRIBUTING.md` and to `check-lockfile-drift.mjs`'s toolchain list. Effort M, risk medium, P2. Neither regeneration ran in this environment (no Docker daemon, no protoc), so the diff sizes are template-derived estimates. Verification: done (registry 64 claims / 34 confirmed; codebase 54 / 40; the stale-output, typecheck-only check, unpinned protoc and normaliser-gap findings all reproduced, including by running the exported normaliser; count corrections folded in).

### 5.13 Coordination with upstream (read this before opening any PR)

The fork has zero divergence from upstream, so every change in this plan should be an upstream PR first (§7). Upstream is already active in most of these areas:

| Upstream item | State (2026-09-19) | Effect on this plan |
|---|---|---|
| Issue **#14142** "migrate MUI v5 to v9" | open since 2026-08-24; milestone KFP 3.0 (due 2026-11-06); assignees jeffspahr, ayushagarwalhere; plan in the description: "one major at a time, using the official codemods"; its listed blockers (MLMD removal #11760/#13986, Dependabot `@mui` group #14090, dark-mode overlap #13709) have all closed. The issue has 4 comments (last 2026-09-07) whose bodies could not be fetched from this environment, so read them before assuming the plan is unchanged | Conflicts with §5.1's single hop only in hop count and codemod use. The evidence for one hop: zero targets in `src` for any v6 or v7 codemod, one hard break (`RemoveCircleOutline`), 7.3.11 as fallback. **Agree this in the issue before coding**, and correct the stale rationale on the Dependabot PRs (their comment cites `withStyles`/`makeStyles`, which the code no longer uses). |
| Dependabot PRs #14312 / #14313 (`@mui/*` 5.18 → 9.4) | open since 2026-09-07, CI red (peer conflict); per the maintainer's comment this is the third time Dependabot has recreated the pair as separate PRs despite the `@mui` group, held pending #14142 | superseded by the MUI 9 PR; once 9.4 is on master the next refresh should close them as "update no longer possible" (§5.16); check, and ask an OWNER to close them if they linger |
| PR **#13857** dark mode (issue #13709) | open since 2026-07-25, never ran CI (`needs-ok-to-test`), XXL; rewrites the `createTheme` block in `src/Css.tsx` into `getAppTheme(mode)` and replaces `ThemeProvider` in `src/index.tsx`; also carries 4 unrelated backend files | edits exactly the lines MUI 9 must change (§5.1 item 4). #14142 already says "verify dark mode post-migration": land MUI 9 first, then ask #13857 to rebase and re-verify `palette.mode` overrides |
| PR **#13950** (remove the `preserve-manual-memoization` suppression in `NewRunV2.tsx`) | open since 2026-08-01, never ran CI | drops React Compiler suppressions from 6 to 5 (§5.5); the compiler itself has no upstream issue, so enabling it is new scope |
| Dependabot #14378 `react` / #14379 `react-dom` 19.3.0 | open, checks pending, held | must merge together (§5.5); add the `react` group first |
| Dependabot #14314 `react-router`, #14315 `react-router-dom`, #14317 `runtypes` | open, CI red | **obsolete**: superseded by #14417 and #14416 already on master; #14314 and #14317 should close themselves on the next refresh, #14315 will not because its dependency is gone from every manifest (§5.16); ask an OWNER to close all three |
| Dependabot #14316 `recharts` 3.10.1 | open, CI red: 3 of 150 test files fail at runtime | adopt it: fix the ROCCurve snapshot/test fallout (§5.6) |
| Dependabot #14380 `ts-proto` 2.12.3 | open, stale (2.12.4 is out), no regenerated output | fold into §5.12 PR A |
| Dependabot #14381 `@types/js-yaml` 4.0.9 | open | superseded by deletion (§5.7) |
| Dependabot open-PR limit | 10, saturated by `/frontend` PRs, three obsolete | `/frontend/server` and integration-test bumps cannot surface until slots free up |
| Issue **#14028** post-MLMD hardening of runtime metadata views | open, unassigned | plans edits to `RunDetailsV2`, `CompareV2`, `DynamicFlow`, `ArtifactDetails`, all MUI-heavy pages whose snapshots MUI 9 regenerates; sequence or expect rebases |
| Release 2.18 (#14421) | branch cut 2026-09-13 at `f1f94058`, before MLMD removal; master is the KFP 3.0 track; three `frontend/server` fixes are cherry-pick-approved for 2.18 (#14362, #14431, #14434), and #14407/#14408 (artifact ownership via proxies) are open against master | target master only; schedule the Express un-fork and `@kubernetes/client-node` 2 after those land to avoid rebasing over them |
| Tailwind 4, Vitest 5, jsdom, TypeScript 7, Node patch bump, react-virtualized, dagre replacement, bundle size | **no upstream issue or PR** | uncontested; open one `area/frontend` tracking issue each (plus a `frontend/docs/*-upgrade-checklist.md` if more than one PR), following the React 18/19 pattern |

Review mechanics: `frontend/OWNERS` approvers are chensun, HumairAK, jeffspahr, manaswinidas (reviewers mprahl, droctothorpe); prow `/lgtm` + `/approve`; PRs from non-members wait at `needs-ok-to-test` until a member comments `/ok-to-test` (why #13857 and #13950 have sat for 7 to 8 weeks); DCO sign-off (`git commit -s`), conventional-commit titles (`chore(frontend): … Fixes #n`), and the repo's `docs/agents/conventions.md` forbids AI agents as commit co-authors, so fork PRs must not carry such trailers. Recent frontend dependency PRs (#14417, #14416, #14090) went from open to merged in 1 to 6 days. Effort M, risk medium, P1. Verification: done (104 claims; registry, repo and primary issue/PR facts reproduced; corrections folded in; the #14142 comment bodies were unreachable, see above).

### 5.14 Browser-support floor and build targets (the decision three other items wait on)

No document, issue or PR in the repository states which browsers the KFP UI supports. The build encodes a 2015-era floor in five places: `vite.config.mts:59` (`build.target: 'es2015'`, with no `cssTarget`, so Lightning CSS minifies for Chrome 49 / Safari 10-era browsers), `tsconfig.json:13`, `mock-backend/tsconfig.json:7`, `scripts/check-spec-generation.sh:14`, and the `browserslist` key in `package.json` (`supports es6-module`, which caniuse resolves to Chrome 61 / Edge 16 / Firefox 60 / Safari 10.1). That floor protects nobody: `index.html` is module-only and Vite ships no polyfills, so es2015 only lowers syntax (async/await, classes, optional chaining) into a 2.98 MB chunk, and every browser it nominally admits is already outside MUI 5.18's documented support (Chrome ≥ 90 / Safari ≥ 14) and TanStack Query 5's. It also produces the "Big integer literals are not available in the configured target environment" warning in the Storybook build.

Library floors that depend on the decision: Vite 8 default `baseline-widely-available` = Chrome/Edge 111, Firefox 114, Safari/iOS 16.4 (raised from Vite 7's Chrome/Edge 107, Firefox 104, Safari 16; the constant moves with each Vite major, per the Vite 8 migration guide); Tailwind 4 = Safari 16.4 / Chrome 111 / Firefox 128 (§5.2); MUI 9 = Chrome 117 / Edge 121 / Firefox 121 / Safari 17 (§5.1). Raising the floor to Vite's baseline drops browsers with about 3.1% of global usage per caniuse-lite (Chrome 61-110 1.5%, UC Android 0.7%, iOS Safari 10.3-16.3 0.6%). Only 0.18% of that (21 browser versions below Vite's Chrome 64 / Firefox 67 / Safari 11.1 / Edge 79 ESM minimum) cannot load the app today; the UC and QQ Android share (0.78%) does support dynamic `import()` and would be a deliberate drop, which web.dev's Baseline core-browser set (Chrome, Edge, Firefox, Safari) gives a principled reason for. The Node counterpart of the floor is already satisfied: Vite 8 needs Node ^20.19 or ≥ 22.12 and react-router 8 needs ≥ 22.22, and `.nvmrc` is 24.14.0.

**Recommendation** (S effort, low risk, P1 because Tailwind 4, MUI 9 and the bundle work are blocked on it): adopt a one-paragraph policy, approved by a `frontend/OWNERS` member and recorded in `frontend/README.md`: evergreen browsers; floor = Vite's `baseline-widely-available` constant for the Vite major in use; library floors above it (MUI 9, Tailwind 4) are the *tested* set, not a reason to raise `build.target`; no legacy chunks or polyfills; re-evaluated per Vite major. Then, as its own PR before Tailwind 4 and MUI 9: `build.target: 'baseline-widely-available'` (leave `cssTarget` to inherit); both tsconfigs to `target: es2022`, `lib: [es2022, dom, dom.iterable]` with `useDefineForClassFields: false` pinned (verified: `tsc --noEmit` is clean with and without the pin; the pin keeps emitted class-field code byte-identical for the 12 class files that declare fields without an initializer, found by a TypeScript AST walk rather than a grep, since `Tensorboard.tsx:90` declares `timerID` with no access modifier; 9 React class files initialize `state` as a field; none of them reads a field before the constructor assigns it, so the pin can be removed in a follow-up); the same target in `check-spec-generation.sh`; the `browserslist` key replaced by the explicit baseline list only if Tailwind 4 has not yet deleted it. Capture `visual:baseline` and the build sizes before, record the after numbers in the PR (expected JS 20 to 50 kB smaller, built CSS loses obsolete vendor prefixes, Storybook warning gone). Measure after Phase 0.6's `vite` 8.3 bump, not before: 8.0.16 pins rolldown 1.0.3 exactly and 8.3 pins a newer one, so sizes across the bump are not comparable. If maintainers do require pre-2023 browsers, `@vitejs/plugin-legacy` is the only supported route (its `terser ^5.16` peer collides with the root `overrides.terser: "5.14.2"`, which Phase 0.2 deletes anyway), and Tailwind 4 and MUI 9 cannot serve those browsers regardless.

Verification: 52 claim groups re-derived (registry, Vite source and docs, browserslist/caniuse-lite arithmetic, a TypeScript AST walk of the class fields, `tsc --noEmit` probes at es2022). The five es2015 sites, the 3.07% figure and the clean typecheck reproduced; three numbers were corrected above (unservable share 0.18% not about 1%, audit set 12 files not 11, `state` initializers 9 files) and the Vite 7 → 8 baseline change was added.

### 5.15 Bundle size, code splitting and shipped sourcemaps

Measured on the baseline build (Vite 8.0.16, `build.target: 'es2015'`): **one JavaScript chunk of 2,978.84 kB (832.36 kB gzip)** plus a **12,748 kB sourcemap** (`vite.config.mts:62` `sourcemap: true`) that the `Dockerfile` copies into the runtime image (line 39) and the Express server serves. There are **0** `React.lazy` / dynamic `import()` sites outside generated and test code, so nothing is code-split. No `chunkSizeWarningLimit`, no size budget script and no CI gate exist; the React 18/19 checklist compared bundle size by hand (0% delta against a 5% budget) but the practice was never automated. Upstream's checklist recorded 4,499 kB / 1,004 kB gzip in March 2026, so the number has already moved by a third without anyone gating it.

Attribution of the minified chunk by source (from the sourcemap of this build):

| Source | Minified | Share | Note |
|---|---|---|---|
| `ace-builds` | 501 kB | 17.2% | code editor; imported eagerly by 4 files (§5.7); the obvious first lazy-load target |
| `src` (application code) | 436 kB | 15.0% | |
| `recharts` | 211 kB | 7.3% | one consumer, `ROCCurve.tsx` (§5.6); second lazy-load target |
| `@mui/material` | 194 kB | 6.7% | will change with MUI 9 (§5.1) |
| `react-dom` | 175 kB | 6.0% | |
| `react-virtualized` | 141 kB | 4.9% | one consumer, `LogViewer.tsx`; drops to a fraction with `react-window` (§5.7) |
| `src/generated` (protobuf) | 111 kB | 3.8% | changes with the ts-proto 2.x regeneration (§5.12) |
| `lodash` | 108 kB | 3.7% | whole-library CJS copy plus 240 per-method modules pulled by `dagre` (§5.7); shrinks when dagre is replaced |
| `src/apis*` (OpenAPI clients) | 94 kB | 3.2% | changes with the generator bump (§5.12) |
| `@xyflow/react` + `@xyflow/system` | 132 kB | 4.5% | DAG canvas |
| `markdown-to-jsx`, `js-yaml`, `@tanstack/query-core`, `react-router`, `victory-vendor` | 38 to 76 kB each | 1.3% to 2.6% | |

**Why it matters for this plan**: MUI 9, Tailwind 4, recharts 3.10, `react-window`, `@dagrejs/dagre`, the ts-proto regeneration and the browser-floor change (§5.14) each move this number, and today there is no baseline to accept or reject them against. Vite 8 replaced `rollupOptions` with `build.rolldownOptions.output.codeSplitting` / `advancedChunks` for manual chunking.

**Recommendations** (none is version-driven; all are cheap):
1. Record the baseline (`vite build` sizes and gzip) in the tracking issue and add a size check to `test:ci` (a ~30-line script comparing `build/static/*.js` totals against a committed budget, or `rollup-plugin-visualizer`'s JSON output), the way `coverage-baseline.mjs` already works for coverage.
2. Lazy-load the two heavy leaf features: the ace editor (`Editor.tsx`, `VisualizationCreator.tsx`, `DetailsTable.tsx`, `PipelineDetails.tsx` imports) and `ROCCurve.tsx`, via `React.lazy` + `Suspense` at their call sites; together they are a quarter of the chunk and neither is on the first screen.
3. Stop shipping the sourcemap: either `sourcemap: 'hidden'` (still generated for error tooling, not referenced from the bundle) or exclude `*.map` from the runtime image copy; 12.7 MB is 4x the application itself.
4. Raise `build.target` off `es2015` when the browser floor is decided (§5.14): Vite transpiles syntax only (no polyfills), so es2015 buys nothing for old browsers while inflating output.

Effort S, risk low, P2. Source: the baseline build in this environment and its sourcemap; the research agent for this gap was blocked by a model safeguard before writing its result, so these figures come from the completeness pass and my own measurement rather than a separately verified finding.

### 5.16 Dependabot: why the groups do not work, and what to change

`.github/dependabot.yml` covers npm for every manifest (`directories: "**/*"`, weekly, limit 10, labels including `do-not-merge/hold`) with two groups: `mui` (`patterns: ["@mui/*"]`) and a catch-all `npm-dependencies` (`group-by: dependency-name`, no patterns). The file's own comment says a dependency joins the first matching group. **That is not how Dependabot resolves it**: in dependabot-core's specificity scoring, a patterns-less `group-by: dependency-name` group scores 500 while the wildcard `@mui/*` scores 91, so every dependency lands in a per-dependency subgroup of the catch-all and the `mui` group is empty. The evidence is in the open PRs themselves: #14312's body says `dependency-group: npm-dependencies/@mui/icons-material` and #14378's says `npm-dependencies/react`. This is why peer-coupled families (react/react-dom, `@mui/*`, vitest/`@vitest/*`, `@wdio/*`, `@storybook/*`, `@testing-library/*`, `@typescript-eslint/*`) keep arriving as separate PRs that cannot install alone (§5.13), and why the maintainer saw the `@mui` pair recreated three times after adding the group.

Two more mechanics matter for this plan: security updates bypass groups and the open-PR limit and are per directory (the Vitest 4.1.11 bumps #14335/#14358 for GHSA-82fw-gwwq-j7x9 arrived that way, one per manifest), and Dependabot stops rebasing a PR it cannot merge after 30 days, so the parked `@mui` majors from 2026-09-07 go stale on 2026-10-07. Upstream is at 10 of 10 open npm version-update PRs, six of them deliberately parked (five majors and the recharts minor #14316, whose three failing test files need fixing in the PR itself), so nothing new can open. Do not rely on the stale ones closing themselves: in dependabot-core's group-refresh path, a PR whose dependency is still present but can no longer be updated (#14314 `react-router`, now 8.4.0; #14317 `runtypes`, now 7.0.5 via #14416) is closed as "update no longer possible" on the next refresh, but a PR whose dependency has been removed from every manifest (#14315 `react-router-dom`) no longer maps to any group, is only warned about, and is never closed. Ask an OWNER to close all three rather than wait for the refresh.

**Recommended config change** (one `chore(ci)` PR upstream; `dependabot_config_test.py` passes unchanged but should gain an assertion that every pattern group is also excluded from the catch-all):
- Pattern groups: `react` (`react`, `react-dom`, `@types/react`, `@types/react-dom`), `mui` (`@mui/*`, `@emotion/*`), `vitest` (`vitest`, `@vitest/*`), `vite` (`vite`, `@vitejs/*`), `typescript-eslint` (`typescript`, `@typescript-eslint/*`), `eslint` (`eslint`, `@eslint/*`, `eslint-plugin-*`), `storybook` (`storybook`, `@storybook/*`), `testing-library` (`@testing-library/*`), `tailwind` (`tailwindcss`, `@tailwindcss/*`), `tanstack` (`@tanstack/*`), `wdio` (`webdriverio`, `@wdio/*`).
- Give the catch-all `patterns: ["*"]` plus `exclude-patterns` listing every family above, so it stops outranking them (dependabot-core supports patterns on a `group-by` group; the GitHub docs are silent, so validate on the first run and fall back to `exclude-patterns` alone if the validator rejects it).
- Leave `versioning-strategy` at its default (`auto`, which resolves to `increase` for an application, as every existing PR shows) or set `increase` explicitly as documentation; never `lockfile-only` or `increase-if-necessary`, because `check-lockfile-drift.mjs` fails when toolchain versions move in `frontend/package-lock.json` without a `package.json` change.
- Expect two things from the first run: the `vite` group opens a PR at once (`vite` 8.3.0 and `@vitejs/plugin-react` 6.1.1 are both pending), which is wanted (Phase 0.6, 1.3); and the `vitest` group's 5.0.1 PR fails `npm ci` until `@types/node` leaves ^20, because Vitest 5 peers `@types/node ^22 || >=24` (Phase 0.1 must land first).
- Replace parked PRs with expiring `ignore` rules and delete each rule in the PR that does the migration: `typescript >= 6.1` (typescript-eslint peer, §5.4), `jsdom >= 30` (until the Node bump, §5.3), `mocha >= 11` (`@wdio/mocha-framework`, §5.9), `@puppeteer/browsers` majors (§5.9), `@mui/*` majors (until #14142 is agreed, §5.13), `tailwindcss` (all updates until §5.2 lands: a majors-only ignore would immediately open a 3.0.11 → 3.4.19 `v3-lts` PR, which is §5.2's stop-gap Option B and only wanted if the browser-floor decision stalls).
- Keep the file shape `dependabot_config_test.py` asserts: no `target-branch` key, the exact `labels:` block quoting and order, and the open-PR limit it checks.
- Then triage the ten open PRs (§5.13): close the three obsolete ones, merge #14381 only if the `@types/js-yaml` deletion is rejected, merge the react pair together or let the new group re-open them as one, close the `@mui` pair pointing at #14142.

**Node and npm are invisible to Dependabot** (`.nvmrc`, `engines`, `packageManager`, the `Dockerfile` `ARG`; dependabot-core #14752). Either a small scheduled workflow that reads `nodejs.org/dist/index.json`, filters on `major == 24` and an `lts` field that is not `false` (Node 26.9.0 is already the Current line and becomes LTS in October, so an unfiltered "newest" check would jump majors), and opens a held PR for that release and its bundled npm, or a documented cadence (within a week of each Node 24 security release). The fork itself has no Dependabot branches and should stay that way: consume upstream merges, do not run a second Dependabot.

Effort S to M, risk low, P1 (it unblocks the react and vitest lockstep bumps and frees the PR limit).

Verification: 55 claims re-derived, the scoring mechanism confirmed from dependabot-core source and from four PR bodies; corrections folded in above (`versioning-strategy` is not "required", which stale group PRs auto-close and which do not, five not six parked majors, the Vitest 5 `@types/node` peer, the Tailwind `v3-lts` side effect, the Node 26 filter).

### 5.17 Security advisories in the lockfiles

`npm audit` on 2026-09-19 (read-only): `frontend` 0 of 790 packages, `mock-backend` 0, `test/frontend-integration-test` 0, `scripts/ui-smoke-test` 0, **`frontend/server` 7** (1 high, 6 moderate). OSV-Scanner (`osv-scanner.yml`, recursive, SARIF, non-blocking) and CodeQL run upstream, so these are visible there but unresolved (#14306 deferred them).

| Advisory | Path | Severity | Reachable from server code? | Fix |
|---|---|---|---|---|
| GHSA-2883-xcg3-v3hh `js-yaml` 4.3.1 | nested under `@kubernetes/client-node` 1.4.0 (`^4.1.0`) | **high** | no (only parses the kubeconfig) | lockfile refresh to 4.3.2 now (`npm update js-yaml` in `server/`, no `--force`); disappears entirely with `@kubernetes/client-node` 2 (js-yaml 5) |
| GHSA-w5hq-g745-h8pq `uuid` 9.0.1 | `google-auth-library` 9 → `gaxios` 6.7.1 | moderate | no (`gaxios` only calls `uuid.v4()`) | no in-range fix; leaves with `google-auth-library` 11 (gaxios 7 has no uuid), already in §5.8 |
| GHSA-528h-pc64-c93x `stream-json` 1.9.1 | `minio` 8.0.7 | moderate | no (used only by `listenBucketNotification`, never called) | **no in-range fix**: fixed in 3.5.0, which is ESM-only and breaks `import 'minio'` at startup if forced via `overrides` (verified) |
| GHSA-vcc3-ghjq-m6fr `decode-uri-component` 0.2.2 | `minio` 8.0.7 → `query-string` 7.1.3 | moderate | no (`minio` never calls `query-string.parse`) | **no in-range fix**: 0.5.0 is ESM-only; forcing it makes `parse()` throw; `query-string` 9 is default-export-only and would break five minio methods |

`minio` 8.0.7 is the latest release and its unreleased master still pins both transitives (minio-js #1494, #1497 open), so an upstream fix implies a minio major. `npm audit fix --force` offers only a downgrade to minio 7.1.3, which keeps `query-string` 7. `frontend/server` is an ESM project (`"type": "module"`, `tsc` then `node dist/server.js`), so minio's `dist/esm` build is the one that runs; the CJS build has the same exposure. Two process facts sharpen the recommendation: the deferral of these two CVEs exists only in the description of PR #14306 (no `*.md`, `*.yml` or `*.toml` in the repository mentions either advisory), and Dependabot version updates follow direct dependencies only, so #14284 bumped the server's direct `js-yaml` to ^5.4.1 while leaving the nested 4.3.1 untouched; the 4.3.2 refresh arrives only from a Dependabot *security* update or the manual `npm update`. One thing for the server cluster to expect: `@kubernetes/client-node` 2.0.0 moves its `@types/node` dependency to ^26 while the server pins ^22 (Phase 0.1 takes it to ^24), so expect a types-only duplicate until that is reconciled.

**Recommendation** (S to M, low risk, P2 after the js-yaml refresh, which is P1 and trivial): (1) refresh the nested `js-yaml` to 4.3.2 and the direct one to 5.4.2 in `server/` now; (2) land `google-auth-library` 11 and `@kubernetes/client-node` 2 per §5.8, which clears the high and the uuid findings; (3) record the two minio transitives as accepted risk in a `frontend/server/osv-scanner.toml` (`IgnoredVulns` with the reachability reason and an expiry) so OSV stays informative; (4) add `npm audit --omit=dev --audit-level=high` for `frontend` and `frontend/server` as a blocking step in `frontend.yml` once (1) lands; (5) track minio-js and, if no release adopts `stream-json >= 3.5` / `query-string >= 9` by the 2.19 planning point, scope replacing `minio` with `@aws-sdk/client-s3` (the server already depends on `@aws-sdk/credential-providers`; ~12 files and the minio-helper integration tests, effort L).

Verification: 70 claims re-derived (registry, the four advisory pages and OSV, `npm audit` on all five lockfiles, the minio/query-string/stream-json internals, a Node 24 `require(esm)` probe); four cosmetic corrections (advisory date labels, range wording, a Node-floor attribution) were folded in above, and the reachability, override-breakage and sequencing conclusions held.

## 6. Recommended plan

Principles, all inherited from upstream's own playbook (§1): one dependency cluster per PR; baselines (`visual:baseline`, `coverage:baseline`, green suite, and a bundle-size number) before each; `npm run test:ci && npm run build` before merge; never `vitest -u` blind; keep each PR small enough to bisect; conventional-commit titles, DCO sign-off, no AI co-author trailers (`docs/agents/conventions.md`). **Open every PR upstream first** and fast-forward the fork afterwards (§5.13, §7): the fork has zero divergence today, every item below is generic to kubeflow/pipelines, and carrying them as fork-only patches recreates the drift you just paid to eliminate. Rough total: 6 to 9 engineer-weeks across the phases, most of it in Phases 2, 4 and 5; Phases 0 and 1 are the ones to start this month.

### Phase 0: prerequisites and hygiene (about a week, low risk, unblocks everything)

| # | Change | Why first | Section |
|---|---|---|---|
| 0.0 | **Upstream coordination**: comment on #14142 with the single-hop evidence and the 7.3.11 fallback and get the assignees' answer; ask an OWNER to close the obsolete Dependabot PRs (#14314, #14315, #14317) and to decide #14381; open one `area/frontend` tracking issue each for the uncontested areas (Tailwind 4, Vitest 5 + jsdom 30, Node patch policy, react-virtualized, dagre, bundle size, Express un-fork, k8s client 2) | every later PR lands upstream first; the MUI hop count must be agreed before coding | §5.13 |
| 0.1 | Node `24.14.0` → `24.21.0` (`.nvmrc`, `engines`, lockfile root), `packageManager` → `npm@11.19.1`, `@types/node` `^24` in frontend and server | three skipped security releases; jsdom 30 and npm 12 both require ≥ 24.15; avoids the npm downgrade in Docker | §5.9 |
| 0.2 | Delete the dead client packages (`fs`, `url`, `http-proxy-middleware`, `@types/express`, `react-textarea-autosize`, four `lodash.*`, `@types/lodash.groupby`, `@types/pako`, `@types/js-yaml`, `proto3-json-serializer`, the dead `src/__mocks__/typestyle.js`) and the CRA-era root `overrides` that resolve to nothing in the lockfile (`ejs`, `json-schema`, `ansi-html`, `body-parser`, `tmpl`, `react-dev-utils`, `nth-check`, `terser`, `express.path-to-regexp`); move `ts-proto` to devDependencies; `@types/lodash` `^4.17` | zero source change; removes a deprecated stub, a placeholder package and an npm-12 install-script exposure | §5.7 |
| 0.3 | Server dead weight (`axios`, `lodash`, `@types/tar`, `@types/crypto-js`, stale overrides) and `crypto-js` → `node:crypto` with a hash-pinning test; refresh the nested `js-yaml` under `@kubernetes/client-node` to 4.3.2 (`npm update js-yaml`, no `--force`) | one-line change; removes a deprecated library; clears the only high `npm audit` finding | §5.8, §5.17 |
| 0.4 | Fix `environmentMatchGlobs`; Storybook imports from `@storybook/react-vite` and drop the three `tsconfig` `paths` hacks; add `.vitest/` to `.gitignore` | defects that exist today | §5.3 |
| 0.5 | Lint stack: `@typescript-eslint` 8.70, `eslint` 10.11, `globals` 17, promote the two downgraded React Compiler rules to `error`; `prettier` 3.9.8 in both manifests with a separate reformat commit; `semver`, `yaml`; delete the dead `check:react-peers:18` script | low risk, lint is 0/0 | §5.4 |
| 0.6 | Patch/minor sweep: React 19.3 lockstep (four packages), `@tanstack/react-query` 5.103, `recharts` 3.10.1 (own PR, snapshot review), `vite` 8.3, `playwright` 1.63, `js-yaml`, `pako`, `markdown-to-jsx`, `react-dropzone`, `@aws-sdk`; relax the `react-redux` override; add `react-is` | small, mostly lockfile | §5.5, §5.6 |
| 0.7 | `e2e-test-frontend.yml` Python 3.9 → 3.13; `run_test.sh` `npm ci`; wdio 9.31.9; `serialize-javascript` override 7.1.1 | CI currency | §5.9 |
| 0.8 | Dependabot config PR: pattern groups for `react`, `mui` (+ `@emotion/*`), `vitest`, `vite`, `typescript-eslint`, `eslint`, `storybook`, `testing-library`, `tailwind`, `tanstack`, `wdio`; give the catch-all `patterns: ["*"]` plus `exclude-patterns` for those families so it stops outranking them; `versioning-strategy: increase`; expiring `ignore` rules for `typescript >= 6.1` (TS 7), `jsdom >= 30`, `mocha >= 11`, `@puppeteer/browsers` majors, `@mui/*` majors, all of `tailwindcss` until Phase 3; extend `dependabot_config_test.py` | the `mui` group has been empty since it was added (§5.16); unblocks #14378/#14379; stops the un-installable TS 7 PR; frees the 10-PR limit | §5.16, §5.13 |
| 0.9 | **Browser-floor policy PR**: one-paragraph policy approved by an OWNER; `build.target: 'baseline-widely-available'`; tsconfigs to es2022 with `useDefineForClassFields: false` pinned; same target in `check-spec-generation.sh`; record before/after build sizes as the bundle baseline (after 0.6's `vite` bump, which changes the bundler) | Tailwind 4, MUI 9 and the bundle work are all blocked on this decision | §5.14, §5.15 |
| 0.10 | Generated-code PR A (protobuf): ts-proto 2.12.4, declare `@bufbuild/protobuf`, drop `protobufjs` (and the undeclared `long`), pin protoc, regenerate `src/generated`, make `check-spec-generation.sh` diff against the committed output. Then PR B (OpenAPI Generator v7.19 → v7.25, normaliser fix, `npm run apis:all`) | removes the silent runtime-library drift; generated-only diffs are easiest to review before the big PRs | §5.12 |

### Phase 1: test toolchain (about a week)

| # | Change | Section |
|---|---|---|
| 1.1 | Vitest 5 + `@vitest/coverage-v8` 5 in frontend and server together (needs 0.1's `@types/node` 24: Vitest 5 drops `@types/node` 20); decide `clearMocks`; confirm `coverage-baseline.mjs` still reads the summary | §5.3 |
| 1.2 | jsdom 30 (needs 0.1) with a deliberate snapshot review; keep the shims | §5.3 |
| 1.3 | `@vitejs/plugin-react` 6, `rollup-plugin-visualizer` 7, `pixelmatch` 7 (recapture visual baseline) | §5.3 |
| 1.4 | Extend `check-lockfile-drift.mjs` toolchain list | §5.3 |

### Phase 2: MUI 9 (one to two weeks, the big one)

Only after #14142's assignees agree the hop count (§5.13); land MUI 9 before the dark-mode PR #13857 rebases onto it. Single hop 5.18 → 9.4 per §5.1: baselines, bump, icon rename (hard break), codemods, 17 manual `<Input>` sites, theme variants, two test assertions, `colors` subpath, typecheck, suite without `-u`, snapshot regeneration only after `visual:diff`. Consider the `renderWithTheme` helper and snapshot-to-assertion conversion in the same PR series.

### Phase 3: Tailwind 4 (two to three days)

Option A per §5.2, its own PR after MUI 9 and after the browser-floor PR (0.9): `@tailwindcss/vite`, `@theme` port pruned to the 10 live tokens, the four visual fixes (shrink/grow rename, ring, SubDAG border colour, button cursor), delete the prebuild scaffolding, drop `postcss`/`autoprefixer`/`browserslist`.

### Phase 4: replacements (one to two weeks, can interleave with 2 and 3)

| # | Change | Section |
|---|---|---|
| 4.1 | `dagre` → `@dagrejs/dagre` + `@dagrejs/graphlib` (spike `layout()` on a `json.read` graph first; explicit imports in `RunDetails.tsx`/`PipelineDetails.tsx`; separate input and laid-out node types) | §5.6 |
| 4.2 | `react-virtualized` → `react-window` 2 in `LogViewer.tsx`; delete the Vite alias; browser check for horizontal scroll and selection | §5.7 |
| 4.3 | ts-proto alignment: done in 0.10 (PR A); if that PR slips, the fallback is to pin ts-proto 1.x and declare `long` so the checked-in output and the plugin agree | §5.7, §5.12 |

### Phase 5: server majors (one to two weeks; needs a dev cluster for two of them)

Wait for the open `frontend/server` security work (#14362, #14431, #14434 cherry-picks; #14407, #14408 on master) to merge, then: un-fork Express 5 (13 route strings, the `req.params[0]` read in `handlers/artifacts.ts:918`, integration tests; make the e2e in-cluster smoke step blocking) → supertest 7 and delete `form-data` + overrides → `@kubernetes/client-node` 2 (in-cluster smoke) → `google-auth-library` 11 (real bucket) → `tar-stream` 3.2.1 (backpressure rewrite, large download test). Clears the high and the `uuid` audit findings; the two `minio` transitives (`stream-json`, `decode-uri-component`) have no in-range fix and are recorded as accepted risk in `frontend/server/osv-scanner.toml` with an expiry, then `npm audit --omit=dev --audit-level=high` becomes a blocking step in `frontend.yml` (§5.17). §5.8.

### Deferred, with triggers

| Item | Trigger |
|---|---|
| TypeScript 7 in frontend | `@typescript-eslint` publishes a peer range including 7.x (after TS 7.1, planned 2026-11-24). Optional server-only pilot earlier. §5.4 |
| npm 12 | after Phase 0.1; run `npm approve-scripts` for `esbuild`/`unrs-resolver`, regenerate `mock-backend` lockfile. §5.9 |
| Node 26 | Active LTS starts 2026-10-28; re-evaluate Q1/Q2 2027; expected to be the same three-file bump. §5.9 |
| Storybook 11 | stable release (only `11.0.0-alpha.1` today). §5.3 |
| React Compiler | after fixing the 6 inline hooks suppressions and deciding the plugin-react 6 wiring; annotation mode first. §5.5 |
| `@xyflow/react` 13 | when `@xyflow/system` 1.0 ships. §5.6 |
| happy-dom | spike branch only, after jsdom 30 lands. §5.3 |
| Tailwind removal (Option C), typestyle replacement, class-component conversion | separate modernisation track, not version-driven. §5.2, §5.7, §5.5 |
| `gunzip-maybe`/`peek-stream` → `node:zlib` | P3 cleanup. §5.8 |

### Nice to have, not version-related

- **Bundle**: one 2.98 MB JS chunk (832 kB gzip), a 12.7 MB sourcemap shipped in the image, no code splitting, no size gate. Phase 0.9 raises the target; §5.15 lists the four cheap wins (size check in `test:ci`, lazy-load the ace editor and the ROC curve, stop shipping the sourcemap, higher target).
- **Docs**: `frontend/README.md` says "Vite 7" (line 7) and "MUI v5" (line 8); `CONTRIBUTING.md` and `docs/agents/frontend.md` repeat the stack list. Update with each phase.

## 7. Guardrails to keep, and how to stay current

**Keep running** (all exist today): `npm run check:react-peers` (React 19 peer gate over the lockfile, empty allowlist; the `:18` variant already fails and should go); `scripts/check-lockfile-drift.mjs` (catches accidental toolchain drift in `frontend/package-lock.json`; it does **not** compare frontend with server and does not see `engines`/`packageManager`-only changes, so extend `DEP_FIELDS`); `typecheck:mock-backend`; the generated-client drift check in `frontend.yml`; `scripts/coverage-baseline.mjs` and `scripts/visual-compare.mjs` (manual, but they are the only checks that see theme overrides and chart rendering). Consider wiring `visual:diff` into CI against the mock backend once Tailwind 4 removes the prebuild dependency.

**Dependabot** already covers npm (`**/*`, weekly, limit 10, `do-not-merge/hold`), but its `mui` group is empty because the patterns-less catch-all outranks it (§5.16). Fix the grouping per Phase 0.8 (pattern groups for every peer-coupled family, an `exclude-patterns` catch-all, `versioning-strategy: increase`) and add expiring `ignore` rules for the majors this plan deliberately defers (`typescript >= 6.1`, `jsdom >= 30`, `mocha >= 11`, `@puppeteer/browsers`, `@mui/*`, `tailwindcss`); delete each rule in the PR that lands the migration. There are no ignore rules today, so Dependabot keeps proposing PRs that cannot install. Note that majors for the server packages in §5.8 never landed not because of config but because of the `do-not-merge/hold` process; someone has to own those PRs. Dependabot does not manage `.nvmrc`, `engines` or `packageManager`: assign an owner or a scheduled check for Node 24.x security releases.

**Fork strategy.** This fork carried no frontend changes of its own, and its master fell 1,062 commits behind in 14 months. Recommendation: treat `master` as a mirror (sync weekly with `git fetch upstream && git merge --ff-only upstream/master`, or GitHub's "Sync fork"), do all work on short-lived branches, and open every change in this plan as an upstream PR first, cherry-picking to the fork only while the upstream PR is in review. Upstream's frontend approvers merged 299 frontend commits in the period and already run the CI these changes need. Close the six stale attempt branches (§3).

**Security gate.** Once the nested `js-yaml` refresh lands, add `npm audit --omit=dev --audit-level=high` for `frontend` and `frontend/server` as a blocking step in `frontend.yml`, and keep OSV-Scanner informative with a `frontend/server/osv-scanner.toml` that records the two `minio` transitives as accepted risk with an expiry (§5.17).

**Repeat this audit** each quarter: `npm outdated`, `npm audit`, `npm view <pkg> dist-tags` for the majors, `check:react-peers`, and a read of the upstream `frontend/docs` for new checklists.

## Appendix A. `frontend/package.json` delta, fork point (`ecfe94eb`) → upstream (`791a19c8`)

**Removed (45)**: `@craco/craco`, `@google-cloud/storage`, `@material-ui/core`, `@material-ui/icons`, `@storybook/addon-actions`, `@storybook/addon-essentials`, `@storybook/node-logger`, `@storybook/preset-create-react-app`, `@storybook/react`, `@types/d3`, `@types/enzyme`, `@types/enzyme-adapter-react-16`, `@types/google-protobuf`, `@types/http-proxy-middleware`, `@types/jest`, `@types/markdown-to-jsx`, `@types/prettier`, `@types/react-router-dom`, `@types/react-test-renderer`, `brace`, `coveralls`, `d3`, `enzyme`, `enzyme-adapter-react-16`, `enzyme-to-json`, `google-protobuf`, `grpc-web`, `jest-environment-jsdom-sixteen`, `portable-fetch`, `react-flow-renderer`, `react-query`, `react-router-dom`, `react-router-test-context`, `react-scripts`, `react-svg-line-chart`, `react-test-renderer`, `react-vis`, `request`, `snapshot-diff`, `swagger-ts-client`, `ts-node`, `ts-node-dev`, `tsconfig-paths`, `tslint-config-prettier`, `webpack-bundle-analyzer`

**Added (30)**: `@emotion/react`, `@emotion/styled`, `@eslint/js`, `@mui/icons-material`, `@mui/material`, `@storybook/react-vite`, `@tanstack/react-query`, `@typescript-eslint/eslint-plugin`, `@typescript-eslint/parser`, `@vitejs/plugin-react`, `@vitest/coverage-v8`, `@xyflow/react`, `ace-builds`, `eslint`, `eslint-plugin-import-x`, `eslint-plugin-react-hooks`, `globals`, `jsdom`, `pixelmatch`, `playwright`, `pngjs`, `react-router`, `recharts`, `rollup-plugin-visualizer`, `semver`, `storybook`, `tsx`, `url`, `vite`, `vitest`

**Range changed (39)**:

| Package | Fork point | Now |
|---|---|---|
| `@storybook/addon-links` | ^6.3.6 | ^10.6.0 |
| `@testing-library/dom` | ^8.6.0 | ^10.4.1 |
| `@testing-library/jest-dom` | ^6.6.3 | ^7.0.1 |
| `@testing-library/react` | ^11.2.6 | ^16.3.3 |
| `@testing-library/user-event` | ^13.2.1 | ^14.5.2 |
| `@types/d3-dsv` | ^1.0.33 | ^3.0.7 |
| `@types/dagre` | ^0.7.40 | ^0.7.54 |
| `@types/express` | ^4.16.0 | ^5.0.6 |
| `@types/lodash.groupby` | ^4.6.6 | ^4.6.9 |
| `@types/node` | ^10.17.60 | ^20.19.28 |
| `@types/pako` | ^1.0.3 | ^3.0.0 |
| `@types/react` | ^16.9.22 | ^19 |
| `@types/react-dom` | ^16.9.5 | ^19 |
| `@types/react-virtualized` | ^9.18.7 | ^9.22.3 |
| `browserslist` | 4.16.5 | 4.28.8 |
| `d3-dsv` | ^1.0.10 | ^3.0.1 |
| `dagre` | ^0.8.2 | ^0.8.5 |
| `http-proxy-middleware` | ^0.19.0 | ^4.2.0 |
| `immer` | ^9.0.6 | ^11.1.18 |
| `js-yaml` | ^3.14.1 | ^5.4.1 |
| `lodash` | ^4.17.21 | ^4.18.1 |
| `markdown-to-jsx` | ^6.11.4 | ^9.10.2 |
| `pako` | ^2.0.4 | ^3.0.1 |
| `postcss` | ^8.4.5 | ^8.5.23 |
| `prettier` | 1.19.1 | ^3.8.1 |
| `proto3-json-serializer` | ^0.1.6 | ^4.0.2 |
| `protobufjs` | ~6.11.2 | ~8.8.0 |
| `re-resizable` | ^4.9.0 | ^6.11.2 |
| `react` | ^16.12.0 | ^19.2.7 |
| `react-ace` | ^7.0.2 | ^15.0.0 |
| `react-dom` | ^16.12.0 | ^19.2.7 |
| `react-dropzone` | ^5.1.0 | ^20.1.1 |
| `react-textarea-autosize` | ^8.3.3 | ^8.5.9 |
| `react-virtualized` | ^9.20.1 | ^9.22.6 |
| `runtypes` | ^6.3.0 | ^7.0.5 |
| `ts-proto` | ^1.95.0 | ^2.12.1 |
| `typescript` | ^3.8.3 | ^6.0.3 |
| `typestyle` | ^2.0.4 | ^2.4.0 |
| `yaml` | ^2.2.2 | ^2.8.3 |
