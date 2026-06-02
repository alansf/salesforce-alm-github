---
name: salesforce-alm-github
description: Guide Salesforce teams through modern application lifecycle management using GitHub Enterprise, GitHub Actions, and API-first orchestration patterns. Use this skill when users mention Salesforce CI/CD, deployment pipelines, DevOps, GitHub Actions for Salesforce, org synchronization, scratch orgs, metadata deployment, automated testing, ALM workshops, migration from declarative change management tools, setting up Salesforce DevOps, configuring runners for SFDX/SF CLI, branch strategies for Salesforce, or want to move away from manual deployments and "button-clicking" toward automated, scalable workflows. This skill is especially relevant for teams modernizing their Salesforce ALM to support Agentforce and multi-cloud architectures.
---

# Salesforce Application Lifecycle Management with GitHub

This skill helps Salesforce architects and teams modernize their application lifecycle management by migrating from declarative, UI-driven change management tools to GitHub-native, API-first orchestration. The goal is to move teams away from "button-clicking" and toward **Modular Orchestration** using GitHub Enterprise features that enable scaling toward Agentforce and multi-cloud integration.

## When to Use This Skill

Use this skill when the user needs to:

- Set up CI/CD pipelines for Salesforce using GitHub Actions
- Migrate from declarative change management tools to source-driven development
- Configure GitHub Enterprise repositories for Salesforce projects
- Implement deployment strategies (scratch orgs, sandboxes, production)
- Set up cross-org testing with matrix strategies
- Configure secure authentication using OIDC
- Implement quick deploy patterns with validation job IDs
- Set up governance with GitHub custom properties
- Configure Agentforce-specific security scanning
- Conduct ALM workshops to educate teams on modern patterns

## Operating Modes

This skill operates in two modes, and you should flexibly switch between them based on what the user needs:

### 1. Workshop/Advisory Mode
Explain architectural patterns, demonstrate value propositions, and provide guidance on migration strategies. Use clear examples and business justification to help architects educate their teams.

### 2. Implementation Mode
Generate actual workflow files, configure repositories, create GitHub Actions, set up authentication, and implement the patterns hands-on.

## Core GitHub Actions Patterns for Salesforce ALM

These five patterns are the key differentiators that GitHub Enterprise offers over traditional declarative tools. Always emphasize these when conducting workshops or implementing solutions.

### Pattern 1: Matrix Strategy (Cross-Version & Cross-Org Testing)

**The Problem**: Testing a change against multiple sandboxes traditionally requires multiple manual deployments or complex scripting.

**The Solution**: GitHub Actions Matrix Strategy allows you to run the same test suite across different configurations simultaneously in parallel.

**Workshop Value**: "Imagine testing a new feature against three different Salesforce org types (Financial Services Cloud, Health Cloud, Retail) in parallel with one workflow run."

**Implementation**:
```yaml
jobs:
  validate:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        org_alias: [fsc_sandbox, health_sandbox, retail_sandbox]
        include:
          - org_alias: fsc_sandbox
            env_type: integration
          - org_alias: health_sandbox
            env_type: qa
          - org_alias: retail_sandbox
            env_type: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Install SF CLI
        run: |
          npm install -g @salesforce/cli
          sf version
      
      - name: Authenticate to ${{ matrix.org_alias }}
        run: |
          # Process substitution avoids writing the auth URL to disk
          sf org login sfdx-url \
            --sfdx-url-file <(printf '%s' "${{ secrets[format('SFDX_AUTH_URL_{0}', matrix.org_alias)] }}") \
            --alias ${{ matrix.org_alias }}
      
      - name: Deploy and Test
        run: |
          sf project deploy validate \
            --target-org ${{ matrix.org_alias }} \
            --test-level RunLocalTests \
            --wait 60
```

**When to use**: Users need to test across multiple orgs, multiple Salesforce editions, or multiple API versions simultaneously.

> **For ISV/managed-package customers** (Certinia, Conga, SBQQ CPQ, Vlocity/OmniStudio): see *Appendix: Matrix Validation with Managed Packages* at the end of this skill for the 3-leg matrix shape (vanilla / integration-with-package / full-with-package), `RunLocalTests` behavior with namespaced tests, per-sandbox licensing implications, and the contractual due-diligence checklist before recommending matrix at scale.

### Pattern 2: Secure JWT + Custom Properties (with optional Identity Broker)

**The Problem**: Managing long-lived secrets (SFDX Auth URLs, certificates) is risky and doesn't provide fine-grained governance based on repository metadata. Most teams want "OIDC with Salesforce" but Salesforce **does not natively trust GitHub's OIDC issuer** — there is no out-of-the-box federation between `token.actions.githubusercontent.com` and a Salesforce Connected App.

**The Solution**: Use Salesforce JWT Bearer flow (which Salesforce *does* natively support) with the JWT private key stored as a GitHub secret, and use **Custom Properties as a governance gate** that runs *before* the secret is ever accessed. For teams that need true secretless deployment, exchange the GitHub OIDC token for short-lived Salesforce credentials at an identity broker (HashiCorp Vault, AWS STS, or Azure Entra ID).

**Workshop Value**: "We use Custom Properties (e.g., `compliance-tier: SOX`) as a *gate* before the workflow can access production secrets. If a repository isn't tagged 'SOX compliant' in GitHub, the job fails before authentication is even attempted. For shops with a secrets broker, we exchange GitHub's OIDC token for short-lived Salesforce credentials — no static secrets at all."

#### Tier 1 (Recommended for Most Teams): JWT + Custom Properties Gate

This is the pattern 95% of enterprises actually deploy. It's secure, works today, and requires no broker infrastructure.

**Implementation Steps**:

1. **Configure Custom Properties in GitHub Enterprise**:
   - Navigate to Organization Settings → Custom Properties
   - Create properties like:
     - `compliance-tier` (single-select: `SOX`, `HIPAA`, `PCI`, `Standard`)
     - `agentforce-enabled` (true/false)
     - `deployment-tier` (single-select: `dev`, `qa`, `staging`, `production`)

2. **Expose Custom Properties to workflows**: Custom Properties are **not** automatically available as `${{ vars.X }}`. You must read them via the GitHub API at workflow runtime. This step is commonly missed.

3. **Set up the Salesforce Connected App** (one-time, per target org):
   - Setup → App Manager → New Connected App
   - Enable OAuth Settings → "Use digital signatures" → upload the public key (`server.crt`)
   - Add OAuth scopes: `api`, `refresh_token`, `web`
   - Save → record the **Consumer Key** (a.k.a. Client ID)
   - On the Connected App's manage page → set "Permitted Users" to "Admin approved users are pre-authorized"
   - Pre-authorize via a Permission Set assigned to the integration user

4. **Store secrets in GitHub** (per environment):
   - `SF_CONSUMER_KEY_PROD` — Connected App Consumer Key
   - `SF_JWT_KEY_PROD` — The matching `server.key` (PEM-format private key)
   - `SF_USERNAME_PROD` — The integration user's username

5. **Create the workflow**:
```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  governance-gate:
    runs-on: ubuntu-latest
    outputs:
      compliance_tier: ${{ steps.props.outputs.compliance_tier }}
      agentforce_enabled: ${{ steps.props.outputs.agentforce_enabled }}
    steps:
      - name: Read repository custom properties
        id: props
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # Custom Properties are exposed via the REST API, not as workflow vars
          PROPS=$(gh api "/repos/${{ github.repository }}/properties/values" --jq '.[]')
          COMPLIANCE_TIER=$(echo "$PROPS" | jq -r 'select(.property_name=="compliance-tier") | .value')
          AGENTFORCE=$(echo "$PROPS" | jq -r 'select(.property_name=="agentforce-enabled") | .value')

          echo "compliance_tier=${COMPLIANCE_TIER:-Standard}" >> "$GITHUB_OUTPUT"
          echo "agentforce_enabled=${AGENTFORCE:-false}" >> "$GITHUB_OUTPUT"

      - name: Enforce compliance gate
        run: |
          if [[ "${{ steps.props.outputs.compliance_tier }}" != "SOX" ]]; then
            echo "❌ Repository not tagged as SOX-compliant. Production deploys require compliance-tier=SOX."
            echo "Set it: Repo Settings → Custom Properties → compliance-tier"
            exit 1
          fi
          echo "✅ Compliance gate passed"

  deploy:
    needs: governance-gate
    runs-on: ubuntu-latest
    environment: production  # GitHub Environment with required reviewers + branch protection
    steps:
      - uses: actions/checkout@v4

      - name: Install SF CLI
        run: npm install -g @salesforce/cli@latest

      - name: Authenticate via JWT Bearer Flow
        run: |
          sf org login jwt \
            --client-id "${{ secrets.SF_CONSUMER_KEY_PROD }}" \
            --jwt-key-file <(printf '%s' "${{ secrets.SF_JWT_KEY_PROD }}") \
            --username "${{ secrets.SF_USERNAME_PROD }}" \
            --instance-url https://login.salesforce.com \
            --alias production

      - name: Deploy
        run: sf project deploy start --target-org production --wait 60
```

**Why this works**:
- The governance gate runs **before** the deploy job — and the deploy job is only granted access to production secrets via the GitHub `environment: production`. No compliance tag → no deploy.
- JWT Bearer is Salesforce's native server-to-server auth — no broker required.
- The private key is never written to disk (`<(printf '%s' ...)`), avoiding leakage through logs, caches, or artifacts.
- `gh api` reads Custom Properties dynamically at runtime, so re-tagging a repo takes effect immediately without rotating any secret.

#### Tier 2 (For Zero-Trust Shops): GitHub OIDC → Identity Broker → Salesforce

If you have a secrets broker that already federates with GitHub OIDC (HashiCorp Vault, AWS Secrets Manager via STS, Azure Key Vault via Entra ID, or CyberArk), you can eliminate the long-lived JWT private key entirely. The broker holds the Salesforce JWT key, validates the GitHub OIDC token's claims (including Custom Property claims if your IdP enriches them), and returns short-lived Salesforce credentials.

