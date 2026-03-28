# Git Workflow Standards

A consistent branching and commit strategy reduces merge conflicts, enables reliable CI/CD, and creates a legible history for debugging and auditing.

---

## Branching Strategy

Use **trunk-based development** for teams deploying frequently (multiple times per day), or **GitFlow** for teams with scheduled releases. Choose one; document it in the repo README.

### Trunk-Based Development (Default Recommendation)

```
main          (always deployable, protected)
  └── feature/USER-123-add-oauth-login
  └── fix/USER-456-null-pointer-in-auth
  └── chore/upgrade-dependencies
```

- `main` is the single trunk; all CI/CD deployments target it
- Feature branches live ≤3 days; long-lived branches indicate a design problem
- Merge via PR with at least 1 review + all CI checks passing
- Use **feature flags** to ship incomplete features without blocking deployments

### GitFlow (Scheduled Releases)

```
main           (production; only receives merges from release/ and hotfix/)
develop        (integration branch; default PR target)
  └── feature/USER-123-add-oauth-login
  └── fix/USER-456-null-pointer
release/v2.1   (staging stabilization; bug fixes only)
hotfix/v2.0.1  (emergency production fixes; merges to main AND develop)
```

---

## Branch Naming Conventions

```
<type>/<ticket-id>-<short-description-in-kebab-case>

Examples:
feature/AUTH-101-google-sso
fix/API-234-missing-refresh-token-expiry
chore/INFRA-89-upgrade-node-22
docs/DOCS-12-add-deployment-runbook
test/QA-55-add-rbac-integration-tests
```

| Prefix | When to Use |
|--------|-------------|
| `feature/` | New functionality |
| `fix/` | Bug fix |
| `hotfix/` | Emergency production fix |
| `chore/` | Tooling, dependencies, scaffolding (no production code change) |
| `docs/` | Documentation-only changes |
| `test/` | Adding or fixing tests |
| `refactor/` | Code restructure with no behavior change |
| `perf/` | Performance improvements |
| `release/` | GitFlow release stabilization branches |

---

## Commit Message Standards (Conventional Commits)

All commits **must** follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<optional scope>): <short imperative summary>

[optional body — explain WHY, not WHAT. What is visible in the diff.]

