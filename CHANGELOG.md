# Changelog

All notable changes to the `salesforce-alm-github` skill are documented here. This project adheres to [Semantic Versioning](https://semver.org/).

## [2.1.0] — 2026-06-01

### Added

- **Pattern 8: Sandbox Lifecycle with Data Mask & Seed** — native Salesforce refresh → mask → seed chain using Data Mask & Seed on Core (BETA April 2026, GA July 2026). Anchored to verified Salesforce documentation:
  - *The 5 Stages of Agent Application Lifecycle Management* (Salesforce, official 5-stage ALM framework)
  - *FY27 Salesforce Data Mask & Seed — First Call Deck (FCD)*
  - *Sandboxes FY27 Use Cases Deck*
  - *Ultimate Guide to Data Masking & Seeding* (sandbox tier table, 5 seeding challenges, 4 approaches, NIST 30x cost-of-bugs citation)
  - *FY26 Data Mask & Seed Use Cases Deck* (sensitive data inventory, use case taxonomy)
- **Appendix: Matrix Validation with Managed Packages** — design constraints, `RunLocalTests` behavior with namespaced tests, per-sandbox licensing implications (Certinia, Conga, SBQQ, Vlocity), test-data setup factories, and contractual due-diligence checklist for ISV-heavy customers.
- Three eval cases (11, 12, 13) covering Pattern 8: GitHub Actions integration, third-party migration, HIPAA de-identification.

### Changed

- Pattern 1 (Matrix Strategy) now cross-references the new managed-package appendix at the end of the skill.
- README updated to reflect **eight** patterns (was seven).
- `evals.json` version bumped from `2.0.0` to `2.1.0`; eval count from 10 to 13.

### Notes

- v2.1.0 explicitly recommends **native Salesforce tooling** (Data Mask & Seed + DevOps Center + Sandboxes) over third-party for the common-case sandbox lifecycle. Third-party recommendations (Prodly, Flosum Data Migrator, Own Accelerate, AutoRABIT) remain documented as defensible non-defaults with explicit deviation criteria — this is a deliberate reversal of the pre-2026 bias that "native is incomplete."
- **No Agentforce Vibes recommendations in this version.** The pattern is human-authored with deterministic runtime — LLMs are not in the data path. See the related memory entry on LLM runtime placement.
- Lifecycle dates customers must plan around:
  - **Data Mask Managed Package end-of-service: November 2026**
  - **Own Accelerate end-of-life: December 2027**
  - **Salesforce Data Mask & Seed on Core GA: July 2026** (BETA April 2026)
  - **Data Cloud / Data 360 seeding (DLO → DLO): Coming Soon — not GA today**

## [2.0.0] — 2026-05-09

Reviewed by Claude Opus 4.7. The original v1.0.0 was authored with Sonnet 4.5 and contained several technically inaccurate patterns that needed correction before broader rollout. This release fixes those issues, adds two new patterns, and rewrites the Agentforce guidance to be honest about CI's limits.

### ⚠️ Breaking changes

- **Pattern 2 (OIDC) was conceptually replaced.** The original implementation claimed GitHub OIDC could authenticate directly to Salesforce, which is not true — Salesforce does not natively trust GitHub's OIDC issuer. Pattern 2 is now **Secure JWT + Custom Properties** (Tier 1) with an optional **Identity Broker** path (Tier 2) using HashiCorp Vault, AWS STS, or Azure Entra ID. If you copy-pasted the previous Pattern 2 workflow, it would have failed at runtime; the new pattern actually works.
- **Branch strategy guidance reversed.** The previous "GitFlow with release branches" recommendation for large teams was harmful for Salesforce metadata (long-lived branches accumulate drift). The skill now recommends trunk-based + environment branches for medium teams and unlocked packages (Pattern 7) for large teams. **GitFlow is now explicitly called out as an anti-pattern for Salesforce.**
- **Eval contract changed.** The eval suite expanded from 5 to 10 cases and adds negative assertions (e.g., "must NOT write secrets to disk", "must NOT recommend GitFlow"). Re-run evaluations against the new schema.

### 🚨 Security fixes

- **JWT private keys no longer written to disk.** Previous examples used `echo "${{ secrets.SF_JWT_KEY }}" > jwt-key.txt && rm jwt-key.txt`, which can leak through workflow logs (when `set -x` is enabled), runner caches, or post-job hooks. Now uses process substitution `<(printf '%s' "$SECRET")`, keeping the key in a transient file descriptor that the kernel reaps when the process exits.
- **SFDX Auth URLs no longer written to disk** for the same reason. All `sf org login sfdx-url` and `sf org login jwt` examples now use process substitution consistently.
- **`echo` → `printf '%s'`** for secret piping. `echo` may interpret backslashes in some shells; `printf '%s'` is portable.

### 🐛 Bug fixes

- **`sfdx-scanner` migrated to `sf code-analyzer`.** The `@salesforce/sfdx-scanner` plugin and `sfdx scanner:run` command were deprecated by Salesforce in 2024. Replaced with `@salesforce/plugin-code-analyzer` and `sf code-analyzer run` using the modern `--workspace`, `--rule-selector`, and `--severity-threshold` flags.
- **Code coverage validation no longer broken.** Previous example called `sf apex get test` without the required `--test-run-id`, parsed `testRunCoverage` as if it were a `"87%"` string (it's a numeric `87`), and used `bc` for floating-point comparison (not on every runner image). Now uses `sf apex run test --json --synchronous` with proper `jq` extraction and `awk` comparison.
- **Quick Deploy 4-day expiration warning added.** Salesforce-side validation results expire after 4 calendar days regardless of GitHub artifact retention. The skill now includes age-aware fallback logic (72-hour buffer) to prevent confusing `Validation Id is invalid` failures.
- **Custom Properties access via `gh api`.** Previous examples used `${{ vars.X }}` for Custom Properties, which only works for Repository or Organization Variables. Custom Properties must be read at workflow runtime via `gh api /repos/.../properties/values`.

### ✨ New patterns

- **Pattern 6: Delta Deployments with `sfdx-git-delta`.** Deploy only the metadata that changed between two git refs. Collapses 30-minute full deploys to under a minute. Includes `fetch-depth: 0` requirement, destructive-changes handling, and a caveat about needing periodic full-deploy reconciliation to catch drift between git and the org.
- **Pattern 7: Unlocked Package Development.** The Salesforce-recommended pattern for 20+ developer codebases. Decompose `force-app/` into versioned, independently installable packages with version-pinned dependencies. Includes Dev Hub authentication, `sf package version create --code-coverage`, and `sf package version promote` workflows.

### 🤖 Agentforce rework

- **Replaced grep-based "prompt injection scanning"** with structured XML validation using `xmllint`. The previous regex-based approach produced false positives (`grep -r "user.*input.*prompt"`) and false negatives, and pretended to do at deploy-time what only the runtime Einstein Trust Layer can actually do.
- **Added honest framing** distinguishing CI validation (structural correctness) from runtime safety (Trust Layer responsibility). The new `scripts/validate-agentforce.sh` validates:
  - Bot has at least one `<botVersions>` (otherwise won't activate)
  - Plugin actions declare `<permissionSetName>` (least-privilege)
  - Hard-coded secret detection in agent description text (XML-aware)
  - Prompt template grounding hygiene (flags `{!$Input.X}` with no `{!$Resource.X}`/`{!$Apex.X}`/`{!$Flow.X}` grounding)
  - Einstein Trust Layer settings: `enableMasking`, `enableToxicityScoring`, `enablePromptInjectionDefense` not explicitly disabled
- **Behavioral testing delegated** to the dedicated `testing-agentforce` skill (`sf agent test create/run` with AI Evaluation Definitions).

### 📚 Documentation

- README now documents seven patterns, not five.
- Added "Branch Strategies" warning that GitFlow is harmful for Salesforce.
- Comparison table updated: secret management is JWT + Custom Properties or identity broker, not OIDC.
- "Common gotchas" section added to Pattern 2 (Connected App pre-authorization, JWT key format, GitHub Environments).

### 🧪 Eval suite (5 → 10 cases)

New cases:

- **Eval 6**: Delta deployments — verifies `sfdx-git-delta` usage, `fetch-depth: 0`, destructive changes handling, drift caveat.
- **Eval 7**: Unlocked packages — verifies `sfdx-project.json` structure, version-pinned deps, Dev Hub auth, decomposition cost honesty.
- **Eval 8**: Identity broker (HashiCorp Vault) — explicitly verifies the answer acknowledges Salesforce does not natively trust GitHub OIDC.
- **Eval 9**: Code coverage parsing — negative assertions block the previous broken `tr/cut` patterns.
- **Eval 10**: Scanner migration — verifies recommendation to use `sf code-analyzer` over `sfdx-scanner`.

Negative assertions added across multiple cases (e.g., "must NOT write secrets to disk", "must NOT recommend GitFlow", "must NOT use grep-based prompt injection scanning") to prevent the original bugs from recurring.

---

## [1.0.0] — 2026-05-06

Initial release.