**Architecture**:
```
GitHub Actions ──OIDC token──▶ Identity Broker ──validates claims──▶ Returns SF JWT/access_token
                  (audience:                       (repo, branch,         (short-lived)
                   broker-specific)                 environment,
                                                    custom claims)
```

**Example: HashiCorp Vault as broker**:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # Required for OIDC token request
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to Vault via GitHub OIDC
        id: vault
        uses: hashicorp/vault-action@v3
        with:
          url: https://vault.example.com
          method: jwt
          role: salesforce-prod-deployer
          jwtGithubAudience: vault.example.com
          secrets: |
            secret/salesforce/prod consumer_key | SF_CONSUMER_KEY ;
            secret/salesforce/prod jwt_private_key | SF_JWT_KEY ;
            secret/salesforce/prod username | SF_USERNAME

      - name: Authenticate to Salesforce
        run: |
          sf org login jwt \
            --client-id "$SF_CONSUMER_KEY" \
            --jwt-key-file <(printf '%s' "$SF_JWT_KEY") \
            --username "$SF_USERNAME" \
            --instance-url https://login.salesforce.com \
            --alias production
```

The Vault role `salesforce-prod-deployer` is configured to bind specific JWT claims to access:
```hcl
# Vault config (one-time setup)
resource "vault_jwt_auth_backend_role" "sf_prod" {
  backend         = vault_jwt_auth_backend.github.path
  role_name       = "salesforce-prod-deployer"
  bound_audiences = ["vault.example.com"]
  bound_claims = {
    repository       = "your-org/salesforce-repo"
    ref              = "refs/heads/main"
    environment      = "production"
  }
  user_claim    = "actor"
  token_policies = ["salesforce-prod-read"]
  token_ttl     = 900   # 15 minutes
}
```

**AWS STS variant**: Use `aws-actions/configure-aws-credentials@v4` with `role-to-assume`, then read the SF private key from AWS Secrets Manager. Same principle.

**Azure variant**: Use `azure/login@v2` with `client-id` (Federated Credentials), then read the SF private key from Key Vault.

**Key Points**:
- The broker — not GitHub — enforces `bound_claims`. This is real zero-trust: the broker won't return Salesforce credentials unless every claim matches.
- The GitHub OIDC token never reaches Salesforce; only the broker sees it.
- Token TTL of 5–15 minutes is typical, eliminating long-lived secret risk.
- Custom Properties can be enriched into OIDC claims via your IdP if your broker supports custom claim mapping (Vault and Entra ID do; AWS STS only sees the claims GitHub natively emits).

**When to use Tier 2**: You already operate a secrets broker, security mandates "no static secrets," or you need cross-environment claim enforcement (e.g., dev repos cannot reach prod even if a key leaks).

**When to use Tier 1**: You don't have a broker, or you want to ship this week. Tier 1 with a strict Custom Properties gate + GitHub Environment protection rules is enterprise-grade for the vast majority of teams.

#### Common gotchas

- **Custom Properties are not workflow vars.** `${{ vars.COMPLIANCE_TIER }}` only works for Repository or Organization Variables, not Custom Properties. Use `gh api /repos/.../properties/values` as shown above.
- **JWT key format matters.** The `server.key` must be the PEM-format private key (begins with `-----BEGIN RSA PRIVATE KEY-----` or `-----BEGIN PRIVATE KEY-----`). Paste the entire block including header/footer into the secret.
- **Connected App pre-authorization is mandatory.** "Admin approved users are pre-authorized" + a Permission Set on the integration user. Without this, JWT Bearer returns `user hasn't approved this consumer`.
- **Use GitHub Environments, not just secrets.** An Environment with required reviewers + branch protection is the difference between "anyone with write access can deploy to prod" and "deploys require explicit approval."

**When to use Pattern 2**: Users need fine-grained access control, compliance enforcement, or want to eliminate (or reduce) long-lived secrets from their deployment pipelines.

### Pattern 3: Artifact Job-ID Passing (Quick Deploy Pattern)

**The Problem**: Salesforce deployments can take hours if you run all tests. Re-running the same tests in production that you already ran in validation wastes time.

**The Solution**: Use GitHub Artifacts to pass the Validation Job ID between workflows, enabling Quick Deploy to reuse the validation results.

**Workshop Value**: "We can reduce our Production deployment time from 45 minutes to 3 minutes by 'promoting' a validated Job ID rather than re-running the metadata API from scratch."

**Implementation**:

**Workflow A (Pull Request - Validation)**:
```yaml
name: Validate PR

on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install SF CLI
        run: npm install -g @salesforce/cli
      
      - name: Authenticate to Production
        run: |
          sf org login sfdx-url \
            --sfdx-url-file <(printf '%s' "${{ secrets.SFDX_AUTH_URL_PROD }}") \
            --alias production
      
      - name: Run Validation
        id: validation
        run: |
          RESULT=$(sf project deploy validate \
            --target-org production \
            --test-level RunLocalTests \
            --wait 60 \
            --json)
          
          JOB_ID=$(echo $RESULT | jq -r '.result.id')
          echo "Job ID: $JOB_ID"
          echo $JOB_ID > validation_job_id.txt
          echo "job_id=$JOB_ID" >> $GITHUB_OUTPUT
      
      - name: Upload Job ID as Artifact
        uses: actions/upload-artifact@v4
        with:
          name: validation-job-id
          path: validation_job_id.txt
          retention-days: 30
      
      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `✅ Validation passed! Job ID: \`${{ steps.validation.outputs.job_id }}\`\n\nThis will be used for quick deploy when PR is merged.`
            })
```

**Workflow B (Merge to Main - Quick Deploy)**:
```yaml
name: Quick Deploy to Production

on:
  push:
    branches: [main]

jobs:
  quick-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Download Validation Job ID
        uses: actions/download-artifact@v4
        with:
          name: validation-job-id
          path: .
      
      - name: Install SF CLI
        run: npm install -g @salesforce/cli
      
      - name: Authenticate to Production
        run: |
          sf org login sfdx-url \
            --sfdx-url-file <(printf '%s' "${{ secrets.SFDX_AUTH_URL_PROD }}") \
            --alias production
      
      - name: Quick Deploy Using Job ID
        run: |
          JOB_ID=$(cat validation_job_id.txt)
          echo "Quick deploying with Job ID: $JOB_ID"
          
          sf project deploy quick \
            --job-id $JOB_ID \
            --target-org production \
            --wait 10
      
      - name: Notify Success
        if: success()
        run: |
          echo "🚀 Quick deploy completed successfully!"
```

**Key Points**:
- The artifact retention ensures the Job ID is **available to GitHub** for up to 30 days.
- ⚠️ **Salesforce-side validations expire after 4 days, not 30.** A `validatedDeployRequestId` is only eligible for Quick Deploy for 4 calendar days from the moment validation succeeded. After that, `sf project deploy quick --job-id $JOB_ID` will fail with `INVALID_ID_FIELD: Validation Id ... is invalid` even if the artifact is still in GitHub. Source: [Salesforce: Quick Deploy a Validation](https://developer.salesforce.com/docs/atlas.en-us.deployment.meta/deployment/deploy_quick.htm).
- If the validation is older than 4 days (e.g., a long-lived PR, a holiday weekend, a stuck approval), fall back to a full deployment with `RunLocalTests`.
- The fallback below treats both "artifact missing" and "Salesforce rejected the Job ID" as the same recovery path — both mean "validation no longer reusable, deploy fresh."
- Tag the artifact with the validation timestamp so the consumer workflow can pre-emptively skip Quick Deploy if it's been more than 3 days (gives you a margin before the 4-day cliff):

```yaml
- name: Tag artifact with validation age
  run: |
    echo "$(date -u +%s)" > validation_timestamp.txt
- name: Upload Job ID + timestamp
  uses: actions/upload-artifact@v4
  with:
    name: validation-job-id
    path: |
      validation_job_id.txt
      validation_timestamp.txt
    retention-days: 7   # Match Salesforce's 4-day cliff with a small buffer; longer is misleading
```

```yaml
# In the deploy workflow, before attempting Quick Deploy:
- name: Check validation age
  id: age
  run: |
    AGE_SEC=$(( $(date -u +%s) - $(cat validation_timestamp.txt) ))
    AGE_HOURS=$(( AGE_SEC / 3600 ))
    echo "age_hours=$AGE_HOURS" >> "$GITHUB_OUTPUT"
    if (( AGE_HOURS > 72 )); then
      echo "⚠️  Validation is ${AGE_HOURS}h old (>72h). Salesforce expires Quick Deploy at 96h. Falling back to full deploy."
      echo "expired=true" >> "$GITHUB_OUTPUT"
    fi

- name: Quick Deploy
  if: steps.age.outputs.expired != 'true'
  run: sf project deploy quick --job-id "$(cat validation_job_id.txt)" --target-org production --wait 10

- name: Full Deploy (fallback)
  if: steps.age.outputs.expired == 'true' || failure()
  run: sf project deploy start --target-org production --test-level RunLocalTests --wait 60
