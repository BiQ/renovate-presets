# Renovate Presets for the BiQ Organization

> Centralized, composable Renovate configuration presets for consistent, safe, and observable dependency management across BiQ repositories.

---

## Table of Contents

1. Motivation
2. What Is a Renovate Preset?
3. Quick Start
4. Available Presets (Overview)
5. Preset Naming Conventions
6. Layering & Composition Strategy
7. Usage Examples
8. Ruby (Rails + MSSQL) Specific Guidance
9. Go Module Strategy
10. JavaScript/TypeScript Strategy
11. Docker Image Strategy
12. GitHub Actions Strategy
13. Terraform & IaC Strategy
14. Dependency Grouping Patterns
15. Scheduling & Rate Control
16. Versioning Philosophy for Presets
17. Creating / Updating a Preset
18. Validation & Testing
19. Observability & Metrics
20. Security & Supply Chain Considerations
21. FAQ
22. Contributing
23. License
24. Roadmap / Future Ideas
25. Appendix: Reference Snippets

---

## 1. Motivation

Managing dependency updates consistently across multiple services is difficult when:
- Each repo reinvents Renovate configuration.
- Update policies differ (pin vs range, schedules, grouping).
- Security posture (e.g., automerging patch security updates) is uneven.

This repository centralizes **organization-wide Renovate presets** so all BiQ projects can inherit best practices:
- Deterministic layering and minimal duplication.
- Faster adoption of policy changes (edit once; benefit everywhere).
- Enforced baseline constraints (e.g., ignore deprecated sources, standard grouping).
- Controlled noise levels (rate limiting, grouping, scheduling).
- Prioritized security updates.

---

## 2. What Is a Renovate Preset?

A Renovate "preset" is a sharable JSON (or JavaScript) configuration fragment that can be referenced from other Renovate configs using `extends`. Presets allow:
- Reuse of patterns (e.g., "group all devDependencies").
- Enforcement of org standards.
- Composition: A repo can extend multiple presets.

Technically each preset lives under the `renovate.json` root or in `./presets/*.json5` (recommended for organization). Renovate supports:
- Publishing via GitHub (referenced using `github>owner/repo:filename`).
- Hierarchical `extends` arrays.
- Merging rules (later entries override earlier ones for conflicting keys).

---

## 3. Quick Start

Add (or edit) `.renovaterc.json` or `renovate.json` in your target repository:

```json
{
  "extends": [
    "github>BiQ/renovate-presets:base",
    "github>BiQ/renovate-presets:security-tight",
    "github>BiQ/renovate-presets:ruby-rails",
    "github>BiQ/renovate-presets:go-modules"
  ]
}
```

Then open (or re-run) Renovate onboarding PR. Renovate will fetch these presets directly.

---

## 4. Available Presets (Overview)

(Exact list evolves; see `/presets` directory.)

| Preset | Purpose |
|--------|---------|
| `base` | Core defaults: stability days, minimal noise, standard labels. |
| `security-tight` | Aggressive security updates & automerge for safe patches. |
| `security-standard` | Balanced security posture (no automerge by default). |
| `ruby-rails` | Ruby + Rails + MSSQL gem grouping, bundler config. |
| `go-modules` | Go module update strategy, internal module pinning. |
| `js-app` | Node app dependencies: dev/prod grouping, schedule. |
| `js-lib` | Library-specific approach (semver ranges retained). |
| `docker-images` | Docker base image policy, digest pinning. |
| `terraform` | Terraform provider grouping + module update strategy. |
| `github-actions` | Conservative update schedule for actions. |
| `ci-fast` | High frequency (e.g., experimental repos). |
| `slow-and-steady` | Weekly rollups for low-criticality services. |
| `monorepo` | Dependency grouping for large multi-package codebases. |

---

## 5. Preset Naming Conventions

