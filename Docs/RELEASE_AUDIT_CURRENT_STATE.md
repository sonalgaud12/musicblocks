# Release Infrastructure Audit — Current State

**Document version:** 1.0  
**Audit date:** 2026-05-25  
**Auditor:** Infrastructure Audit (Deliverable 1)  
**Repository:** [sugarlabs/musicblocks](https://github.com/sugarlabs/musicblocks)  
**Branch audited:** `master` (HEAD: `d7e16bc81`)

---

## Table of Contents

1. [Scope and Methodology](#1-scope-and-methodology)
2. [Repository Structure Overview](#2-repository-structure-overview)
3. [Workflow Audit — All 11 Workflows](#3-workflow-audit--all-11-workflows)
   - 3.1 [auto-rebase.yml](#31-auto-rebaseyml)
   - 3.2 [conflict-check.yml](#32-conflict-checkyml)
   - 3.3 [lighthouse-ci.yml](#33-lighthouse-ciyml)
   - 3.4 [linter.yml](#34-linteryml)
   - 3.5 [node.js.yml (Smoke Test)](#35-nodejsyml--smoke-test)
   - 3.6 [po-to-json-validation.yml](#36-po-to-json-validationyml)
   - 3.7 [pr-category-check.yml](#37-pr-category-checkyml)
   - 3.8 [pr-cypress-e2e.yml](#38-pr-cypress-e2eyml)
   - 3.9 [pr-jest-tests.yml](#39-pr-jest-testsyml)
   - 3.10 [security_scan.yml](#310-security_scanyml)
   - 3.11 [stale.yml](#311-staleyml)
4. [Build Infrastructure](#4-build-infrastructure)
   - 4.1 [Express Server (index.js)](#41-express-server-indexjs)
   - 4.2 [Gulp Pipeline (gulpfile.js / gulpfile.mjs)](#42-gulp-pipeline)
   - 4.3 [Service Worker (sw.js)](#43-service-worker-swjs)
   - 4.4 [Translation Pipeline (convert_po_to_json.py)](#44-translation-pipeline)
5. [End-to-End Deployment Flow](#5-end-to-end-deployment-flow)
6. [Workflow Dependency Graph](#6-workflow-dependency-graph)
7. [Baseline DORA Metrics](#7-baseline-dora-metrics)
8. [Secrets, Environment Variables, and Configuration](#8-secrets-environment-variables-and-configuration)
9. [Identified Risks and Consolidation Opportunities](#9-identified-risks-and-consolidation-opportunities)

---

## 1. Scope and Methodology

This document provides a ground-truth snapshot of all automation, build systems, and deployment processes as they exist in the `sugarlabs/musicblocks` repository on 2026-05-25. It is intended as the authoritative baseline for all subsequent automation design decisions.

**Sources examined:**
- All 11 files under `.github/workflows/`
- `index.js` (Express server)
- `gulpfile.js` and `gulpfile.mjs` (Gulp pipeline, both CJS and ESM variants)
- `sw.js` (Service Worker)
- `convert_po_to_json.py` (Translation pipeline)
- `package.json` (project manifest, scripts, and dependencies)
- `jest.config.js`, `cypress.config.js`, `lighthouserc.js`
- `CONTRIBUTING.md`, `README.md`, `Docs/PRODUCTION_BUILD_STRATEGY.md`
- Full `git log` of the `master` branch

**Methodology:** Direct source inspection of all workflow YAML files; `git log` analysis for commit frequency, revert counts, and release markers; static analysis of all environment variable references across workflows and source files.

---

## 2. Repository Structure Overview

```
musicblocks/
├── .github/
│   ├── ISSUE_TEMPLATE/          # 8 issue templates
│   ├── PULL_REQUEST_TEMPLATE/
│   │   └── generic.md           # Enforced PR checklist with CI-checked category boxes
│   ├── FUNDING.yml
│   └── workflows/               # 11 CI/CD workflow files (audited below)
├── js/                          # Application JavaScript (~209 files, ~6.42 MB)
├── css/                         # Stylesheets
├── lib/                         # Library files (~39 files, ~6.52 MB)
├── po/                          # Gettext translation sources (.po files)
├── locales/                     # Generated JSON translations (output of pipeline)
├── cypress/                     # Cypress E2E tests
├── Docs/                        # Architecture and strategy documentation
├── index.js                     # Express server entry point
├── gulpfile.js                  # Gulp pipeline (CommonJS)
├── gulpfile.mjs                 # Gulp pipeline (ESM)
├── sw.js                        # Progressive Web App service worker
├── convert_po_to_json.py        # Python translation conversion script
├── package.json                 # v3.4.1, Node ≥20 required
├── jest.config.js               # Jest unit test config
├── cypress.config.js            # Cypress E2E config
└── lighthouserc.js              # Lighthouse CI thresholds
```

**Package version:** `3.4.1`  
**Node engine requirement:** `^20.0.0`  
**Primary module system:** RequireJS/AMD (deeply integrated; bundling constrained)

---

## 3. Workflow Audit — All 11 Workflows

All workflows share these common traits unless noted otherwise:
- Runner: `ubuntu-latest`
- Timeout: `timeout-minutes: 30` (added uniformly in [#7322](https://github.com/sugarlabs/musicblocks/pull/7322))
- Checkout: `actions/checkout@v4`
- Node setup: `actions/setup-node@v5`

---

### 3.1 `auto-rebase.yml`

| Property | Value |
|----------|-------|
| **Name** | Auto Rebase PRs |
| **Trigger** | `push` to `master`; `workflow_dispatch` |
| **Runs on** | `ubuntu-latest` |
| **Job** | `rebase` |
| **Concurrency** | `group: auto-rebase`, `cancel-in-progress: true` |
| **Permissions** | `contents: write`, `pull-requests: write` |
| **Secrets used** | `secrets.GITHUB_TOKEN` |
| **External actions** | `actions/checkout@v4`, `actions/github-script@v7` |

**Purpose:** After every push to `master`, fetches all open non-draft PRs targeting `master`. Filters out PRs with labels `no-rebase`, `wip`, or `do-not-merge`, and PRs from forks that have not enabled "Allow edits from maintainers." For each eligible PR, attempts a `git rebase origin/master` and pushes back with `--force-with-lease`. Failures are skipped gracefully with a warning.

**Security note:** Uses `execFileSync('git', args)` (no shell interpolation) to prevent injection via branch names or committer fields. Preserves original committer identity on the rebase.

**Limitations:**
- Cannot rebase fork PRs that do not allow maintainer edits (logs a skip warning).
- If the fork's source repository was deleted, the PR is skipped.
- No downstream notification of rebase success/failure sent to PR authors.

---

### 3.2 `conflict-check.yml`

| Property | Value |
|----------|-------|
| **Name** | Label Conflicting PRs |
| **Trigger** | `workflow_run` (after "Auto Rebase PRs" completes); `pull_request_target` (synchronize) |
| **Runs on** | `ubuntu-latest` |
| **Job** | `check` |
| **Permissions** | `contents: read`, `pull-requests: write` |
| **Secrets used** | `secrets.GITHUB_TOKEN` (via `repoToken`) |
| **External actions** | `eps1lon/actions-label-merge-conflict@v3` |

**Purpose:** Runs after `auto-rebase.yml` completes (ensuring we only flag PRs that auto-rebase could not fix). Labels conflicting PRs with `needs-rebase` and posts a comment with rebase instructions. Clears the label when conflicts are resolved (`commentOnClean: ""`). Uses `retryAfter: 120s`, `retryMax: 5` to handle eventual consistency in the GitHub API.

**Design note:** The `workflow_run` trigger ensures conflict-check never runs concurrently with auto-rebase — only PRs with real conflicts get flagged.

---

### 3.3 `lighthouse-ci.yml`

| Property | Value |
|----------|-------|
| **Name** | Lighthouse CI |
| **Trigger** | `pull_request_target` (opened, synchronize, reopened); `push` to `master` |
| **Runs on** | `ubuntu-latest` |
| **Job** | `lighthouse` |
| **Strategy** | Matrix: `preset: [desktop, mobile]` |
| **Concurrency** | `group: ${{ github.workflow }}-${{ github.event.pull_request.number \|\| github.ref }}`, cancel-in-progress |
| **Permissions** | `pull-requests: write`, `contents: read`, `issues: write` |
| **Secrets used** | None (uses automatic `GITHUB_TOKEN`) |
| **External actions** | `actions/checkout@v4`, `actions/setup-node@v5`, `thollander/actions-comment-pull-request@v3`, `actions/upload-artifact@v4` |
| **Global tool** | `@lhci/cli@0.14.x` (installed via `npm install -g`) |

**Purpose:** Runs Lighthouse performance audits in parallel for desktop and mobile profiles. Parses scores from `.lighthouseci/manifest.json` and posts a formatted table comment on the PR (recreated on each run, tagged `lighthouse-report-{preset}`). Uploads raw Lighthouse reports as artifacts (30-day retention). All thresholds are **warnings only** (no build failure on score degradation).

**Score thresholds (from `lighthouserc.js`):**

| Metric | Desktop | Mobile |
|--------|---------|--------|
| Performance | ≥50 (warn) | ≥50 (warn) |
| Accessibility | ≥80 (warn) | ≥80 (warn) |
| Best Practices | ≥80 (warn) | ≥80 (warn) |
| SEO | ≥80 (warn) | ≥80 (warn) |
| FCP | ≤4000ms | ≤5000ms |
| LCP | ≤6000ms | ≤7500ms |
| CLS | ≤0.25 | ≤0.25 |
| TBT | ≤600ms | ≤1000ms |
| Speed Index | ≤5000ms | ≤6500ms |

**Lighthouse target:** `temporary-public-storage` (free, no API key required; reports expire).

**Limitation:** Uses `pull_request_target` trigger which runs in the base repo context — correctly isolated from fork code since no user code is executed during the Lighthouse audit of the static site.

---

### 3.4 `linter.yml`

| Property | Value |
|----------|-------|
| **Name** | ESLint & Prettier |
| **Trigger** | `pull_request` to `master` |
| **Runs on** | `ubuntu-latest` |
| **Job** | `lint` |
| **Concurrency** | `group: ${{ github.workflow }}-${{ ... }}`, cancel-in-progress |
| **Permissions** | Default (read) |
| **Secrets used** | None |
| **Node version** | `22.x` |

**Purpose:** On PRs, identifies changed JavaScript files (`.js`, `.mjs`) via `git diff` with `--diff-filter=ACMRT`, then runs `npx eslint` and `npx prettier --check` on those files only. If no JS files changed, both steps are skipped. Does **not** lint the entire codebase — incremental only.

**Configuration:**
- ESLint config: `eslint.config.mjs` (flat config, ESLint v9+)
- Prettier config: implicit (project root)
- `lint-staged` configured in `package.json` for pre-commit hooks via Husky

**Note:** Uses `pull_request` (not `pull_request_target`), so it cannot write comments but also cannot be exploited by fork code with elevated privileges.

---

### 3.5 `node.js.yml` — Smoke Test

| Property | Value |
|----------|-------|
| **Name** | Smoke Test |
| **Trigger** | `push` to `master`; `pull_request` to `master` |
| **Runs on** | `ubuntu-latest` |
| **Job** | `build` |
| **Strategy** | Matrix: `node-version: [20.x, 22.x]` |
| **Concurrency** | `group: ${{ github.workflow }}-${{ ... }}`, cancel-in-progress |
| **Permissions** | Default (read) |
| **Secrets used** | None |

**Purpose:** Verifies that `npm ci` succeeds and `npm run build --if-present` passes on both Node 20 and Node 22. Does **not** run tests (tests are handled by `pr-jest-tests.yml`). Runs on both PR and push-to-master, making it the primary gate for dependency integrity.

**Note:** The `npm run build` script is `--if-present`, and the `package.json` has no `build` script defined — so effectively only `npm ci` is validated. This workflow's primary value is confirming clean dependency installation across Node versions.

---

### 3.6 `po-to-json-validation.yml`

| Property | Value |
|----------|-------|
| **Name** | Convert PO to JSON and verify changes |
| **Trigger** | `push` (paths: `po/**/*.po`); `pull_request` (paths: `po/**/*.po`) |
| **Runs on** | `ubuntu-latest` |
| **Job** | `convert-and-verify` |
| **Permissions** | Default |
| **Secrets used** | None |
| **Runtime** | Python 3.x |

**Purpose:** Path-filtered workflow that activates only when `.po` translation files change. Identifies changed `.po` files via `git diff`, runs `convert_po_to_json.py` on each changed file, then verifies that the resulting `locales/*.json` files are already committed. If the generated JSON does not match what's in the working tree, the workflow fails with an actionable error message.

**Implication:** Contributors who edit `.po` files must also run the conversion script locally and commit the resulting JSON before pushing. There is no automated commit-back step.

**Special case:** Japanese translations require both `ja.po` and `ja-kana.po` to be present; they are merged into a single `ja.json` with `{ kanji: ..., kana: ... }` structure.

---

### 3.7 `pr-category-check.yml`

| Property | Value |
|----------|-------|
| **Name** | PR Category Check |
| **Trigger** | `pull_request_target` (opened, edited, synchronize, reopened) |
| **Runs on** | `ubuntu-latest` |
| **Jobs** | Three sequential steps in one job: category check → size labeling → area labeling |
| **Permissions** | `pull-requests: write`, `contents: read`, `issues: write` |
| **Secrets used** | Automatic `GITHUB_TOKEN` |
| **External actions** | `actions/github-script@v7` × 3 |

**Purpose:** Three-phase label management system:

1. **Category validation:** Fails CI if no category checkbox is checked in the PR body. Valid categories: `Bug Fix`, `Feature`, `Performance`, `Tests`, `Documentation`. Creates labels if they don't exist.

2. **Size labeling:** Computes `additions + deletions` and applies one of six size labels: `size/XS` (<10), `size/S` (10-49), `size/M` (50-249), `size/L` (250-499), `size/XL` (500-999), `size/XXL` (1000+). Removes stale size labels.

3. **Area labeling:** Inspects changed file paths and applies area labels: `area/javascript`, `area/css`, `area/plugins`, `area/docs`, `area/tests`, `area/i18n`, `area/assets`, `area/ci-cd`, `area/lib`, `area/core`.

**Commented-out feature:** Release notes enforcement (`## Release Notes` section checking) is present but commented out.

**Security note:** Uses `pull_request_target` safely because no fork code is checked out or executed — only the PR body and metadata from the event payload are read.

---

### 3.8 `pr-cypress-e2e.yml`

| Property | Value |
|----------|-------|
| **Name** | E2E Test |
| **Trigger** | `pull_request` (opened, synchronize); `push` to `master` |
| **Runs on** | `ubuntu-latest` |
| **Job** | `e2e` |
| **Concurrency** | `group: ${{ github.workflow }}-${{ ... }}`, cancel-in-progress |
| **Permissions** | Default |
| **Secrets used** | None |
| **Node version** | `22` |
| **Browser** | Chrome (headless) |

**Purpose:** Runs Cypress end-to-end tests using `cypress-io/github-action@v6`. Starts the Express server with `npm start` and waits up to 120 seconds for `http://127.0.0.1:3000` to be available. Video recording is disabled. On failure, uploads screenshots from `cypress/screenshots` as artifacts.

**Cypress configuration (`cypress.config.js`):**
- Viewport: 1400×1000
- `testIsolation: false`
- No custom plugins or node event listeners

**Note:** Uses `pull_request` trigger (not `pull_request_target`) — cannot write PR comments but is safer for fork PRs.

---

### 3.9 `pr-jest-tests.yml`

| Property | Value |
|----------|-------|
| **Name** | Run Jest Tests on PR |
| **Trigger** | `pull_request_target` (opened, synchronize) |
| **Runs on** | `ubuntu-latest` |
| **Job** | `test` |
| **Concurrency** | `group: ${{ github.workflow }}-${{ ... }}`, cancel-in-progress |
| **Permissions** | `pull-requests: write`, `contents: read`, `issues: write` |
| **Secrets used** | `secrets.GITHUB_TOKEN` |
| **External actions** | `actions/checkout@v4` × 2, `actions/setup-node@v5`, `thollander/actions-comment-pull-request@v3` |
| **Node version** | `22` |

**Purpose:** Runs `jest --coverage --json --outputFile=jest-results.json` on the PR branch. Then **also** checks out `master` into `./master-branch/` and runs Jest again to provide a coverage delta comparison. Posts a detailed PR comment with pass/fail status, coverage summary, and failed test names if any. Fails CI if tests fail.

**Coverage thresholds (from `jest.config.js`):**

| Metric | Threshold |
|--------|-----------|
| Statements | 34% |
| Branches | 29% |
| Functions | 41% |
| Lines | 34% |

**Coverage scope:** `js/**/*.js`, `planet/js/**/*.js` (excluding test files)
**Test environment:** `jsdom`

**Performance concern:** This workflow runs `npm ci` **twice** (once for the PR branch, once for master) and runs Jest **twice**. This is the most expensive workflow in the suite.

---

### 3.10 `security_scan.yml`

| Property | Value |
|----------|-------|
| **Name** | Security Scans |
| **Trigger** | `push` to `master`; any `pull_request`; `workflow_dispatch` |
| **Runs on** | `ubuntu-latest` |
| **Job** | `security-scans` |
| **Permissions** | Default |
| **Secrets used** | None |
| **Node version** | `22` |

**Purpose:** Runs `npm audit --omit=dev --audit-level=high`. Only production dependencies are audited; dev dependency vulnerabilities are ignored. Fails CI if any `high` or `critical` severity vulnerability is found in production dependencies.

**Scope:** This is the narrowest security scan possible — only npm audit, no SAST, no secret scanning, no dependency review beyond npm.

---

### 3.11 `stale.yml`

| Property | Value |
|----------|-------|
| **Name** | Close stale pull requests |
| **Trigger** | `schedule: cron: '30 19 * * *'` (daily at 19:30 UTC); `workflow_dispatch` |
| **Runs on** | `ubuntu-latest` |
| **Job** | `stale` |
| **Permissions** | Default |
| **Secrets used** | None |
| **External actions** | `actions/stale@v9` |

**Purpose:** Marks PRs stale after 60 days of inactivity. Closes stale PRs after a further 3 days (63 days total). Issues are explicitly **not** managed (`days-before-issue-stale: -1`). Posts a warning comment when marking stale; posts a close message when closing.

---

## 4. Build Infrastructure

### 4.1 Express Server (`index.js`)

The project includes a lightweight Express 5 server (`express@5.2.1`) that serves as the development and test server.

**Key behaviors:**
- **Environment detection:** `process.env.NODE_ENV !== "production"` controls caching headers and debug modes
- **`/env.js` endpoint:** Dynamically generates a JavaScript snippet that exposes `window.MB_ENV` and `window.MB_IS_DEV` to the browser — avoids bundling environment config
- **Compression:** `compression` middleware with `level: 9, threshold: 0` — compresses all responses including small ones
- **Security middleware:** Blocks requests to `package.json`, `index.js`, `Dockerfile`, all `.github/` paths, `node_modules/`, `cypress/`, `scripts/` with HTTP 403
- **Static file serving:** `express.static(__dirname)` serves the project root; dotfiles blocked with `dotfiles: "deny"`
- **Caching:** `maxAge: 0` in development, `"1h"` in production; ETags disabled in development
- **Binding:** `HOST` (default `127.0.0.1`), `PORT` (default `3000`)

**npm scripts:**
| Script | Command |
|--------|---------|
| `start` | `node index.js` |
| `dev` | `cross-env NODE_ENV=development nodemon index.js` |
| `prod` | `cross-env NODE_ENV=production node index.js` |
| `serve` | `http-server -a 127.0.0.1 -p 3000 --gzip --brotli` (static, no Express) |
| `serve:dev` | `http-server -a 127.0.0.1 -p 3000 -c-1` (no cache) |

---

### 4.2 Gulp Pipeline

Two Gulp configuration files coexist, representing a migration in progress:

#### `gulpfile.js` (CommonJS — legacy, currently active for `require()`-based tooling)

**Tasks:**
| Task | Input | Output | Operation |
|------|-------|--------|-----------|
| `sassTask` | `css/*.sass` | `dist/css/` | SASS → PostCSS (autoprefixer + cssnano) + sourcemaps |
| `cssTask` | `css/*.css` | `dist/css/` | CleanCSS (IE8 compat) |
| `jsTask` | `js/**/*.js` | `dist/` → `app.min.js` | concat → Babel (@babel/env) → uglify |
| `vendorTask` | node_modules (6 libs) | `dist/` → `vendor.min.js` | concat only |
| `cacheBustTask` | `index.html` | `.` | replaces `cb=\d+` with current timestamp |
| `prettify` | `js/**/*.js` | `dist/js/` | Prettier formatting |
| `validate` | `js/**/*.js` | — | Prettier format check |
| `watchTask` | `js/**/*.js`, `css/*` | — | Watch + re-run JS/CSS/SASS tasks |

**Default export:** `series(parallel(vendorTask, jsTask, cssTask, sassTask), prettify, cacheBustTask, validate, watchTask)`

**Vendor libraries bundled:**
- `jquery/dist/jquery.min.js`
- `jquery-ui-dist/jquery-ui.min.js`
- `materialize-css/dist/js/materialize.min.js`
- `abcjs/dist/abcjs-basic-min.js`
- `howler/howler.min.js`
- `tone/build/Tone.js`

#### `gulpfile.mjs` (ESM — newer variant)

**Tasks:**
| Task | Input | Output | Operation |
|------|-------|--------|-----------|
| `jsTask` | `js/**/*.js` | `dist/js/` → `main.min.js` | sourcemaps → Babel → uglify → concat |
| `cssTask` | `css/**/*.css` | `dist/css/` → `*.min.css` | sourcemaps → PostCSS → CleanCSS → rename |
| `sassTask` | `scss/**/*.scss` | `dist/css/` → `*.min.css` | sourcemaps → sass → PostCSS → CleanCSS → rename |
| `prettify` | `js/**/*.js` | `js/` | Prettier (in-place) |
| `validate` | `js/**/*.js` | — | Prettier check |

**Default export:** `series(parallel(jsTask, cssTask, sassTask), validate)`

**Key differences from CJS version:**
- Targets `scss/` source (not `css/*.sass`)
- No `vendorTask` (vendor libs handled separately)
- No `cacheBustTask`
- No `watchTask`

**Status:** Neither Gulp file is invoked by any CI workflow. The Gulp pipeline exists for **local development use only**. The `npm run build` script in `package.json` has no value (empty), meaning CI effectively skips it.

---

### 4.3 Service Worker (`sw.js`)

The service worker implements a **stale-while-revalidate** pattern for Progressive Web App offline support.

**Cache name:** `"pwabuilder-precache"`

**Pre-cached files:** Only `index.html` (explicitly listed in `precacheFiles`)

**Caching strategy:**
1. **Install event:** Pre-caches `precacheFiles` list
2. **Activate event:** Immediately claims all clients (`self.clients.claim()`)
3. **Fetch event:** Three-tier strategy:
   - If request is in cache → serve from cache, then revalidate in background
   - If not in cache → fetch from network, cache if eligible, serve response
   - If network fails → serve cached fallback, or return HTTP 503

**What gets runtime-cached (`isStaticAssetRequest` filter):**
- Same-origin GET requests only
- No query parameters
- Destinations: `style`, `script`, `image`, `font`, `manifest`
- No `Range` requests
- No credentialed requests
- Only `basic` (same-origin) responses are cached (no opaque responses)

**Known limitations (per `Docs/PRODUCTION_BUILD_STRATEGY.md`):**
- Only `index.html` is pre-cached; all ~248 JS/CSS files are cached dynamically on first visit
- No content-hashing means cache invalidation depends on browser/SW revalidation logic
- Fragmented caching of 200+ AMD modules increases risk of partial cache states during offline use

---

### 4.4 Translation Pipeline (`convert_po_to_json.py`)

**Language:** Python 3.x (no external dependencies beyond stdlib)

**Input:** `.po` files under `po/`  
**Output:** `.json` files under `locales/`

**Standard conversion:** Parses `msgid`/`msgstr` pairs, outputs `{ msgid: msgstr }` JSON. Falls back to `msgid` if `msgstr` is empty.

**Special case — Japanese:**
- `ja.po` + `ja-kana.po` → single `ja.json` with `{ key: { kanji: "...", kana: "..." } }` structure
- Both files must exist for the merge to succeed

**CI integration:** `po-to-json-validation.yml` runs this script on changed `.po` files and verifies the output is already committed. There is no automated commit-back; contributors must run the script locally.

---

## 5. End-to-End Deployment Flow

### 5.1 Environments

| Environment | URL | Audience | Update Frequency |
|-------------|-----|----------|-----------------|
| **Local dev** | `http://127.0.0.1:3000` | Developers | On `npm start` |
| **Preview (experimental)** | `https://sugarlabs.github.io/musicblocks` | Contributors, testers | Every push to `master` (GitHub Pages) |
| **Production** | `https://musicblocks.sugarlabs.org` | End users, educators | Manual, infrequent |

### 5.2 PR → `sugarlabs.github.io/musicblocks` Flow

```
Developer fork
     │
     ▼
Feature branch
     │
     ▼  PR opened/updated
     ├──► pr-category-check.yml  (label validation)
     ├──► linter.yml              (ESLint + Prettier, PR only)
     ├──► node.js.yml             (npm ci on Node 20 + 22)
     ├──► pr-jest-tests.yml       (Jest unit tests + coverage delta)
     ├──► pr-cypress-e2e.yml      (Cypress E2E, Chrome)
     ├──► security_scan.yml       (npm audit)
     └──► lighthouse-ci.yml       (Lighthouse desktop + mobile)
     │
     ▼  PR approved and merged to master
     │
     ├──► auto-rebase.yml         (rebase remaining open PRs)
     ├──► conflict-check.yml      (label PRs that couldn't be rebased)
     ├──► node.js.yml             (smoke test on master)
     ├──► pr-cypress-e2e.yml      (E2E on master)
     ├──► security_scan.yml       (audit on master)
     └──► lighthouse-ci.yml       (Lighthouse on master)
     │
     ▼
master branch HEAD
     │
     ▼  Automatic (GitHub Pages)
sugarlabs.github.io/musicblocks
(serves master branch directly, no build step)
```

### 5.3 `master` → `musicblocks.sugarlabs.org` Flow (Production)

```
sugarlabs.github.io/musicblocks (master)
     │
     │  MANUAL STEP — no automation
     │  Actor: Walter Bender (primary maintainer, ~5990 commits)
     │  Mechanism: Not documented in-repo (private server/hosting configuration)
     │  Frequency: Infrequent (verified releases: v3.7.1 on 2026-02-15,
     │                          version bump on 2025-12-31)
     ▼
musicblocks.sugarlabs.org (production)
```

**Key fact confirmed in CONTRIBUTING.md (#5657, 2026-02-14):**
> "production deployments of Music Blocks are **manual**. This means that even after your pull request is merged, your changes may not immediately appear."

**Who deploys:** Walter Bender is the primary maintainer identified as responsible for production releases, based on git history and commit authorship of release markers (`3.7.1 release`, `bump version number for new release`, deployment-related commits).

**How:** The exact deployment mechanism to `musicblocks.sugarlabs.org` is **not documented in this repository**. No workflow, Dockerfile, or deployment script exists in-tree for production. This is the primary operational gap.

**When:** No defined release cadence. Historical data suggests major releases every 1–3 months, with long gaps possible (e.g., Dec 2025 to Feb 2026).

**Where:** `musicblocks.sugarlabs.org` (presumed SugarLabs-hosted server; hosting infrastructure not documented here).

---

## 6. Workflow Dependency Graph

### 6.1 Trigger Relationships

```
                        ┌─────────────────────────────────────────────┐
                        │               TRIGGERS                       │
                        └─────────────────────────────────────────────┘

push to master ──────► auto-rebase.yml
                │       node.js.yml (build matrix)
                │       pr-cypress-e2e.yml
                │       security_scan.yml
                └──────► lighthouse-ci.yml

auto-rebase ──────────► conflict-check.yml  (workflow_run)
  completes

pull_request ─────────► linter.yml
  to master  │           node.js.yml
             │           pr-cypress-e2e.yml
             └─────────► security_scan.yml

pull_request_target ──► pr-category-check.yml
                  │      pr-jest-tests.yml
                  └────► lighthouse-ci.yml
                          conflict-check.yml (synchronize only)

schedule (daily) ─────► stale.yml
workflow_dispatch ─────► auto-rebase.yml
                          security_scan.yml
                          stale.yml
```

### 6.2 Shared Steps (Consolidation Targets)

The following steps are duplicated across workflows:

| Step | Workflows | Count per PR |
|------|-----------|-------------|
| `actions/checkout@v4` | All except stale | 7+ checkouts |
| `actions/setup-node@v5` | node.js.yml, lighthouse-ci.yml, linter.yml, pr-cypress-e2e.yml, pr-jest-tests.yml, security_scan.yml | 6+ setups |
| `npm ci` | node.js.yml (×2 matrix), lighthouse-ci.yml (×2 profiles), linter.yml, pr-jest-tests.yml (**×2** for master comparison), security_scan.yml | **8+ npm ci runs per PR** |
| Node 22 setup | lighthouse-ci.yml, pr-cypress-e2e.yml, pr-jest-tests.yml, security_scan.yml | 4+ identical setups |

### 6.3 Consolidation Opportunities

| Opportunity | Workflows Affected | Estimated Saving |
|-------------|-------------------|-----------------|
| Reusable workflow for `npm ci` + setup-node | All 6 workflows with npm ci | Eliminate ~5 redundant installs per PR |
| Cache `node_modules` across workflow runs | All npm ci steps | Significant speed improvement (dependency tree is stable) |
| Merge security scan into smoke test | `security_scan.yml` + `node.js.yml` | 1 fewer workflow run per push to master |
| Eliminate double `npm ci` in `pr-jest-tests.yml` | `pr-jest-tests.yml` | Halve the most expensive workflow |
| Run Lighthouse only on PRs (not on every push to master) | `lighthouse-ci.yml` | Reduce redundant audits |

---

## 7. Baseline DORA Metrics

These metrics are derived from `git log` analysis and documented sources. They represent estimates where exact data is not directly measurable.

### 7.1 Deployment Frequency

**To `sugarlabs.github.io/musicblocks` (master/experimental):**

GitHub Pages serves the `master` branch directly. Every push to master is an automatic deployment.

| Period | Commits to master | Avg per day |
|--------|------------------|-------------|
| 2026-01 | 163 | 5.3/day |
| 2026-02 | 295 | 10.5/day |
| 2026-03 | 166 | 5.4/day |
| 2026-04 | 415 | 13.8/day |
| 2026-05 (to 2026-05-25) | 95 | 3.8/day |
| **2025 full year** | **570** | **1.6/day** |

Peak single-day: **45 commits** (2026-04-08)

**Classification:** Elite (multiple deploys per day to the experimental URL)

**To `musicblocks.sugarlabs.org` (production):**

| Period | Known releases |
|--------|---------------|
| 2025-12-31 | Version bump (implied release) |
| 2026-02-15 | v3.7.1 official release |

**Classification:** Low (estimated 2–4 production deployments per year)

---

### 7.2 Lead Time for Changes

**Definition:** Time from PR merge to production availability.

| Environment | Lead Time |
|-------------|-----------|
| `sugarlabs.github.io/musicblocks` | **Near-instant** (~minutes after merge; GitHub Pages propagation) |
| `musicblocks.sugarlabs.org` | **Days to months** (depends entirely on maintainer manual action) |

**PR review cycle time:** The stale threshold is 60 days, suggesting some PRs wait weeks before review. No automated SLA metric is tracked. Based on commit volume and active DMP program, active contributor PRs are reviewed within days; older/inactive PRs may wait significantly longer.

**Lead time classification:** 
- For experimental: **Elite** (< 1 hour)
- For production: **Low** (1 month to 6 months between deployments)

---

### 7.3 Change Failure Rate

**Definition:** Proportion of deployments that cause a production incident requiring a revert or hotfix.

**Measured reverts to master (2024-01-01 to 2026-05-25):**

| Date | Description |
|------|-------------|
| 2026-02-18 | Revert Dependabot npm multi-package update |
| 2026-02-17 | Revert of specific commit |
| 2025-12-25 | Reverting #4834 |
| 2025-03-02 | Revert toolbar resizing regression |
| 2024-09-28 | Revert part of #4015 |
| 2024-05-30 | Revert to old jQuery (×2 commits) |

**Total reverts:** 7 in ~18 months  
**Total merges to master (2024-2025):** ~1,950  
**Estimated change failure rate:** **~0.36%** (well below the industry "Elite" threshold of <15%)

**Caveat:** This measures reverts to the `master` branch, not production incidents specifically. Since production deployments are manual and infrequent, most reverts happen before they reach production users.

---

### 7.4 Mean Time to Recovery (MTTR)

**Definition:** Time from production failure detected to service restored.

| Environment | MTTR |
|-------------|------|
| `sugarlabs.github.io/musicblocks` | **Minutes to hours** (merge a fix to master → auto-deploys) |
| `musicblocks.sugarlabs.org` | **Unknown/unbounded** (requires manual maintainer action; no on-call rotation documented) |

**Limitation:** No incident tracking system is documented in this repository. No post-mortems or incident reports were found in the git history. MTTR for production is entirely dependent on Walter Bender's availability — there is no documented escalation path or backup deployer.

**Classification:**
- For experimental: **Elite** (< 1 hour)  
- For production: **Cannot be measured** (no incident history; no documented SLA)

---

## 8. Secrets, Environment Variables, and Configuration

### 8.1 GitHub Secrets (Repository-Level)

| Secret | Used In | Purpose | Notes |
|--------|---------|---------|-------|
| `GITHUB_TOKEN` | `auto-rebase.yml`, `conflict-check.yml`, `pr-jest-tests.yml` | GitHub API access (rebase PRs, label management, post comments) | Automatically provided by GitHub Actions; no manual configuration required |

**No custom repository secrets are configured.** There are no deployment tokens, no external API keys, no signing keys, no container registry credentials in any workflow file.

### 8.2 Environment Variables (CI Workflows)

| Variable | Workflow | Values | Purpose |
|----------|----------|--------|---------|
| `LHCI_PROFILE` | `lighthouse-ci.yml` | `desktop` or `mobile` (from matrix) | Selects Lighthouse configuration profile |
| `GITHUB_OUTPUT` | Multiple | Automatic GitHub file | Step output passing |
| `GITHUB_ENV` | Multiple | Automatic GitHub file | Cross-step environment variable sharing |
| `GITHUB_EVENT_PATH` | `pr-jest-tests.yml` | Automatic | Read PR number from event payload |

### 8.3 Environment Variables (Express Server / Runtime)

| Variable | Default | Source | Purpose |
|----------|---------|--------|---------|
| `NODE_ENV` | `"development"` | Process environment | Controls caching headers, build mode; exposed to browser via `/env.js` |
| `HOST` | `"127.0.0.1"` | Process environment | Bind address for Express server |
| `PORT` | `3000` | Process environment | Listen port for Express server |

### 8.4 Browser-Exposed Runtime Configuration

The Express server dynamically generates a `/env.js` script that exposes:

| Variable | Value |
|----------|-------|
| `window.MB_ENV` | `process.env.NODE_ENV \|\| "development"` |
| `window.MB_IS_DEV` | `process.env.NODE_ENV !== "production"` |

These are served with aggressive `no-cache` headers to prevent stale environment state.

### 8.5 Configuration Files Summary

| File | Purpose | Consumed by |
|------|---------|-------------|
| `package.json` | Project metadata, scripts, deps | All workflows, local dev |
| `jest.config.js` | Test patterns, coverage thresholds, environment | `pr-jest-tests.yml`, local `npm test` |
| `cypress.config.js` | E2E viewport, isolation settings | `pr-cypress-e2e.yml`, local `npm run cypress:run` |
| `lighthouserc.js` | Lighthouse collect URLs, assertions, upload target | `lighthouse-ci.yml` |
| `eslint.config.mjs` | Lint rules (flat config) | `linter.yml`, pre-commit hook |
| `.prettierrc` / inline | Formatting rules | `linter.yml`, pre-commit hook, gulp validate |
| `gulpfile.js` | CJS build pipeline (local only) | Not used by CI |
| `gulpfile.mjs` | ESM build pipeline (local only) | Not used by CI |

### 8.6 Pre-Commit Hooks (Husky + lint-staged)

`package.json` configures `lint-staged` to run on `*.{js,mjs}` staged files:
```
eslint --max-warnings=0 --fix --no-warn-ignored
prettier --write
```

Husky is initialized via the `prepare` npm script. These hooks enforce formatting locally before commit, complementing `linter.yml` which runs the same checks in CI.

### 8.7 GitHub Environments (Current State)

**No GitHub Environments are configured.** There are no `environment:` keys in any workflow file. This means:
- No environment-specific secrets
- No deployment protection rules
- No required reviewers for deployments
- No environment-scoped `GITHUB_TOKEN` restrictions

This is a prerequisite gap for implementing deployment automation with proper access controls.

---

## 9. Identified Risks and Consolidation Opportunities

### 9.1 Critical Gaps

| # | Risk | Severity | Details |
|---|------|----------|---------|
| R1 | **No automated production deployment** | 🔴 Critical | `musicblocks.sugarlabs.org` is deployed manually by a single maintainer. No documented process, no backup deployer, no deployment runbook. |
| R2 | **Single point of failure: Walter Bender** | 🔴 Critical | All production deployments depend on one person's availability. No documented escalation path. |
| R3 | **No GitHub Environments configured** | 🔴 Critical | Cannot implement deployment protection, required reviewers, or environment secrets without configuring this first. |
| R4 | **No deployment workflow in-repo** | 🟠 High | There is no workflow file for deploying to either `sugarlabs.github.io/musicblocks` via Pages API or to production. The GitHub Pages serving of master is implicit (configured in repo settings). |
| R5 | **Gulp pipeline not connected to CI** | 🟠 High | `gulpfile.js` and `gulpfile.mjs` exist but are never invoked by any workflow. CI does not minify, bundle, or cache-bust. Production likely serves unbundled ~12.94 MB of raw JS. |
| R6 | **Service worker pre-caches only `index.html`** | 🟡 Medium | ~248 JS/CSS files are dynamically cached, creating partial-cache risk in offline mode. |
| R7 | **No release tagging** | 🟡 Medium | `git tag` returns no results. Version bumps and releases are tracked only by commit message, not by Git tags. |
| R8 | **pr-jest-tests.yml runs npm ci twice** | 🟡 Medium | The master branch coverage comparison doubles the workflow duration and compute cost. |
| R9 | **No LHCI server — reports on temporary storage** | 🟡 Medium | Lighthouse reports are uploaded to `temporary-public-storage` which expires. No historical performance trend is preserved. |
| R10 | **Security scan scope limited to npm audit** | 🟡 Medium | No SAST, no secret scanning, no dependency review GitHub Action. |

### 9.2 Consolidation Opportunities

| # | Opportunity | Impact |
|---|-------------|--------|
| C1 | Create a reusable workflow for `checkout + setup-node + npm ci` | Eliminates 8+ redundant installs per PR |
| C2 | Cache `node_modules` with `actions/cache@v4` keyed on `package-lock.json` hash | Reduces install time from ~60s to ~5s on cache hit |
| C3 | Eliminate double `npm ci` in `pr-jest-tests.yml` by using artifacts to pass coverage data | Halves the most expensive workflow |
| C4 | Merge `security_scan.yml` into `node.js.yml` | One fewer workflow trigger per push |
| C5 | Add a GitHub Pages deployment workflow (replacing implicit Pages settings) | Makes deployment observable, configurable, and auditable |
| C6 | Create a `production-release.yml` workflow with environment protection | Formalizes the manual process; adds required approvers |
| C7 | Add Git tagging to the release process | Enables `git describe`, changelog automation, and release artifacts |
| C8 | Connect Gulp pipeline (or esbuild) to CI for asset minification | Reduces production payload from ~12.94 MB unbundled |
| C9 | Add LHCI server (or GitHub Gist target) for persistent performance history | Enables performance regression detection over time |

---

*This document reflects the verified state of the repository as of 2026-05-25. All findings are based on direct source inspection and `git log` analysis. No assumptions have been made about behavior not confirmed by the source files.*