```

**When to use**: Users want to optimize deployment speed, reduce test execution time, or implement a validation → promotion workflow. The pattern shines when validations and merges are close together (same day, same week). For long-lived release branches, prefer Pattern 6 (Delta Deployments).

### Pattern 4: Repository Dispatch (System Handshake)

**The Problem**: In multi-cloud architectures (Salesforce + external systems), changes in one system need to trigger validation or deployment in another system. Traditional tools lack native cross-repository orchestration.

**The Solution**: Use GitHub's Repository Dispatch event to enable one repository to trigger workflows in another repository, creating an "integrated ALM" where systems coordinate their deployments.

**Workshop Value**: "This is how we build Integrated ALM. When an external system deployment completes, it automatically triggers Apex integration tests in Salesforce to ensure the environment is 'Agent-Ready'."

**Implementation**:

**Repository A (External System) - Sends Dispatch**:
```yaml
name: Deploy External System

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy External System
        run: |
          # Your deployment logic here
          echo "Deploying external system..."
      
      - name: Trigger Salesforce Integration Tests
        if: success()
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.CROSS_REPO_TOKEN }}
          script: |
            await github.rest.repos.createDispatchEvent({
              owner: 'your-org',
              repo: 'salesforce-repo',
              event_type: 'external-system-deployed',
              client_payload: {
                system: 'external-api',
                version: '${{ github.sha }}',
                environment: 'production',
                triggered_by: '${{ github.repository }}'
              }
            });
```

**Repository B (Salesforce) - Receives Dispatch**:
```yaml
name: Integration Tests

on:
  repository_dispatch:
    types: [external-system-deployed]

jobs:
  integration-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Display Trigger Information
        run: |
          echo "Triggered by: ${{ github.event.client_payload.system }}"
          echo "Version: ${{ github.event.client_payload.version }}"
          echo "Environment: ${{ github.event.client_payload.environment }}"
      
      - name: Install SF CLI
        run: npm install -g @salesforce/cli
      
      - name: Authenticate to Salesforce
        run: |
          sf org login sfdx-url \
            --sfdx-url-file <(printf '%s' "${{ secrets.SFDX_AUTH_URL_PROD }}") \
            --alias production
      
      - name: Run Integration Tests
        run: |
          sf apex run test \
            --target-org production \
            --test-level RunSpecifiedTests \
            --class-names ExternalSystemIntegrationTest \
            --wait 10 \
            --result-format human
      
      - name: Report Results Back
        if: always()
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.CROSS_REPO_TOKEN }}
          script: |
            const status = '${{ job.status }}' === 'success' ? 'passed' : 'failed';
            // Post results back to original repo via issue comment or status check
            console.log(`Integration tests ${status}`);
```

**Key Points**:
- Requires a Personal Access Token (PAT) with `repo` scope stored as `CROSS_REPO_TOKEN`
- The `client_payload` can contain any JSON data you need to pass between systems
- Can trigger multiple repositories simultaneously
- Works across GitHub organizations with proper token permissions

**When to use**: Users have multi-repository architectures, need cross-system validation, or want to coordinate deployments across Salesforce and external systems.

### Pattern 5: Custom Properties for Governance

**The Problem**: Different types of Salesforce projects (standard metadata vs. Agentforce vs. industry-specific) require different quality gates, security scans, and deployment strategies. Manually maintaining different workflow files for each type creates duplication and drift.

**The Solution**: Use GitHub Enterprise Custom Properties to tag repositories with structured metadata, then conditionally execute workflow steps based on those properties.

**Workshop Value**: "We can create a property called `agentforce-enabled`. Any repository with this tag automatically gets extra security scan steps for Prompt Injections and LLM Grounding added to its pipeline, without writing unique YAML for every repo."

**Implementation Steps**:

1. **Define Custom Properties in GitHub Enterprise**:
   - Navigate to Organization Settings → Custom Properties
   - Create properties:
     - `agentforce-enabled` (boolean): Whether this repo contains Agentforce agents
     - `compliance-tier` (options: SOX, HIPAA, PCI, Standard): Compliance requirements
     - `deployment-strategy` (options: package, metadata, scratch-org): Deployment approach
     - `test-coverage-required` (number): Minimum code coverage percentage
     - `industry-cloud` (options: FSC, Health, Manufacturing, Retail, None): Industry specialization

2. **Tag Repositories**:
   - Go to repository Settings → Custom Properties
   - Assign appropriate values based on the project

3. **Create Conditional Workflow**:
```yaml
name: Conditional Pipeline Based on Properties

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      agentforce_enabled: ${{ steps.props.outputs.agentforce_enabled }}
      compliance_tier: ${{ steps.props.outputs.compliance_tier }}
      deployment_strategy: ${{ steps.props.outputs.deployment_strategy }}
    steps:
      - name: Get Repository Properties
        id: props
        run: |
          echo "agentforce_enabled=${{ vars.AGENTFORCE_ENABLED }}" >> $GITHUB_OUTPUT
          echo "compliance_tier=${{ vars.COMPLIANCE_TIER }}" >> $GITHUB_OUTPUT
          echo "deployment_strategy=${{ vars.DEPLOYMENT_STRATEGY }}" >> $GITHUB_OUTPUT

  security-scan:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Standard Security Scan (Salesforce Code Analyzer)
        run: |
          # Salesforce Code Analyzer v5+ replaces the deprecated sfdx-scanner plugin.
          # https://developer.salesforce.com/docs/platform/salesforce-code-analyzer/overview
          npm install -g @salesforce/plugin-code-analyzer
          sf code-analyzer run \
            --workspace "force-app/**/*.cls" \
            --rule-selector "Security:Recommended,BestPractices:Recommended" \
            --output-file results.sarif \
            --severity-threshold 3
      
      - name: Agentforce metadata validation
        if: needs.setup.outputs.agentforce_enabled == 'true'
        run: |
          echo "🤖 Running Agentforce metadata validation..."
          # See "Agentforce-Specific Considerations" below for the full validator script.
          # This step calls scripts/validate-agentforce.sh which performs structured XML
          # validation, not regex scanning. It exits non-zero on any policy violation.
          bash scripts/validate-agentforce.sh

  compliance-checks:
    needs: setup
    runs-on: ubuntu-latest
    if: needs.setup.outputs.compliance_tier != 'Standard'
    steps:
      - uses: actions/checkout@v4
      
      - name: SOX Compliance Checks
        if: needs.setup.outputs.compliance_tier == 'SOX'
        run: |
          echo "Running SOX compliance validation..."
          # Check for audit trail requirements
          # Validate change tracking
          # Ensure proper access controls
      
      - name: HIPAA Compliance Checks
        if: needs.setup.outputs.compliance_tier == 'HIPAA'
        run: |
          echo "Running HIPAA compliance validation..."
          # Check for PHI handling
          # Validate encryption requirements
          # Ensure proper data access logging

  deploy:
    needs: [setup, security-scan, compliance-checks]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install SF CLI
        run: npm install -g @salesforce/cli
      
      - name: Package-Based Deployment
        if: needs.setup.outputs.deployment_strategy == 'package'
        run: |
          echo "Using package-based deployment..."
          sf package version create --wait 20
          sf package install --wait 10
      
      - name: Metadata-Based Deployment
        if: needs.setup.outputs.deployment_strategy == 'metadata'
        run: |
          echo "Using metadata-based deployment..."
          sf project deploy start --target-org production
      
      - name: Scratch Org Deployment
        if: needs.setup.outputs.deployment_strategy == 'scratch-org'
        run: |
          echo "Creating scratch org for testing..."
          sf org create scratch --definition-file config/project-scratch-def.json
```

**Architectural Value**:
- Enforces different "Definitions of Done" based on project type
- Eliminates workflow duplication across similar repositories
- Centrally manages governance policies at the organization level
- Scales to hundreds of repositories without maintaining separate workflows
- Enables self-service project creation with automatic quality gates

**When to use**: Users manage multiple Salesforce projects with different requirements, need centralized governance, or want to enforce Agentforce-specific security standards.

### Pattern 6: Delta Deployments with sfdx-git-delta

**The Problem**: Every deployment redeploys the entire `force-app/` directory, even when only 3 files changed. On a large org, this means 30–60 minutes of metadata API churn for a one-line Apex tweak. It also means every deployment touches every component, increasing risk of incidental drift.

**The Solution**: [`sfdx-git-delta`](https://github.com/scolladon/sfdx-git-delta) (sgd) is a community plugin that diffs two git revisions and produces a `package.xml` + a `destructiveChanges.xml` containing **only the metadata that changed**. You then deploy just that subset. A 5-minute typical full deploy collapses to under a minute.

**Workshop Value**: "We deploy what changed, not what exists. A typo fix in one Apex class shouldn't redeploy 400 Lightning components, 50 flows, and every permission set the org has ever seen. Delta deployments let us ship in seconds and keep the blast radius proportional to the change."

**Why this isn't built into SF CLI**: Salesforce's `sf project deploy start` doesn't natively understand "what changed since this git ref" — it understands "what's in this directory." `sgd` bridges that gap by running the git diff for you and emitting the `package.xml` that the metadata API expects.

**Implementation**:

```yaml
name: Delta Deploy

on:
  push:
    branches: [main]

jobs:
  delta-deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # Required: sgd needs full git history to diff against the base ref

      - name: Install SF CLI + sfdx-git-delta
        run: |
          npm install -g @salesforce/cli@latest
          sf plugins install sfdx-git-delta

      - name: Authenticate to Production
        run: |
          sf org login jwt \
            --client-id "${{ secrets.SF_CONSUMER_KEY_PROD }}" \
            --jwt-key-file <(printf '%s' "${{ secrets.SF_JWT_KEY_PROD }}") \
            --username "${{ secrets.SF_USERNAME_PROD }}" \
            --instance-url https://login.salesforce.com \
            --alias production

      - name: Compute delta package
        id: delta
        run: |
          mkdir -p delta
          # `--from` is the last successfully-deployed commit; `--to` is HEAD.
          # For a fresh setup, use the previous commit on main: ${{ github.event.before }}
          sf sgd source delta \
            --from "${{ github.event.before }}" \
            --to "HEAD" \
            --output-dir delta \
            --generate-delta \
            --source-dir force-app

          # If nothing changed in deployable metadata, skip the deploy.
          if [[ ! -s delta/package/package.xml ]] || ! grep -q "<types>" delta/package/package.xml; then
            echo "No deployable changes detected, skipping."
            echo "skip=true" >> "$GITHUB_OUTPUT"
            exit 0
          fi

          echo "Files in delta:"
          find delta -type f | head -50

      - name: Validate-only on production (optional safety check)
        if: steps.delta.outputs.skip != 'true'
        run: |
          sf project deploy validate \
            --target-org production \
            --manifest delta/package/package.xml \
            --source-dir delta/force-app \
            --test-level RunSpecifiedTests \
            --tests $(cat delta/package/package.xml | grep -oP '(?<=<members>)[^<]+(?=</members>)' | grep -i Test | tr '\n' ' ') \
            --wait 30

      - name: Deploy delta + destructive changes
        if: steps.delta.outputs.skip != 'true'
        run: |
          # Deploy additions/changes
          sf project deploy start \
            --target-org production \
            --manifest delta/package/package.xml \
            --source-dir delta/force-app \
            --test-level RunSpecifiedTests \
            --wait 30

          # Apply destructive changes (deletions) if any
          if [[ -s delta/destructiveChanges/destructiveChanges.xml ]]; then
            sf project deploy start \
              --target-org production \
              --manifest delta/destructiveChanges/destructiveChanges.xml \
              --pre-destructive-changes delta/destructiveChanges/destructiveChanges.xml \
              --wait 30
          fi