- Use `kebab-case`.
- Domain / ecosystem first: `ruby-*`, `go-*`, `terraform-*`.
- Policy adjectives: `*-tight`, `*-standard`, `*-fast`.
- Avoid ambiguous names; prefer `docker-images` over `docker`.
- If variant: append suffix: `ruby-rails-enterprise`, `ruby-rails-minimal`.

---

## 6. Layering & Composition Strategy

Recommended order (top earlier means more "foundational"):

1. `base`
2. Ecosystem-specific (e.g., `ruby-rails`, `go-modules`, `docker-images`)
3. Policy overlays (`security-tight`, `slow-and-steady`)
4. Repo-specific final overrides (defined inside the target repo)

Conflict resolution heuristics:
- Schedules: last wins (prefer repo-specific).
- Labels: arrays merge (deduplicate).
- Package rules: Renovate merges by matcher criteria; ensure specificity.

---

## 7. Usage Examples

### Basic service (Rails + Go microservice sidecar):
```json
{
  "extends": [
    "github>BiQ/renovate-presets:base",
    "github>BiQ/renovate-presets:ruby-rails",
    "github>BiQ/renovate-presets:go-modules",
    "github>BiQ/renovate-presets:security-standard"
  ]
}
```

### Low-risk internal tool:
```json
{
  "extends": [
    "github>BiQ/renovate-presets:base",
    "github>BiQ/renovate-presets:js-app",
    "github>BiQ/renovate-presets:ci-fast"
  ]
}
```

### Infrastructure repo (Terraform + GitHub Actions):
```json
{
  "extends": [
    "github>BiQ/renovate-presets:base",
    "github>BiQ/renovate-presets:terraform",
    "github>BiQ/renovate-presets:github-actions",
    "github>BiQ/renovate-presets:security-tight"
  ]
}
```

---

## 8. Ruby (Rails + MSSQL) Specific Guidance

Challenges:
- Some gems bump frequently (e.g., `rubocop-*`, `aws-sdk-*`).
- Rails minor/major upgrades need isolation.
- Database driver gems (e.g., `tiny_tds`, `activerecord-sqlserver-adapter`) may require manual review.

Preset strategies:
- Group linters: `rubocop`, `brakeman`, test gems.
- Isolate `rails` + `activerecord-sqlserver-adapter` updates (no automerge).
- Pin engine versions if internal engines exist.
- Use `rangeStrategy: "update-lockfile"` for consistent Gemfile.lock alignment.
- Optional stability days for production-critical gems: e.g., 2–3 days on patch.

Example snippet (inside `ruby-rails` preset):

```json5
{
  "packageRules": [
    {
      "matchPackageNames": ["rails", "activerecord-sqlserver-adapter"],
      "groupName": "rails-core",
      "automerge": false,
      "stabilityDays": 3
    },
    {
      "matchPackagePatterns": ["^rubocop", "^brakeman$"],
      "groupName": "ruby-static-analysis",
      "schedule": ["after 02:00 on monday"]
    },
    {
      "matchPackageNames": ["tiny_tds"],
      "automerge": false,
      "labels": ["db-driver"]
    }
  ]
}
```

---

## 9. Go Module Strategy

Key points:
- Go modules often safe to automerge patch versions.
- Major bumps may introduce API breaks; isolate with labels.
- Internal modules (BiQ org) may require synchronous update across repos (consider grouping).

Possible rule fragment:

```json5
{
  "packageRules": [
    {
      "matchDatasources": ["gomod"],
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true
    },
    {
      "matchDatasources": ["gomod"],
      "matchUpdateTypes": ["major"],
      "labels": ["go-major-review"],
      "automerge": false
    },
    {
      "matchSourceUrls": ["https://github.com/BiQ/*"],
      "groupName": "internal-go-modules",
      "schedule": ["before 05:00 on tuesday"]
    }
  ]
}
```

---

## 10. JavaScript/TypeScript Strategy

General:
- Distinguish `dependencies` vs `devDependencies`.
- Group test frameworks + tooling.
- Pin critical infra libs (`express`, security libs).
- Use lockFile maintenance weekly.

Snippet:

