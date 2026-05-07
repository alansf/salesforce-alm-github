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
          echo "${{ secrets[format('SFDX_AUTH_URL_{0}', matrix.org_alias)] }}" > auth_url.txt
          sf org login sfdx-url --sfdx-url-file auth_url.txt --alias ${{ matrix.org_alias }}
      
      - name: Deploy and Test
        run: |
          sf project deploy validate \
            --target-org ${{ matrix.org_alias }} \
            --test-level RunLocalTests \
            --wait 60
```

**When to use**: Users need to test across multiple orgs, multiple Salesforce editions, or multiple API versions simultaneously.

### Pattern 2: OIDC (OpenID Connect) with Custom Properties

**The Problem**: Managing long-lived secrets (SFDX Auth URLs, certificates) is risky and doesn't provide fine-grained governance based on repository metadata.

**The Solution**: Use GitHub OIDC to authenticate without secrets, combined with Custom Properties as claims to enforce governance at the identity layer.

**Workshop Value**: "We can use Custom Properties (e.g., `compliance-tier: SOX`) as claims in our OIDC tokens. If a repository isn't tagged as 'SOX compliant' in GitHub, it literally cannot access the Production Salesforce secret."

**Implementation Steps**:

1. **Configure Custom Properties in GitHub Enterprise**:
   - Navigate to Organization Settings → Custom Properties
   - Create properties like:
     - `compliance-tier` (options: SOX, HIPAA, PCI, Standard)
     - `agentforce-enabled` (boolean)
     - `deployment-tier` (options: dev, qa, staging, production)

2. **Set up Connected App in Salesforce**:
   - Create a Connected App with OAuth settings
   - Enable "Use digital signatures"
   - Configure callback URL for GitHub Actions
   - Note the Consumer Key and Consumer Secret

3. **Create GitHub OIDC Workflow**:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate Repository Compliance
        run: |
          # GitHub provides custom properties via API
          COMPLIANCE_TIER="${{ vars.COMPLIANCE_TIER }}"
          if [[ "$COMPLIANCE_TIER" != "SOX" ]]; then
            echo "❌ Repository not SOX compliant. Cannot deploy to production."
            exit 1
          fi
      
      - name: Get OIDC Token
        id: oidc
        run: |
          OIDC_TOKEN=$(curl -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
            "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=salesforce" | jq -r .value)
          echo "::add-mask::$OIDC_TOKEN"
          echo "token=$OIDC_TOKEN" >> $GITHUB_OUTPUT
      
      - name: Authenticate to Salesforce via OIDC
        run: |
          sf org login jwt \
            --client-id ${{ secrets.SF_CONSUMER_KEY }} \
            --jwt-key-file <(echo "${{ secrets.SF_JWT_KEY }}") \
            --username ${{ secrets.SF_USERNAME }} \
            --instance-url https://login.salesforce.com \
            --alias production
```

**When to use**: Users need fine-grained access control, compliance enforcement, or want to eliminate long-lived secrets from their deployment pipelines.

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
          echo "${{ secrets.SFDX_AUTH_URL_PROD }}" > auth_url.txt
          sf org login sfdx-url --sfdx-url-file auth_url.txt --alias production
      
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
          echo "${{ secrets.SFDX_AUTH_URL_PROD }}" > auth_url.txt
          sf org login sfdx-url --sfdx-url-file auth_url.txt --alias production
      
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
- The artifact retention ensures the Job ID is available for 30 days
- If the artifact is missing (e.g., expired), fall back to a full deployment
- This pattern works across workflow files and workflow runs

**When to use**: Users want to optimize deployment speed, reduce test execution time, or implement a validation → promotion workflow.

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
          echo "${{ secrets.SFDX_AUTH_URL_PROD }}" > auth_url.txt
          sf org login sfdx-url --sfdx-url-file auth_url.txt --alias production
      
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
      
      - name: Standard Security Scan
        run: |
          npm install -g @salesforce/sfdx-scanner
          sfdx scanner:run --target "force-app/**/*.cls" --format sarif --outfile results.sarif
      
      - name: Agentforce-Specific Security Scan
        if: needs.setup.outputs.agentforce_enabled == 'true'
        run: |
          echo "🤖 Running Agentforce security checks..."
          
          # Check for prompt injection vulnerabilities
          echo "Scanning for prompt injection patterns..."
          grep -r "user.*input.*prompt" force-app/ || echo "No prompt injection risks found"
          
          # Check for proper grounding implementation
          echo "Validating LLM grounding implementation..."
          grep -r "@AuraEnabled.*grounding" force-app/ || echo "Warning: No grounding methods found"
          
          # Check for data privacy in AI context
          echo "Checking for PII in agent instructions..."
          # Add your custom Agentforce security logic here

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

## Additional Key Workflows

### Branch Strategy for Salesforce

Recommend a branch strategy based on team size and deployment frequency:

**For Small Teams (1-5 developers)**:
- **Trunk-based development**: Direct commits to `main`, short-lived feature branches
- Feature branches: `feature/description`
- Validate on PR, quick deploy on merge
- Use scratch orgs for feature development

**For Medium Teams (5-20 developers)**:
- **GitHub Flow variant**:
  - `main` (production)
  - `develop` (integration)
  - `feature/*` (features)
  - `hotfix/*` (production fixes)
- Validate on PR, deploy to sandbox from `develop`, promote to production from `main`