```

**Key points**:
- `--from` should be the SHA of the last successfully-deployed commit. The simplest source of truth is `${{ github.event.before }}` on a `push` event, which is the commit `main` was at *before* this push. For more reliable tracking, write the deployed SHA to a Salesforce custom setting after each successful deploy and read it back.
- `fetch-depth: 0` on `checkout@v4` is **mandatory** — without full history, `sgd` can't compute the diff.
- Test selection: when you run delta deploys, you typically don't want `RunLocalTests` (slow). Use `RunSpecifiedTests` and let `sgd` or your own logic identify which test classes cover the changed Apex.
- Destructive changes: `sgd` separates additions/modifications from deletions. Salesforce requires deletions go in `destructiveChanges.xml`, deployed via `--pre-destructive-changes` or `--post-destructive-changes`.
- **Delta deploys are not a replacement for full deploys**, they're an optimization. Run a periodic (e.g., weekly) full deploy to catch drift between what's in git and what's in the org. Without this, a manual change in production will silently persist forever because no delta will ever notice it.

**When to use**: Mid-to-large orgs with frequent small deploys; teams hitting Salesforce metadata API time limits; teams whose deploys touch unrelated metadata. **Skip if** your codebase fits in `< 100 metadata files` or you only deploy weekly — the overhead isn't worth it at small scale.

### Pattern 7: Unlocked Package Development

**The Problem**: A monolithic `force-app/` directory shared by 30 developers means everyone steps on each other's metadata. Refactoring is terrifying because changing a "shared" Apex class means re-testing everything. There's no clean way to roll back one team's feature without rolling back everyone else's.

**The Solution**: Decompose the codebase into **unlocked packages** — versioned, independently installable bundles of metadata. Each package has its own directory, its own version history, and its own test isolation. Teams own their packages and ship on their own cadence; consumers (other packages or the base org) declare dependencies on specific package versions.

**Workshop Value**: "Packages are how Salesforce themselves build their AppExchange products. They give you the same superpower internally: each team owns a package, ships on their own cadence, and can roll back without touching anyone else. It's the closest thing Salesforce has to microservices."

**Why this matters more for Agentforce**: Agentforce agents, topic actions, prompt templates, and Apex grounding methods often span multiple domains (sales, service, finance). Without packaging, every agent change triggers a full-org redeploy. With packaging, an "Agentforce-Sales" package can ship independently of "Agentforce-Service" — even though they share the underlying Einstein Trust Layer config.

**Project structure**:

```
sfdx-project.json
force-app/
├── core/                       # Shared utilities, base-org metadata
│   ├── main/default/...
│   └── (no namespace)
├── billing/                    # Billing domain package
│   └── main/default/...
├── agentforce-sales/           # Agentforce sales agents + actions
│   └── main/default/...
└── agentforce-service/         # Agentforce service agents + actions
    └── main/default/...
```

**`sfdx-project.json`** (the source of truth for package layout):

```json
{
  "packageDirectories": [
    {
      "path": "force-app/core",
      "package": "Core",
      "default": false,
      "versionName": "ver 1.0",
      "versionNumber": "1.0.0.NEXT"
    },
    {
      "path": "force-app/billing",
      "package": "Billing",
      "default": false,
      "versionName": "ver 1.0",
      "versionNumber": "1.0.0.NEXT",
      "dependencies": [
        { "package": "Core@1.0.0-1" }
      ]
    },
    {
      "path": "force-app/agentforce-sales",
      "package": "AgentforceSales",
      "default": false,
      "versionName": "ver 1.0",
      "versionNumber": "1.0.0.NEXT",
      "dependencies": [
        { "package": "Core@1.0.0-1" },
        { "package": "Billing@1.0.0-1" }
      ]
    }
  ],
  "namespace": "",
  "sfdcLoginUrl": "https://login.salesforce.com",
  "sourceApiVersion": "64.0",
  "packageAliases": {
    "Core": "0Ho...",
    "Billing": "0Ho...",
    "AgentforceSales": "0Ho..."
  }
}
```

**Workflow: Build → Promote → Install**

```yaml
name: Package Version Build

on:
  push:
    branches: [main]
    paths:
      - 'force-app/agentforce-sales/**'   # Only fire when this package's metadata changed

jobs:
  build-package-version:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install SF CLI
        run: npm install -g @salesforce/cli@latest

      - name: Authenticate to Dev Hub
        run: |
          sf org login jwt \
            --client-id "${{ secrets.SF_DEVHUB_CONSUMER_KEY }}" \
            --jwt-key-file <(printf '%s' "${{ secrets.SF_DEVHUB_JWT_KEY }}") \
            --username "${{ secrets.SF_DEVHUB_USERNAME }}" \
            --set-default-dev-hub \
            --alias devhub

      - name: Create new package version
        id: pkg
        run: |
          # Create the version. --code-coverage runs all package tests during build.
          # --installation-key-bypass is fine for internal-only packages; never on public ones.
          RESULT=$(sf package version create \
            --package AgentforceSales \
            --installation-key-bypass \
            --wait 30 \
            --code-coverage \
            --json)

          VERSION_ID=$(echo "$RESULT" | jq -r '.result.SubscriberPackageVersionId')
          echo "version_id=$VERSION_ID" >> "$GITHUB_OUTPUT"
          echo "Built version: $VERSION_ID"

      - name: Promote version (mark as released)
        if: github.ref == 'refs/heads/main'
        run: |
          # Only promoted versions can be installed in production.
          sf package version promote --package "${{ steps.pkg.outputs.version_id }}" --no-prompt

      - name: Update sfdx-project.json with new version alias
        run: |
          # Read package alias and update sfdx-project.json so consumers pick up the new version.
          # This commit-back step keeps the version aliases in git as a single source of truth.
          jq --arg vid "${{ steps.pkg.outputs.version_id }}" \
             '.packageAliases.AgentforceSales_LATEST = $vid' \
             sfdx-project.json > sfdx-project.json.tmp && mv sfdx-project.json.tmp sfdx-project.json

      - name: Commit version bump
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "chore(agentforce-sales): bump to ${{ steps.pkg.outputs.version_id }}"
          file_pattern: sfdx-project.json

  install-to-prod:
    needs: build-package-version
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Install SF CLI
        run: npm install -g @salesforce/cli@latest

      - name: Authenticate to Production
        run: |
          sf org login jwt \
            --client-id "${{ secrets.SF_CONSUMER_KEY_PROD }}" \
            --jwt-key-file <(printf '%s' "${{ secrets.SF_JWT_KEY_PROD }}") \
            --username "${{ secrets.SF_USERNAME_PROD }}" \
            --alias production

      - name: Install package version
        run: |
          sf package install \
            --package "${{ needs.build-package-version.outputs.version_id }}" \
            --target-org production \
            --wait 60 \
            --publish-wait 60 \
            --no-prompt