```json5
{
  "packageRules": [
    {
      "matchDepTypes": ["devDependencies"],
      "groupName": "js-dev-tooling",
      "schedule": ["after 01:00 on monday"]
    },
    {
      "matchPackageNames": ["express", "jsonwebtoken"],
      "stabilityDays": 2
    }
  ],
  "lockFileMaintenance": {
    "enabled": true,
    "schedule": ["before 04:00 on sunday"],
    "automerge": true
  }
}
```

---

## 11. Docker Image Strategy

Policy:
- Prefer digest pinning for immutability.
- Group base images (e.g., `ruby`, `golang`, `postgres`, `mcr.microsoft.com/mssql/server`).
- Automerge patch-level for base images if security scanning passes (future integration).

Example:

```json5
{
  "packageRules": [
    {
      "matchDatasources": ["docker"],
      "pinDigests": true
    },
    {
      "matchPackagePatterns": ["^ruby$", "^golang$", "^mcr.microsoft.com/mssql/server$"],
      "groupName": "core-base-images",
      "automerge": false
    }
  ]
}
```

---

## 12. GitHub Actions Strategy

Practice:
- Group all action updates weekly to reduce noise.
- Automerge patch/minor for trusted publishers (e.g., `actions/*`, `github/*`), but manual review for 3rd party.

Snippet:

```json5
{
  "packageRules": [
    {
      "matchDatasources": ["github-actions"],
      "groupName": "github-actions-weekly",
      "schedule": ["before 03:00 on saturday"]
    },
    {
      "matchPackagePatterns": ["^actions/"],
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true
    }
  ]
}
```

---

## 13. Terraform & IaC Strategy

- Provider versioning: avoid surprise major jumps.
- Group providers by vendor (AWS, Azure, etc.).
- Module updates require manual review.

Example:

```json5
{
  "packageRules": [
    {
      "matchDatasources": ["terraform-provider"],
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true
    },
    {
      "matchDatasources": ["terraform-provider"],
      "matchUpdateTypes": ["major"],
      "labels": ["terraform-major"],
      "automerge": false
    },
    {
      "matchDatasources": ["terraform-module"],
      "automerge": false
    }
  ]
}
```

---

## 14. Dependency Grouping Patterns

Patterns used broadly:
- Ecosystem grouping (e.g., `ruby-static-analysis`, `js-dev-tooling`).
- Security critical isolation (e.g., `rails-core`, `openssl-libs`).
- Internal vs external separation (Org-owned modules).
- Temporal grouping (batch updates weekly vs daily).

---

## 15. Scheduling & Rate Control

Scheduling reduces CI queue contention:
- Nightly small updates.
- Weekly rollups (lockfile maintenance).
- Critical security updates: immediate (no schedule + stricter automerge rules).

Rate limiting:
```json5
{
  "prHourlyLimit": 4,
  "prConcurrentLimit": 10
}
```

For high-volume monorepos, escalate via `ci-fast` preset.

---

## 16. Versioning Philosophy for Presets

Since presets influence policy across many repos:
- Treat changes as "configuration API".
- Use Git tags for stable snapshot references (optional).
- Document breaking changes (e.g., enabling new automerge set).
- Consider semantic version tags for the repo itself: `vMAJOR.MINOR.PATCH`.

---

## 17. Creating / Updating a Preset

Workflow:
1. Add / modify file under `presets/NAME.json5`.
2. Run validation (see section 18).
3. Document new preset in README (Available Presets).
4. Open PR with change log bullet:
   - Added preset: `go-modules-internal`
   - Changed: stricter stability on `rails`.
5. Merge; dependent repos naturally pick it up next Renovate cycle.

---

## 18. Validation & Testing

Local validation:
```bash
docker run --rm -v "$PWD":/repo renovate/renovate \
  renovate-config-validator --config-file=presets/ruby-rails.json5
```

Full synthetic test (simulate config):
```bash
docker run --rm -e LOG_LEVEL=debug -v "$PWD":/repo renovate/renovate \
  renovate --dry-run --config-file=presets/ruby-rails.json5 some/fake-repo
```

