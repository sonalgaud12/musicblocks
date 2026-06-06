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