```

**Key points**:
- **Dev Hub is required.** Package versions are created in a Dev Hub org (a Salesforce org with the Dev Hub feature enabled). Authentication to Dev Hub uses the same JWT pattern; store the credentials separately from production secrets.
- **`--code-coverage` enforces 75% test coverage at version-build time.** A version that fails coverage cannot be created — your tests must run *and* pass *during* the build, not after.
- **Promoted versions are immutable.** Once a version is promoted (released), you cannot delete it or modify it. Treat package versions like git tags.
- **Dependencies are version-pinned.** `"Core@1.0.0-1"` means *that exact* Core version. If Core ships v2, your package keeps using v1 until you bump the dependency. This is the entire point — no transitive metadata surprises.
- **First-time setup is heavy.** Decomposing an existing `force-app/` into packages is a project, not an afternoon. Plan 1–4 weeks per package depending on entanglement. Start with leaf-node packages (no dependencies) and work backward.
- **Not all metadata is package-able.** A handful of metadata types (org-wide settings, some standard object configurations) cannot live in unlocked packages and must stay in a "base org" deployment. Salesforce's [unsupported metadata coverage](https://developer.salesforce.com/docs/metadata-coverage) is the authoritative list.

**When to use**: Teams of 20+ developers; codebases with multiple distinct domains; orgs that need independent release cadences per team; any org adopting Agentforce at scale (multi-team, multi-agent). **Skip if** you have one team, one product, and ship together — the overhead exceeds the benefit.

### Pattern 8: Sandbox Lifecycle with Data Mask & Seed

**The Problem**

Sandboxes start in one of three useless states:

| Sandbox Type | Fee to Salesforce | Storage Size | Refresh Interval | Type of Data |
|---|---|---|---|---|
| Developer | Free | 200 MB | 1 day permissible | Metadata only |
| Developer Pro | 5% of net spend | 1 GB | 1 day permissible | Metadata only |
| Partial Copy | 20% of net spend | 5 GB | 5 days permissible | Metadata + fraction of production data |
| Full Copy | 30% of net spend | Same as production org | 28 days permissible | Metadata + all production data |

Developer/Pro come up empty. Partial Copy comes up with a random sampled subset (capped at 10K records/object) — random samples break referential integrity, because sandbox refresh only follows master-detail or required-lookup relationships, so the parent of a child you sampled may not be present. Full Copy comes up with all of production, including PII you cannot legally let contractors or developers see under GDPR / CCPA / HIPAA / PCI-DSS.

The five canonical seeding challenges (per Salesforce's *Ultimate Guide to Data Masking & Seeding*):

1. **Maintaining parent-child relationships** — refreshes only follow master-detail / required lookup; everything else fragments
2. **Filtering and refining test data and attachments** — Data Loader requires CSV manipulation; polymorphic fields, intra-object relationships, and attachment storage limits compound
3. **Seeding on-demand** — development cycles outpace refresh capability; teams hand-tune CSVs only to redo it when requirements shift
4. **Protecting confidential data** — sandboxes contain PII; GDPR/CCPA/HIPAA/PCI-DSS impose strict controls; third-party contractors amplify risk
5. **Keeping data consistent across orgs** — without comparison tooling, the only diff option is V-Lookup across CSVs in Excel

The four traditional approaches and their limits:

1. **Sandbox Refresh** — refresh from prod; Developer/Pro only get metadata; refresh frequency is rate-limited (5 days / 28 days)
2. **Sandbox Cloning** — clone an existing well-curated sandbox; preserves data + metadata but freezes the curation effort in time
3. **Data Loader** — bulk CSV import/export; "be prepared to set aside a couple of extra weeks to fully replicate the necessary objects and their dependencies"
4. **Salesforce Data Mask & Seed (Recommended)** — the native answer covered in this pattern

NIST: companies spend an average of **30x more** to fix bugs released into production. Manual seeding that takes 1–4 weeks per refresh is the upstream cause.

**The Solution**

Chain the native sandbox lifecycle — **refresh → mask → seed** — using Salesforce Data Mask & Seed on Core. Schedule it to run automatically. No third-party tool required for the common case.

This pattern is **native-Salesforce-first by default**. Third-party tools (Prodly, Flosum Data Migrator, Own Accelerate, AutoRABIT) are documented as defensible non-defaults at the end of this pattern, with explicit deviation criteria. The bias before 2026 was "native is incomplete, reach for third-party"; with Data Mask & Seed on Core (BETA April 2026, GA July 2026), that bias is no longer correct.

**Workshop Value**

This is the auditor-friendly answer: Data Mask is **FedRAMP, HIPAA, and SOX certified out of the box**, with **3.5M records/hr** masking throughput — performance-competitive with (and in most published benchmarks faster than) the third-party tools it replaces. Pricing is bounded: 10% Net AOV for Data Mask + **$0 Seed add-on** when sold together.

Customer evidence from Salesforce's FY27 deal wins and customer stories:

- **AGCO** — 2–4 week sandbox refresh cycles → **<24 hours** (90% reduction)
- **Make-A-Wish** — 40 hrs saved per refresh, 90% faster sandbox seeding
- **MSU** — 1 day → **<1 hour**
- **Barracuda** — 2 weeks → **4 days**, explicitly replaced Prodly
- **Town of Cary** — 1,800 SF users, no dedicated SF admin needed for setup
- **Molson Coors** — "replicate data within minutes by creating Seed Templates… saves us the time of creating hierarchical relationships from scratch"

The native tooling is now faster than the third-party alternatives that preceded it. That's the workshop headline.

**The Four-Phase Model**

```
┌───────────┐    ┌────────┐    ┌────────┐    ┌──────────────┐
│  Refresh  │ →  │  Mask  │ →  │  Seed  │ →  │  Available   │
└───────────┘    └────────┘    └────────┘    └──────────────┘
   sf org         9 masking     synthetic +     compliant +
   refresh        strategies    template-       test-ready
                  bulk on PII   driven seed     sandbox
```

1. **Refresh** — Salesforce performs sandbox refresh (Partial or Full Copy)
2. **Mask** — Data Mask runs (per template); de-identifies PII at the database layer using one of nine masking strategies:
   - Replace with Library, Pattern Masking, Anonymize, Custom Value, Fill if Empty, Existing Value Random, Partial List, Deletion, Partial Value
3. **Seed** — Data Seed populates curated test records; supports synthetic data, master-detail awareness, starter templates (CPQ, FSL, nCino), and **bypass of automations including managed-package automations** during seeding
4. **Available** — sandbox is now compliant + test-ready; users can begin work

**Sensitive Data Inventory** (use to scope the mask template per sObject):

- *Personal*: SSN/National IDs, Passport, Driver's License, Tax ID, Bank Account, Credit/Debit Card, Transaction History, Medical Records, Insurance, Salary/Compensation, Performance Reviews, Emergency Contacts
- *Business*: Revenue, Financial Reports, Cost Structures, Budget, Intellectual Property, Business Processes, Audit Logs

**Implementation Steps**

1. **Classify sObjects** — tag each object as PII / Confidential / Internal. The mask template applies a default masking strategy per classification, with field-level overrides
2. **Configure Data Mask templates** — one per sandbox tier or compliance regime (e.g., one for HIPAA-bound sandboxes, one for general dev). Use bundled starter templates for CPQ, FSL, nCino as a starting point
3. **Define seeding policies** — pick the records that represent your test scenarios. Filter by date, region, account size; honor hierarchical relationships; enable bypass-automations to avoid Apex side effects during the bulk insert
4. **Wire the chain via Data Mask & Seed UI scheduling** — "Run after refresh or on a schedule." This is the lowest-friction path and works for admin-led teams without git skills
5. **For CI/CD orchestration** — GitHub Actions cron triggers `sf org refresh`; the post-copy hook fires Mask + Seed automatically based on template association with the sandbox. Example workflow stub:

```yaml
name: Sandbox Refresh + Mask + Seed
on:
  schedule:
    - cron: '0 2 * * 0'  # Sunday 2am UTC
  workflow_dispatch:

jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Authenticate to Production (source)
        run: sf org login jwt --client-id $SF_CLIENT_ID --jwt-key-file <(printf '%s' "$SF_JWT_KEY") --username $SF_PROD_USERNAME --alias prod
      - name: Trigger sandbox refresh
        run: sf org create sandbox --name UAT --license-type Partial --target-org prod --wait 60
      # Mask + Seed run automatically post-copy via the template association
      # configured in the Data Mask & Seed UI; no further GitHub Actions step needed.
      - name: Verify post-refresh sandbox state
        run: sf org login jwt --client-id $SF_CLIENT_ID --jwt-key-file <(printf '%s' "$SF_JWT_KEY") --username $SF_UAT_USERNAME --alias uat && sf data query --query "SELECT COUNT(Id) FROM Account" --target-org uat