**For Large Teams (20+ developers)**:
- **Gitflow with release branches**:
  - `main` (production)
  - `release/*` (release candidates)
  - `develop` (integration)
  - `feature/*` (features)
  - `hotfix/*` (production fixes)
- Use package-based development for parallel work streams

### Testing Strategy

Implement a comprehensive testing strategy in workflows:

```yaml
- name: Run Apex Tests
  run: |
    sf apex run test \
      --target-org ${{ matrix.org_alias }} \
      --test-level RunLocalTests \
      --code-coverage \
      --result-format human \
      --wait 60

- name: Validate Code Coverage
  run: |
    COVERAGE=$(sf apex get test --target-org ${{ matrix.org_alias }} --json | jq '.result.summary.testRunCoverage' | tr -d '"' | cut -d'%' -f1)
    REQUIRED_COVERAGE=${{ vars.TEST_COVERAGE_REQUIRED || 75 }}
    
    if (( $(echo "$COVERAGE < $REQUIRED_COVERAGE" | bc -l) )); then
      echo "❌ Code coverage $COVERAGE% is below required $REQUIRED_COVERAGE%"
      exit 1
    fi
    echo "✅ Code coverage $COVERAGE% meets requirement"

- name: Static Code Analysis
  run: |
    npm install -g @salesforce/sfdx-scanner
    sfdx scanner:run \
      --target "force-app/**/*.cls,force-app/**/*.trigger" \
      --engine pmd \
      --category "Security,Best Practices,Code Style" \
      --severity-threshold 3

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
    echo "${{ secrets.SFDX_AUTH_URL }}" > auth_url.txt
    sf org login sfdx-url --sfdx-url-file auth_url.txt --alias target-org
```

**2. JWT (Recommended for Production)**:
```yaml
- name: Authenticate via JWT
  run: |
    echo "${{ secrets.SF_JWT_KEY }}" > jwt-key.txt
    sf org login jwt \
      --client-id ${{ secrets.SF_CONSUMER_KEY }} \
      --jwt-key-file jwt-key.txt \
      --username ${{ secrets.SF_USERNAME }} \
      --instance-url https://login.salesforce.com \
      --alias production
    rm jwt-key.txt
```

**3. OIDC (Most Secure, GitHub Enterprise)**:
- See Pattern 2 above

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
          echo "${{ secrets.SFDX_AUTH_URL_DEVHUB }}" > auth_url.txt
          sf org login sfdx-url --sfdx-url-file auth_url.txt --alias devhub --set-default-dev-hub
      
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

When working with Agentforce projects, add these additional steps:

### 1. Agentforce Security Scanning
Create a dedicated security scan job for Agentforce:

```yaml
agentforce-security:
  runs-on: ubuntu-latest
  if: vars.AGENTFORCE_ENABLED == 'true'
  steps:
    - uses: actions/checkout@v4
    
    - name: Scan for Prompt Injection Risks
      run: |
        echo "🔍 Scanning for prompt injection vulnerabilities..."
        
        # Check for unsafe user input handling in prompts
        if grep -r "String.*prompt.*=.*" force-app/ | grep -v "sanitize\|escape"; then
          echo "⚠️  Warning: Found potential prompt injection risks"
          echo "Ensure all user inputs are sanitized before being used in prompts"
        fi
    
    - name: Validate Grounding Implementation
      run: |
        echo "🔍 Validating LLM grounding..."
        
        # Ensure grounding methods exist
        if ! grep -r "@AuraEnabled.*grounding" force-app/; then
          echo "⚠️  Warning: No grounding methods found"
          echo "Agentforce agents should implement proper grounding"
        fi
    
    - name: Check for PII in Agent Instructions
      run: |
        echo "🔍 Checking for PII exposure..."
        
        # Check agent definition files for hardcoded PII
        if grep -r "ssn\|social.*security\|credit.*card\|passport" force-app/ --include="*.agent-meta.xml"; then
          echo "❌ Error: Potential PII found in agent instructions"
          exit 1
        fi
    
    - name: Validate Agent Topics Configuration
      run: |
        echo "🔍 Validating agent topics..."
        
        # Ensure topics are properly configured
        find force-app/ -name "*.agent-meta.xml" -exec echo "Checking: {}" \;
```

### 2. Agent Testing Workflows
```yaml
test-agents:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    
    - name: Install SF CLI
      run: npm install -g @salesforce/cli
    
    - name: Authenticate to Org
      run: |
        echo "${{ secrets.SFDX_AUTH_URL }}" > auth_url.txt
        sf org login sfdx-url --sfdx-url-file auth_url.txt --alias target-org
    
    - name: Deploy Agent Metadata
      run: |
        sf project deploy start \
          --target-org target-org \
          --metadata-dir force-app/main/default/agents
    
    - name: Test Agent Responses
      run: |
        # Use SF CLI or Apex to trigger agent conversations
        # Validate responses against expected outcomes
        echo "Testing agent conversation flows..."
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

This skill enables Salesforce architects to modernize their teams' ALM practices by leveraging GitHub Enterprise's advanced features. The five core patterns (Matrix Strategy, OIDC with Custom Properties, Quick Deploy, Repository Dispatch, and Custom Properties for Governance) provide capabilities that traditional declarative tools cannot match, especially for teams scaling toward Agentforce and multi-cloud architectures. Use this skill to both educate teams on modern practices and to implement production-ready CI/CD pipelines.