[optional footer: BREAKING CHANGE: <description>]
[optional footer: Closes #<issue> or Refs TICKET-123]
```

### Types

| Type | When to Use |
|------|-------------|
| `feat` | New feature (triggers MINOR version bump) |
| `fix` | Bug fix (triggers PATCH version bump) |
| `chore` | Build, tooling, dependencies |
| `docs` | Documentation only |
| `test` | Tests only — no production code |
| `refactor` | Code change that is neither a fix nor a feature |
| `perf` | Performance improvement |
| `ci` | CI/CD pipeline changes |
| `style` | Formatting, whitespace — no logic change |
| `revert` | Revert a previous commit |
| `BREAKING CHANGE` | In footer; triggers MAJOR version bump |

### Examples

```bash
# ✅ Good commit messages
feat(auth): add OAuth2 Google SSO login
fix(api): return 404 instead of 500 when user not found
chore(deps): upgrade prisma to 5.14 for Node 22 support
docs(readme): add local dev setup instructions
test(users): add IDOR tests for GET /users/:id endpoint
refactor(payments): extract stripe webhook handler to dedicated service
feat(notifications)!: replace email-only alerts with multi-channel
BREAKING CHANGE: NotificationService.send() signature changed to accept Channel[]

# ❌ Bad commit messages
fix stuff                          # too vague
WIP                                # not a meaningful commit
fixed the thing                    # no type, no context
Added feature, updated tests, fixed lint, bumped version  # too many concerns in one commit
```

### Rules

- **Subject line**: ≤50 characters; imperative mood ("add" not "added" or "adding"); no period at end
- **Body**: Bullet points only — one line per change; no prose; the diff speaks for itself
- **Keep it succinct** — do not write lengthy explanations or restate what the diff already shows
- **One logical change per commit** — do not mix features, fixes, and formatting in a single commit
- **Do not commit directly to `main` or `develop`** — always use a PR

---

## Pull Request Standards

### PR Size Limits

| Lines Changed | Status |
|--------------|--------|
| ≤200 lines | Green — reviewable |
| 201–400 lines | Yellow — acceptable with good description |
| 401–800 lines | Red — consider splitting |
| >800 lines | Rejected until split — exceptions require tech lead sign-off |

Automated changes (dependency bumps, generated code, lock files) are exempt from line limits.

### PR Title

Follow the same Conventional Commits format as commit messages:
```
feat(auth): add OAuth2 Google SSO login
```

### PR Description Template

```markdown
## Summary
<!-- 2-4 sentences explaining what this PR does and why. Link to the design doc or ticket. -->

## Changes
<!-- Bullet list of notable changes. Not a line-by-line diff summary. -->
- Added `OAuthController` that handles Google redirect callbacks
- Registered Google PKCE flow in `AuthService`
- Added `oauth_state` table to persist PKCE verifier per session

## Security Checklist
- [ ] Input validated with schema
- [ ] Authorization checked (who can call this endpoint?)
- [ ] No secrets hardcoded
- [ ] Sensitive data not logged
- [ ] Dependency additions reviewed for known CVEs

## Testing
- [ ] Unit tests added / updated
- [ ] Integration tests cover the happy path and auth/authz edge cases
- [ ] Tested locally with: <!-- describe what you tested and how -->

## Breaking Changes
<!-- List any breaking changes to APIs, contracts, or config. -->
None

## Screenshots / Demo
<!-- For UI changes, include before/after screenshots or a short screen recording. -->
```

### Review Requirements

| Environment | Approvals Required |
|-------------|-------------------|
| feature branch → develop | 1 approval |
| develop → main | 2 approvals (1 must be a tech lead) |
| hotfix → main | 1 approval from tech lead |

- All CI checks must pass before merge
- Reviewer comments marked `[BLOCKER]` or `[SECURITY]` must be resolved before approval
- Authors do not approve their own PRs

---

## Merge Strategy

| Scenario | Merge Method |
|----------|-------------|
| Feature branch → trunk/develop | **Squash and merge** (clean linear history) |
| Release branch → main (GitFlow) | **Merge commit** (preserve release history) |
| Hotfix → main | **Merge commit** (preserve hotfix history) |
| Main → develop (GitFlow sync) | **Merge commit** |

Do **not** use force-push (`--force`) on shared branches (`main`, `develop`, `release/*`).

---

## Protected Branch Rules

Configure the following branch protection rules for `main` (and `develop` in GitFlow):

```yaml
# GitHub branch protection settings
required_status_checks:
  strict: true
  contexts:
    - CI: Unit Tests
    - CI: Integration Tests
    - CI: Security Scan
    - CI: Coverage Check
enforce_admins: false          # admins must also follow the rules
required_pull_request_reviews:
  required_approving_review_count: 1
  dismiss_stale_reviews: true
  require_code_owner_review: true
allow_force_pushes: false
allow_deletions: false
```

---

## Tagging and Releases

Use [Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH[-prerelease]`

```bash
# Tag a release
git tag -a v2.1.0 -m "feat: multi-channel notification support"
git push origin v2.1.0

# Pre-release / release candidate
git tag -a v2.1.0-rc.1 -m "Release candidate 1 for v2.1"
```

Automate version bumping with `semantic-release` or `release-please` based on Conventional Commits.

---

## Secrets Management in Git

The following must **never** be committed:

- API keys, passwords, tokens, certificates
- `.env` files (except `.env.example` with placeholder values)
- Private keys (`*.pem`, `*.key`, `*.p12`)
- AWS, GCP, Azure credentials
- Database connection strings with credentials

Use a [pre-commit hook](https://pre-commit.com/) with `gitleaks` or `detect-secrets` to prevent accidental secret commits:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

If a secret is accidentally committed, treat it as **immediately compromised** — rotate the secret before attempting to scrub git history.