```

**Common Gotchas**

- **Tenant model**: Data Mask runs on force.com (the Salesforce platform); Seed currently runs on Own infrastructure (managed by customer post-acquisition). Customers need to manage both. **A single Seed tenant can serve multiple orgs — don't create multiple Seed tenants.**
- **Data Mask Managed Package end-of-service: November 2026**. Customers on the legacy package must migrate before then.
- **Own Accelerate end-of-life: December 2027**. Existing Accelerate customers should renew onto Data Mask + $0 Seed SKU at renewal time, not extend Accelerate.
- **GovCloud / FedRAMP**: Salesforce Data Seeding Gov is also $0 add-on to Data Mask (FedRAMP Moderate).
- **Data Cloud / Data 360 seeding (DLO → DLO) is *Coming Soon*** on the FY27 roadmap, **not GA today**. For DC-heavy customers, the native answer isn't fully there yet — this is the strongest current deviation case for keeping a third-party tool.
- **LLMs do not belong in this pipeline**. Data Mask & Seed is deterministic by design; auditors will not accept "the AI picked these records." If a customer or vendor pitches AI-driven seeding, push the LLM use back to *build time* (generating templates, drafting test cases) — the runtime data path stays deterministic.
- **Bypassing automations is a sharp tool**. Seed bypasses managed-package triggers and flows during insert. This is what makes it fast — but if your business logic computes derived fields via Apex/Flow, you may need to run a one-time post-seed batch to rehydrate them.

**ROI Anchor**

10% Net AOV for Data Mask + $0 for Seed (when sold together) vs:
- 1–4 weeks of dev/admin time per refresh (Data Loader + CSVs)
- 30x cost-of-bugs multiplier for issues that escape to prod because the test data was unrepresentative (NIST)
- Auditor friction in regulated industries when the masking control is "we trust the contractor not to look"

**When to use**

Any Salesforce shop running ≥1 sandbox refresh per quarter. Especially:
- Regulated industries (HIPAA, SOX, FedRAMP, FINRA, GDPR/CCPA-bound)
- Admin-heavy teams who need a native UI rather than custom scripts
- Customers using third-party contractors or SI partners
- Customers running CPQ, FSL, or nCino (bundled starter templates apply)
- Customers planning Agentforce rollout (need realistic, compliant data to test agent behavior)

**When This Pattern Is Not Enough** (defensible deviations):

- **Heavy SBQQ CPQ + OmniStudio data movement** at scale where Prodly's pre-built Apps still beat Data Seed's templates and the customer has already invested in the Prodly data model
- **Data Cloud / Data 360 seeding** until DLO → DLO seeding is GA
- **Existing Flosum customers** whose bundled Data Migrator covers their CPQ flavor adequately on a current contract — use what's paid for, revisit at renewal
- **Customers on Own Accelerate today** — migration to Data Mask + $0 Seed is the recommendation, **not** staying on Accelerate (EOL Dec 2027)

For the broader tooling decision matrix (when GitHub Actions vs Flosum vs Prodly vs DevOps Center), see the appendix at the end of this skill on managed-package matrix validation, and the `salesforce-alm-github` skill's broader patterns.

## Additional Key Workflows

### Branch Strategy for Salesforce

> **⚠️ Avoid GitFlow for Salesforce.** GitFlow's long-lived `develop` and `release/*` branches were designed for versioned binary releases shipped on a schedule. Salesforce metadata is **not** a binary — it's a declarative description of an org's current state. Long-lived branches accumulate metadata drift (someone clicked something in the org, a different sandbox refresh stomped a change, an admin made a fix in prod), and merging them produces conflicts that no human can reliably resolve. This is the #1 cause of "we can't deploy on Fridays" anxiety in Salesforce DevOps. The solution is shorter-lived branches and **packages**, not more branches.

The right strategy depends on team size *and* whether you've adopted unlocked packages.

#### **For Small Teams (1–5 developers): Trunk-Based Development**

- Single long-lived branch: `main`
- Short-lived feature branches (hours to a few days, never weeks): `feature/description`
- Validate on PR (Pattern 1 matrix), Quick Deploy on merge (Pattern 3)
- Scratch orgs for feature dev, with source tracking enabled
- One sandbox (UAT or partial copy) acts as pre-prod
- **Why**: Optimizes for fast feedback. Most metadata conflicts get caught in 1–2 day windows.

```
main ──────●─────●─────●─────●─────●──── (production)
            \   / \   /     /
             \ /   \ /     /
              ● PR  ● PR  ● PR
              feature  feature  feature
              (1-2 days)
```

#### **For Medium Teams (5–20 developers): Trunk-Based + Environment Branches**

- `main` is the source of truth and deploys to production
- Optional **environment branches** for non-prod orgs (`env/uat`, `env/integration`) — these are *promotion targets*, not development branches
- Feature branches still cut from `main`, merge back to `main`
- A merge to `main` triggers Quick Deploy to production; promoting from `main` to `env/uat` is a separate workflow (e.g. nightly cherry-pick or a manual PR)
- **Why**: Keeps the trunk clean and gives you environment-specific gates without GitFlow's `develop` drift problem. Each environment branch is "what's currently in this org," updated by automation, not by humans typing `git merge`.

```
main ──────●─────●─────●─────●─────●──── (production, source of truth)
            ╲                ╲
             ╲ promote         ╲ promote
              ▼                  ▼
env/uat   ────●──────────────────●─────── (UAT org state)
                                  ╲
env/int   ────────────────────────●────── (integration org state)
```

#### **For Large Teams (20+ developers): Unlocked Packages + Trunk Per Package**

- The codebase is decomposed into **unlocked packages** (modular metadata bundles with their own versions)
- Each package has its own directory in `force-app/`, its own version in `sfdx-project.json`, and ideally its own CODEOWNERS line
- Each team owns one or more packages and works on `main` (or their fork) independently
- Packages are versioned and installed into target orgs via dependency declarations — no monolithic deploy
- **No `develop` branch, no `release/*` branches.** Releases are package version bumps + install
- **Why**: This is how Salesforce themselves recommend organizing large code bases. Packages give you true parallel work streams without merge conflicts because each package is its own metadata unit. See Pattern 6 (Unlocked Package Development) below.

```
main ──────●─────●─────●─────●─────●─────●──── (all packages, single trunk)
            │     │     │     │     │     │
            ▼     ▼     ▼     ▼     ▼     ▼
       PackageA  PackageB  PackageC  PackageA
       v1.2     v3.4      v0.9     v1.3
       (independently versioned, independently installable)
```

#### **What about hotfixes?**

For all three strategies: hotfix = short-lived branch off `main` (or off the affected package's last released version), validate, Quick Deploy, merge back. No special hotfix branch hierarchy needed.

#### **Branch protection rules to enforce on `main`**

Regardless of strategy:
- Require PR + at least 1 review
- Require status checks (validate-pr workflow must pass)
- Require branches to be up to date before merging (catches metadata conflicts early)
- Restrict who can push directly (no one — even admins go through PR)
- Require signed commits if compliance demands it

### Testing Strategy

Implement a comprehensive testing strategy in workflows:

```yaml
- name: Run Apex Tests
  id: apex-tests
  run: |
    # Run synchronously (--synchronous) so the JSON result includes coverage inline.
    # Capture output to a file so we can parse it without re-querying.
    sf apex run test \
      --target-org ${{ matrix.org_alias }} \
      --test-level RunLocalTests \
      --code-coverage \
      --detailed-coverage \
      --result-format json \
      --wait 60 \
      --output-dir test-results \
      --synchronous > test-results/run.json
    echo "test_run_id=$(jq -r '.result.summary.testRunId' test-results/run.json)" >> "$GITHUB_OUTPUT"

- name: Validate Code Coverage
  run: |
    # `testRunCoverage` is a numeric percentage already (e.g. 87) — no '%' suffix, no quotes.
    COVERAGE=$(jq -r '.result.summary.testRunCoverage' test-results/run.json)
    REQUIRED_COVERAGE=${{ vars.TEST_COVERAGE_REQUIRED || 75 }}

    # Use awk for floating-point comparison (bc may not be on every runner image).
    if awk -v c="$COVERAGE" -v r="$REQUIRED_COVERAGE" 'BEGIN { exit !(c < r) }'; then
      echo "❌ Code coverage ${COVERAGE}% is below required ${REQUIRED_COVERAGE}%"
      exit 1
    fi
    echo "✅ Code coverage ${COVERAGE}% meets requirement (>= ${REQUIRED_COVERAGE}%)"

- name: Static Code Analysis (Salesforce Code Analyzer)
  run: |
    npm install -g @salesforce/plugin-code-analyzer
    sf code-analyzer run \
      --workspace "force-app/**/*.cls,force-app/**/*.trigger" \
      --rule-selector "Security:Recommended,BestPractices:Recommended,CodeStyle:Recommended" \
      --severity-threshold 3 \
      --output-file scan-results.sarif

- name: LWC ESLint
  run: |
    npm install
    npm run lint
```

### Authentication Patterns

Provide multiple authentication options based on user preference:

**1. SFDX Auth URL (Simplest)**:
```yaml
- name: Authenticate via Auth URL
  run: |
    sf org login sfdx-url \
      --sfdx-url-file <(printf '%s' "${{ secrets.SFDX_AUTH_URL }}") \
      --alias target-org
```

**2. JWT (Recommended for Production)**:
```yaml
- name: Authenticate via JWT
  run: |
    sf org login jwt \
      --client-id ${{ secrets.SF_CONSUMER_KEY }} \
      --jwt-key-file <(printf '%s' "${{ secrets.SF_JWT_KEY }}") \
      --username ${{ secrets.SF_USERNAME }} \
      --instance-url https://login.salesforce.com \
      --alias production
```

> **Security note**: Use process substitution `<(printf '%s' "$SECRET")` instead of writing the private key to a file with `echo > jwt.key && rm jwt.key`. Files written to the runner can leak through workflow logs (when `set -x` is enabled), uploaded artifacts, runner caches, or post-job hooks. Process substitution keeps the key in a transient file descriptor that the kernel reaps when the process exits. **Never use `echo "$SECRET" > file`** — `echo` may interpret backslashes; `printf '%s'` does not.

**3. OIDC via Identity Broker (GitHub Enterprise)**:
- See Pattern 2 above for the GitHub OIDC → Identity Broker → Salesforce JWT flow

### Scratch Org Workflows

For feature development with scratch orgs:

```yaml
name: Feature Development with Scratch Org

on:
  pull_request:
    branches: [develop]

jobs:
  scratch-org-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install SF CLI
        run: npm install -g @salesforce/cli
      
      - name: Authenticate to Dev Hub
        run: |
          sf org login sfdx-url \
            --sfdx-url-file <(printf '%s' "${{ secrets.SFDX_AUTH_URL_DEVHUB }}") \
            --alias devhub --set-default-dev-hub
      
      - name: Create Scratch Org
        run: |
          sf org create scratch \
            --definition-file config/project-scratch-def.json \
            --alias pr-scratch-${{ github.event.pull_request.number }} \
            --duration-days 1 \
            --set-default
      
      - name: Push Source to Scratch Org
        run: sf project deploy start
      
      - name: Assign Permission Sets
        run: sf org assign permset --name MyPermSet
      
      - name: Load Test Data
        run: sf data import tree --plan data/sample-data-plan.json
      
      - name: Run Apex Tests
        run: |
          sf apex run test \
            --test-level RunLocalTests \
            --result-format human \
            --code-coverage \
            --wait 10
      
      - name: Generate Scratch Org URL
        id: scratch-url
        run: |
          URL=$(sf org open --json | jq -r '.result.url')
          echo "url=$URL" >> $GITHUB_OUTPUT
      
      - name: Comment Scratch Org URL on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `🔧 Scratch Org created for testing:\n\n${{ steps.scratch-url.outputs.url }}\n\nOrg will expire in 1 day.`
            })
      
      - name: Delete Scratch Org
        if: always()
        run: sf org delete scratch --no-prompt