Automated CI suggestions:
- JSON5 linting.
- Renovate config validator step.
- Generation of a summary matrix for each preset (packages matched count).

---

## 19. Observability & Metrics

Future directions:
- Track PR volume per preset extension set.
- Alert on spikes (e.g., dependency flood).
- Record automerge success / failure counts.
- Security update latency (time from advisory published to PR merged).

---

## 20. Security & Supply Chain Considerations

Checklist enforced via presets:
- Digest pinning for Docker.
- Automerge only for trusted namespaces.
- Stability days for high-impact foundational libs.
- Isolation of major updates.
- Potential integration with vulnerability scanners (e.g., GitHub Dependabot alerts correlation).

---

## 21. FAQ

Q: Can I override a rule locally?
A: Yes—local config entries later in `extends` or directly in project file take precedence.

Q: How do I disable a preset rule?
A: Match the same selector and set a counteracting property (e.g., `automerge: false`).

Q: Can I reference a tag instead of `main`?
A: Yes: `github>BiQ/renovate-presets#v1.2.0:base`.

Q: Why JSON5?
A: Comments + trailing commas + less verbosity.

---

## 22. Contributing

1. Open issue describing motivation.
2. Draft preset adhering to naming conventions.
3. Include examples and justification.
4. Validate + open PR.
5. Request review from maintainers.

Coding style:
- Prefer JSON5 for clarity.
- Order keys: match selectors first (`matchPackageNames`, etc.) then behaviors (`groupName`, `automerge`).

---

## 23. License

(Select appropriate license—e.g., MIT or Apache 2.0. Add `LICENSE` file. Until then this repository is effectively proprietary within the BiQ organization.)

---

## 24. Roadmap / Future Ideas

- Auto-generated documentation from preset metadata.
- Integration with SBOM diff tools.
- Risk scoring per dependency update (CVSS integration).
- ChatOps commands to temporarily throttle updates.
- Preset for MSSQL driver ecosystem (NuGet packages if .NET services exist).

---

## 25. Appendix: Reference Snippets

### Base Preset (illustrative draft)
```json5
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "labels": ["dependencies"],
  "enabledManagers": [
    "bundler",
    "gomod",
    "npm",
    "dockerfile",
    "github-actions",
    "terraform"
  ],
  "rangeStrategy": "update-lockfile",
  "timezone": "UTC",
  "prHourlyLimit": 4,
  "prConcurrentLimit": 10,
  "dependencyDashboard": true,
  "rebaseWhen": "conflicted",
  "semanticCommits": true,
  "packageRules": [
    {
      "matchUpdateTypes": ["major"],
      "labels": ["major-update"],
      "automerge": false
    },
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "pr",
      "platformAutomerge": true
    }
  ]
}
```

### Security Tight Overlay
```json5
{
  "packageRules": [
    {
      "matchDepTypes": ["dependencies"],
      "matchUpdateTypes": ["minor", "patch"],
      "stabilityDays": 0
    },
    {
      "matchPackagePatterns": ["(?i)crypto", "(?i)jwt", "(?i)openssl"],
      "labels": ["security-critical"],
      "automerge": false
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security", "vulnerability"]
  }
}
```

---

## Badge Suggestions (Add Once CI Is Set Up)

| Badge | Purpose |
|-------|---------|
| Renovate enabled | Visibility Renovate is active |
| Config Validator | Confirms no schema failures |
| Release (tag) | Latest preset version |
| Security | Dependabot / Advisory status |

Example placeholder:

```
[![Renovate Enabled](https://img.shields.io/badge/renovate-enabled-brightgreen.svg)](https://docs.renovatebot.com/)
```

---

## Final Notes

This README is a living document—update as presets evolve. Keep changes transparent and announce high-impact shifts (e.g., enabling automerge for new ecosystems).

For questions, open an issue or start a discussion thread (consider enabling GitHub Discussions in this repository).

---
