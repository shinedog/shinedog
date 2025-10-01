# ADR-2025-10-01-github-repo-status-pipeline

**Title:** Central GitHub Actions pipeline to assess health/status of all repositories  
**Date:** 2025-10-01  
**Status:** Proposed  
**Deciders:** Nick Adriaanse  
**Context:**  
- You maintain many repositories across user/org namespaces (`shinedog`, `adriaanse-dev`, etc.). Manual checks are error-prone.  
- You want a single scheduled pipeline that inventories repos and evaluates key hygiene/security signals with minimal dependencies 
- Prefer ISO dates, minimal external services, and artifacts you can graph later (CSV/JSON).  
- Secrets should be minimal; avoid long-lived tokens when possible.  
- Output should fail fast on critical violations, but still publish a full report.

**Problem Statement:**  
Provide an automated, repeatable job that:
1. Enumerates all accessible repositories across specified owners.
2. Evaluates a fixed set of checks.
3. Emits machine-readable results (JSON + CSV) and a human-readable summary (Markdown).
4. Fails the workflow if any **Critical** checks fail; warns (but doesn’t fail) for **Advisory** checks.

**In-Scope Checks (initial):**  
_Critical (fail build):_  
- Default branch exists and is protected (require PRs, reviews = ≥1).  
- Branch protection enforces status checks if workflows exist.  
- Security: Dependabot alerts queryable; code scanning enabled if available.  
- Repo visibility matches allowlist (no accidental public).  
- License present.  
- Actions enabled and not using deprecated `set-output`.  
- Secret scanning enabled (if eligible).

_Advisory (warn only):_  
- Archived/disabled state.  
- Last push > 180 days.  
- Missing topics/description/homepage.  
- Open PRs > threshold (default 25).  
- Default branch behind by > N commits compared to most active branch.  
- Releases missing for repos labeled “releaseable”.

**Decision:**  
Implement a **central orchestrator** workflow in a single “platform” repo using **GitHub Actions** on a cron schedule and manual dispatch. Discovery + checks run via a **small Python script** (stdlib + `requests`) calling GitHub REST/GraphQL. Results are uploaded as artifacts and optionally committed to a `reports/` directory. Violations create/refresh a GitHub Issue per repo with a stable title.

**Rationale:**  
- One place to run, one artifact to read.  
- Python keeps deps minimal and logic clear; no build toolchain.  
- GitHub REST/GraphQL are stable; avoids third-party SaaS.  
- `GITHUB_TOKEN` preferred; falls back to a PAT only when required (e.g., cross-org visibility).  
- Issue-per-repo makes remediation trackable.

**Alternatives Considered:**  
1. **Per-repo workflow:** too noisy; duplication.  
2. **GitHub App + external runner:** stronger security, more work; defer.  
3. **gh CLI + bash:** convenient but adds binary dependency and parsing quirks; Python is cleaner.

**Security:**  
- Default: use `GITHUB_TOKEN` with `permissions:` narrowed to `contents:read`, `security-events:read`, `issues:write` (for findings), `actions:read`, `administration:read` (for protections) where allowed.  
- If `administration:read` isn’t available to GITHUB_TOKEN for some orgs, use a repo secret `GH_TOKEN_RO` scoped to `read:org, repo, security_events`.  
- No secrets in logs. Mask tokens.  
- Artifacts contain only repo metadata; no code.

**Outputs:**  
- `reports/YYYY-MM-DD/repos.json` — full raw results.  
- `reports/YYYY-MM-DD/repos.csv` — flattened table.  
- `reports/YYYY-MM-DD/summary.md` — concise human report with counts + top offenders.  
- GitHub Issues with label `repo-health` on offending repos (upserted).

**Failure Policy:**  
- Any **Critical** finding → job fails.  
- Advisory findings → job succeeds but summary lists warnings.

**Schedule & Triggers:**  
- `cron: "17 3 * * *"` (daily, 03:17 America/New_York).  
- `workflow_dispatch` with inputs: owners (comma-sep), critical_only (bool).

**Ownership:**  
- Repo owner: Nick.  
- Code owners: `@shinedog`.

**Migration Plan:**  
1. Merge this ADR.  
2. Add workflow YAML in `.github/workflows/repo-health.yml`.  
3. Add `tools/repo_health/` with Python script.  
4. (Optional) Add CODEOWNERS and default labels.  
5. First run in “report-only” mode to baseline.  
6. Enable failing on Critical after review.

**Acceptance Criteria:**  
- Single action run inventories all target owners and outputs artifacts.  
- At least one Critical test can be simulated and causes failure.  
- Issues are created/updated idempotently with stable titles.  
- Reports commit to `reports/` on `main` without leaking secrets.  
- Total run time < 10 minutes for current repo set.

**Open Questions / TODO:**  
- Finalize exact branch protection rules per repo class.  
- Decide which repos are allowed to be public.  
- Add SARIF ingestion for code scanning parity (future).  
- Add SBOM presence check (release assets) later.

**Consequences:**  
- Centralized visibility
- Occasional permission edge cases with `GITHUB_TOKEN`; PAT fallback required 
- Small maintenance burden for the Python checker as API fields evolve.