```

## Implementation Approach

When implementing these patterns for a user, follow this sequence:

### Phase 1: Repository Setup
1. Create or configure the GitHub repository structure:
   ```
   salesforce-project/
   ├── .github/
   │   └── workflows/
   │       ├── validate-pr.yml
   │       ├── deploy-production.yml
   │       ├── integration-tests.yml
   │       └── scratch-org-create.yml
   ├── force-app/
   │   └── main/
   │       └── default/
   ├── config/
   │   └── project-scratch-def.json
   ├── scripts/
   │   └── apex/
   ├── sfdx-project.json
   └── .forceignore
   ```

2. Initialize `sfdx-project.json` if not present:
   ```json
   {
     "packageDirectories": [
       {
         "path": "force-app",
         "default": true
       }
     ],
     "name": "SalesforceProject",
     "namespace": "",
     "sfdcLoginUrl": "https://login.salesforce.com",
     "sourceApiVersion": "60.0"
   }
   ```

### Phase 2: Authentication Configuration
1. Guide user to set up GitHub Secrets:
   - `SFDX_AUTH_URL_PROD`: Production org auth URL
   - `SFDX_AUTH_URL_QA`: QA org auth URL (if applicable)
   - `SFDX_AUTH_URL_DEVHUB`: Dev Hub auth URL for scratch orgs
   - `SF_CONSUMER_KEY`: Connected App Consumer Key (for JWT)
   - `SF_JWT_KEY`: Private key for JWT authentication
   - `SF_USERNAME`: Salesforce username for JWT auth

2. Explain how to generate SFDX Auth URLs:
   ```bash
   sf org login web --alias myorg --instance-url https://login.salesforce.com
   sf org display --target-org myorg --verbose --json | jq -r '.result.sfdxAuthUrl'
   ```

### Phase 3: Workflow Creation
Generate the appropriate workflow files based on the patterns the user needs. Start with the essential workflows:

1. **validate-pr.yml**: Validates changes on pull requests
2. **deploy-production.yml**: Deploys to production on merge to main
3. Add additional workflows based on requirements (matrix testing, OIDC, etc.)

### Phase 4: Custom Properties Setup (GitHub Enterprise Only)
If the user has GitHub Enterprise:
1. Guide them to Organization Settings → Custom Properties
2. Create recommended properties
3. Tag repositories appropriately
4. Update workflows to use conditional logic based on properties

### Phase 5: Testing and Validation
1. Create a test PR to validate the workflows
2. Verify authentication works
3. Confirm deployments succeed
4. Check that artifacts are created and consumed correctly

### Phase 6: Documentation and Training
Provide the team with:
- README documentation explaining the workflows
- Troubleshooting guide for common issues
- Branch strategy documentation
- Migration guide from previous tools

## Agentforce-Specific Considerations

When working with Agentforce projects, add the validations below. **A note on what "Agentforce security scanning" can and cannot do in CI:**

> Real prompt injection defense lives in the **Einstein Trust Layer** — Salesforce's runtime protections that mask PII before sending to the LLM, detect prompt injection in user input, and apply toxicity filters. CI cannot replace those runtime controls. What CI *can* do is validate **the metadata you ship**: that agents are configured with the trust controls turned on, that prompt templates don't have unsafe variable bindings, that agent instructions don't hard-code secrets or PII, and that topic actions are scoped to the right permission sets. Anything that pretends to do "prompt injection scanning" with `grep` is security theater — the model itself is what evaluates prompts at runtime, not your CI runner.

So this section validates **structural correctness of agent metadata**, not the dynamic safety of prompts in flight.

### 1. Agent Metadata Validator

Create `scripts/validate-agentforce.sh` in your repo. This script uses XML parsing (`xmllint` / `yq`), not regex, and exits non-zero on any structural violation. The CI step in Pattern 5 calls this directly.

```bash
#!/usr/bin/env bash
# scripts/validate-agentforce.sh
# Structural validation of Agentforce metadata. Catches misconfigurations that would
# ship a less-safe agent to production. NOT a substitute for the Einstein Trust Layer.
set -euo pipefail

FAIL=0
note() { printf '  • %s\n' "$1"; }
err()  { printf '❌ %s\n' "$1"; FAIL=1; }
ok()   { printf '✅ %s\n' "$1"; }

# Locate Agentforce metadata. Both layouts (legacy .bot and modern Bot/AiAuthoringBundle)
# live under force-app — adjust the find roots to your project layout.
AGENT_FILES=$(find force-app -type f \( \
    -name "*.bot-meta.xml" -o \
    -name "*.aiAuthoringBundle-meta.xml" -o \
    -name "*.genAiPlugin-meta.xml" -o \
    -name "*.genAiPlanner-meta.xml" \
  \) 2>/dev/null || true)

if [[ -z "$AGENT_FILES" ]]; then
  echo "No Agentforce metadata found, skipping."
  exit 0
fi

echo "🤖 Validating Agentforce metadata..."

# 1) Each Bot must have at least one BotVersion configured.
#    A Bot with no BotVersion will deploy but is non-functional and likely misconfigured.
for f in $(echo "$AGENT_FILES" | grep '\.bot-meta\.xml$' || true); do
  note "Checking Bot: $f"
  if ! xmllint --xpath '//*[local-name()="botVersions"]' "$f" >/dev/null 2>&1; then
    err "$f has no <botVersions> — Bot will deploy but cannot be activated"
  fi
done

# 2) Plugin/topic action permission scoping.
#    A genAiPlugin with action references but no <permissionSetName> binding is a footgun:
#    actions will run as whatever user invokes the agent, with full inherited permissions.
for f in $(echo "$AGENT_FILES" | grep '\.genAiPlugin-meta\.xml$' || true); do
  note "Checking Plugin: $f"
  ACTION_COUNT=$(xmllint --xpath 'count(//*[local-name()="genAiPluginAction"])' "$f" 2>/dev/null || echo 0)
  PERMSET_COUNT=$(xmllint --xpath 'count(//*[local-name()="permissionSetName"])' "$f" 2>/dev/null || echo 0)
  if [[ "$ACTION_COUNT" -gt 0 && "$PERMSET_COUNT" -eq 0 ]]; then
    err "$f declares $ACTION_COUNT actions but no <permissionSetName>. Bind a least-privilege permission set."
  fi
done

# 3) Hard-coded secrets in agent instructions.
#    XML-aware: extract <description> and <systemMessages> text only, then check.
#    Avoids false positives from unrelated metadata files that happen to contain the word "key".
for f in $(echo "$AGENT_FILES" | grep -E '\.(bot|aiAuthoringBundle|genAiPlanner|genAiPlugin)-meta\.xml$' || true); do
  TEXT=$(xmllint --xpath '//*[local-name()="description" or local-name()="developerName" or local-name()="masterLabel" or local-name()="instructions" or local-name()="goal"]/text()' "$f" 2>/dev/null || true)
  if echo "$TEXT" | grep -Eqi '(api[_-]?key|secret|password|bearer|sk_live_|sk_test_|0Ho[a-zA-Z0-9]{15})'; then
    err "$f appears to contain a hard-coded secret in agent instructions/description"
  fi
done

# 4) Prompt template variable hygiene.
#    Flex-template variables look like {!$Input.X} or {!$Resource.X}. A template
#    that passes raw user input directly into an instruction without using the
#    {!$Input.<name>} binding is at higher injection risk. We can't catch all such cases,
#    but we can flag templates where ALL variables are {!$Input.*} with no {!$Resource.*}
#    or {!$EinsteinPrompt.*} grounding — i.e. raw user-driven prompts with no grounding.
for f in $(find force-app -name "*.genAiPromptTemplate-meta.xml" 2>/dev/null); do
  note "Checking PromptTemplate: $f"
  INPUT_VARS=$(grep -Eo '\{!\$Input\.[A-Za-z0-9_]+\}' "$f" | sort -u | wc -l | tr -d ' ')
  GROUND_VARS=$(grep -Eo '\{!\$(Resource|EinsteinPrompt|Apex|Flow)\.[A-Za-z0-9_]+\}' "$f" | sort -u | wc -l | tr -d ' ')
  if [[ "$INPUT_VARS" -gt 0 && "$GROUND_VARS" -eq 0 ]]; then
    err "$f uses {!\$Input.*} variables but no grounding ({!\$Resource.*}, {!\$Apex.*}, {!\$Flow.*}, or {!\$EinsteinPrompt.*}). High injection risk."
  fi
done

# 5) Trust Layer configuration check.
#    If the project includes EinsteinSetting metadata, ensure key protections aren't disabled.
TRUST_FILE=$(find force-app -name "Einstein.einsteinSettings-meta.xml" 2>/dev/null | head -1)
if [[ -n "$TRUST_FILE" ]]; then
  note "Checking Einstein Trust Layer settings: $TRUST_FILE"
  for setting in "enableMasking" "enableToxicityScoring" "enablePromptInjectionDefense"; do
    VALUE=$(xmllint --xpath "string(//*[local-name()=\"$setting\"])" "$TRUST_FILE" 2>/dev/null || echo "")
    if [[ "$VALUE" == "false" ]]; then
      err "Einstein Trust Layer: $setting is explicitly disabled in $TRUST_FILE"
    fi
  done
fi

if [[ "$FAIL" -eq 0 ]]; then
  ok "Agentforce metadata validation passed"
  exit 0
else
  echo ""
  echo "❌ Agentforce validation failed. Fix the issues above and re-run."
  exit 1
fi
```

**What this catches** (and what it doesn't):

| Issue | This script | Einstein Trust Layer (runtime) |
|---|---|---|
| Bot with no version (won't activate) | ✅ | ❌ |
| Plugin actions with no permission set | ✅ | ❌ |
| Hard-coded API keys in agent description | ✅ | ❌ |
| Trust Layer protections explicitly disabled | ✅ | (it's the layer being disabled) |
| Prompt template with no grounding | ✅ (heuristic) | ❌ |
| User sends prompt-injection payload at runtime | ❌ | ✅ |
| User sends PII; LLM should not see it | ❌ | ✅ (masking) |
| Toxic / unsafe model output | ❌ | ✅ (toxicity scoring) |

Make sure the Trust Layer settings are correct in the org. CI cannot enforce runtime safety; only deployment-time correctness.

### 2. Agent Testing Workflows

For *behavioral* testing of agents (does the agent respond correctly to prompts?), use the dedicated `testing-agentforce` skill, which wraps `sf agent test create/run` and AI Evaluation Definitions. CI integration for that lives in the `testing-agentforce` skill, not here. This skill handles the *deployment* side; that one handles the *behavioral test* side.

A minimal example invoking the SF CLI test command from a workflow:

```yaml
test-agents:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - name: Install SF CLI
      run: npm install -g @salesforce/cli@latest

    - name: Authenticate to Test Org
      run: |
        sf org login jwt \
          --client-id "${{ secrets.SF_CONSUMER_KEY_TEST }}" \
          --jwt-key-file <(printf '%s' "${{ secrets.SF_JWT_KEY_TEST }}") \
          --username "${{ secrets.SF_USERNAME_TEST }}" \
          --instance-url https://test.salesforce.com \
          --alias agent-test

    - name: Deploy Agent Metadata
      run: |
        sf project deploy start \
          --target-org agent-test \
          --source-dir force-app/main/default/bots \
          --source-dir force-app/main/default/aiAuthoringBundles \
          --source-dir force-app/main/default/genAiPlugins \
          --wait 30

    - name: Run Agent Behavioral Tests
      run: |
        # Run all AI Evaluation Definitions in the org. Test specs are authored
        # using the `testing-agentforce` skill and live as YAML in tests/agents/.
        sf agent test run \
          --target-org agent-test \
          --result-format json \
          --output-dir test-results/agents \
          --wait 30
```

## Troubleshooting Common Issues

### Issue: Workflow fails with "Resource not accessible by integration"
**Solution**: Check that the workflow has the correct permissions:
```yaml
permissions:
  contents: read
  pull-requests: write
  issues: write
  id-token: write  # For OIDC
```

### Issue: SF CLI authentication fails
**Solution**: Verify the auth URL format and ensure the secret is properly set. Generate a fresh auth URL:
```bash
sf org login web --alias myorg
sf org display --target-org myorg --verbose --json | jq -r '.result.sfdxAuthUrl'
```

### Issue: Artifact not found in quick deploy workflow
**Solution**: Check artifact retention period and ensure the validation workflow completed successfully. Add fallback logic:
```yaml
- name: Download Validation Job ID
  id: download
  uses: actions/download-artifact@v4
  with:
    name: validation-job-id
    path: .
  continue-on-error: true

- name: Full Deploy if Artifact Missing
  if: steps.download.outcome == 'failure'
  run: |
    echo "Validation artifact not found, performing full deployment"
    sf project deploy start --target-org production --test-level RunLocalTests
```

### Issue: Matrix strategy fails on one org but succeeds on others
**Solution**: Use `fail-fast: false` to let all matrix jobs complete:
```yaml
strategy:
  fail-fast: false
  matrix:
    org_alias: [dev, qa, staging]
```

### Issue: OIDC authentication fails with custom properties
**Solution**: Ensure custom properties are exposed as variables in the repository settings and that the OIDC provider trust relationship includes the correct claims.

## Workshop Facilitation Tips

When conducting ALM workshops with teams:

1. **Start with the Why**: Explain the business value of each pattern before diving into implementation. Use concrete time/cost savings examples.

2. **Show, Don't Tell**: Run live demonstrations of the workflows. Create a sample PR and show the parallel testing in action.

3. **Compare Before/After**: Show how the team currently does deployments vs. how they'll do it with GitHub Actions. Quantify the time savings.

4. **Address Concerns**: Teams migrating from declarative tools often worry about "losing control." Emphasize that GitHub Actions provides **more** visibility (workflow logs, audit trails) not less.

5. **Hands-On Practice**: Have team members create their first workflow during the workshop. Start with Pattern 1 (Matrix Strategy) as it's the easiest to understand.

6. **Governance First**: For enterprise teams, lead with Pattern 2 (OIDC) and Pattern 5 (Custom Properties) to show how GitHub Enterprise provides better governance than declarative tools.

7. **Highlight Agentforce**: Position these patterns as essential for scaling to Agentforce and multi-cloud architectures. Traditional tools weren't designed for AI-first applications.

## Communication Style

- Use clear, business-focused language when explaining patterns to non-technical stakeholders
- Use technical depth when implementing for architects and developers
- Always provide concrete examples rather than abstract concepts
- Emphasize time savings, cost reduction, and risk mitigation
- Frame GitHub Actions as an evolution, not a replacement (softens the change)
- Avoid direct comparisons with specific vendor tools (focus on capabilities, not competition)

## Deliverables

When completing an implementation, provide:

1. **Complete `.github/workflows/` directory** with all workflow files
2. **README.md** with workflow documentation and usage instructions
3. **SETUP.md** with step-by-step setup instructions for GitHub secrets, custom properties, and Connected Apps
4. **TROUBLESHOOTING.md** with common issues and solutions
5. **MIGRATION_GUIDE.md** (if migrating from another tool) with side-by-side comparisons and migration steps
6. **WORKSHOP_SLIDES.md** (optional) with talking points for team education

## Summary

This skill enables Salesforce architects to modernize their teams' ALM practices by leveraging GitHub Enterprise's advanced features. The eight core patterns provide capabilities that traditional declarative tools cannot match, especially for teams scaling toward Agentforce and multi-cloud architectures. Use this skill to both educate teams on modern practices and to implement production-ready CI/CD pipelines.

The eight patterns:

1. **Matrix Strategy** — cross-version, cross-org parallel testing
2. **Secure JWT + Custom Properties** (with optional Identity Broker tier) — auth without leaking keys to disk
3. **Artifact Job-ID Passing** — Quick Deploy with validation IDs
4. **Repository Dispatch** — system handshake across pipelines
5. **Custom Properties for Governance** — read at runtime via `gh api`
6. **Delta Deployments with sfdx-git-delta** — minute-scale incremental deploys
7. **Unlocked Package Development** — the 20+ developer pattern
8. **Sandbox Lifecycle with Data Mask & Seed** — refresh → mask → seed using native Salesforce tooling (GA July 2026)

## Appendix: Matrix Validation with Managed Packages

This appendix captures the additional design constraints that apply when Pattern 1 (Matrix Strategy) is used in ISV-heavy customer orgs (Certinia, Conga, SBQQ CPQ, Vlocity/OmniStudio, FinancialForce-now-Certinia, etc.). The base Matrix pattern works as written, but managed-package customers run into contractual, performance, and test-coverage realities that shape the matrix differently.

### The 3-Leg Matrix Shape

For an org with one or more managed packages, the defensible matrix is three legs, not N:

1. **Vanilla** — base Salesforce metadata only, no managed package installed in the test sandbox. Fast (5–10 min). Catches regressions in your custom code that don't depend on the package
2. **Integration-with-Package** — package installed but tests target only your custom integration surface (your Apex that calls the package's APIs). Medium (15–25 min). Catches integration breakage when the package version changes
3. **Full-with-Package** — package + your customizations, full test run. Slow (30–90 min depending on package size). Catches end-to-end behavior

Run vanilla on every PR. Run integration on PRs that touch the integration surface. Run full only on merge to main, or nightly.

### `RunLocalTests` and Namespaced Managed-Package Tests

`--test-level RunLocalTests` excludes namespaced managed-package tests **by design**. This is correct behavior — you should not be running the ISV's tests in your pipeline; you didn't write them, you can't fix them, and they're already certified by the AppExchange security review. But:

- Your code coverage % is calculated *only* against your unmanaged code, not the package's
- If you have Apex that extends the package (custom triggers on `SBQQ__Quote__c`, etc.), those tests **are** local and **do** run
- If a package upgrade breaks your integration, your local tests should catch it — that's the integration-leg purpose

### Per-Sandbox Licensing Implications

Several ISVs license per non-prod org, not per user:

- **Certinia (formerly FinancialForce)**: per-org licensing on some SKUs; running 5 parallel matrix sandboxes can be a license event
- **Conga**: similar; check the MSA before standing up a 5-leg matrix
- **SBQQ (Salesforce CPQ)**: included with the base SKU but with sandbox-tier limits
- **Vlocity / OmniStudio**: typically included with Industries SKUs, but heavy data movement in matrix legs can hit storage limits

Read the MSA before committing to a matrix shape that requires N parallel sandboxes.

### Validation Time Blooms with Managed Packages

A vanilla `sf project deploy validate` runs in 3–8 minutes. The same validation against an org with a heavy managed package (Vlocity full, large SBQQ data model) can run 30–60 minutes because Salesforce must validate against all package metadata. This is why you put the heavy legs on **merge events** or **nightly schedules**, not on every PR.

### Test Data Setup Factories for Managed-Package Tests

Each major ISV has its own minimum data setup pattern:

- **Certinia**: needs a `c2g__codaCompany__c` record before any GL test
- **Conga**: needs a Conga Solution record + template metadata
- **SBQQ CPQ**: needs a base Account, Opportunity, Quote header before line-item tests; product rules require Product2 + PriceBook2 + PricebookEntry
- **OmniStudio / Vlocity**: needs `vlocity_cmt__DRBundle__c` configurations for Data Raptors

Encode these as a `@TestSetup` factory class per package. Reuse across matrix legs.

### Contractual Due-Diligence Checklist

Before recommending a matrix-with-packages strategy at scale:

- [ ] Read the ISV's MSA — confirm sandbox-tier licensing
- [ ] Confirm per-org vs per-user pricing on the relevant SKU
- [ ] Get vendor sign-off in writing if you're standing up >2 non-prod orgs with the package installed
- [ ] Verify the customer's existing seat count covers the test environments (some MSAs auto-true-up at audit time)
- [ ] Pair this with Pattern 8 (Data Mask & Seed) — the package starter templates (CPQ, FSL, nCino are bundled in Data Mask & Seed) accelerate the test-data-setup work above

### When the Matrix Itself Is Wrong

For 20+ developer codebases with ISV-heavy customizations, **Pattern 7 (Unlocked Packages) is a better answer than a wider matrix**. Decompose your customizations into unlocked packages, version-pin against the ISV's package, and run per-package tests. Matrix is a stopgap when packages aren't yet feasible.
