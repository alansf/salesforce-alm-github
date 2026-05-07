# Salesforce ALM with GitHub

## Modern Application Lifecycle Management for Salesforce Using GitHub Actions

This Claude Code skill guides Salesforce teams through modernizing their application lifecycle management by migrating from declarative, UI-driven change management tools to GitHub-native, API-first orchestration patterns designed for Agentforce and multi-cloud architectures.

[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blue.svg)](https://claude.ai/code)
[![Salesforce](https://img.shields.io/badge/Salesforce-ALM-00a1e0.svg)](https://www.salesforce.com/)
[![GitHub Actions](https://img.shields.io/badge/GitHub-Actions-2088FF.svg)](https://github.com/features/actions)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)]()

---

## Table of Contents

* [Modern Application Lifecycle Management for Salesforce Using GitHub Actions](#modern-application-lifecycle-management-for-salesforce-using-github-actions)
* [Table of Contents](#table-of-contents)
  * [What Problem Does This Solve?](#what-problem-does-this-solve)
  * [Solution Overview](#solution-overview)
  * [The Five GitHub Actions Superpowers](#the-five-github-actions-superpowers)
  * [Key Features](#key-features)
  * [Quick Start](#quick-start)
    * [Installation](#installation)
    * [Basic Usage](#basic-usage)
  * [Common Salesforce ALM Scenarios](#common-salesforce-alm-scenarios)
    * [1. Set Up CI/CD for Salesforce Project](#1-set-up-cicd-for-salesforce-project)
    * [2. Test Across Multiple Orgs in Parallel](#2-test-across-multiple-orgs-in-parallel)
    * [3. Implement Governance and Agentforce Security](#3-implement-governance-and-agentforce-security)
    * [4. Speed Up Production Deployments](#4-speed-up-production-deployments)
    * [5. Conduct ALM Workshop](#5-conduct-alm-workshop)
  * [The Five Patterns Explained](#the-five-patterns-explained)
    * [Pattern 1: Matrix Strategy](#pattern-1-matrix-strategy)
    * [Pattern 2: OIDC with Custom Properties](#pattern-2-oidc-with-custom-properties)
    * [Pattern 3: Quick Deploy Pattern](#pattern-3-quick-deploy-pattern)
    * [Pattern 4: Repository Dispatch](#pattern-4-repository-dispatch)
    * [Pattern 5: Custom Properties for Governance](#pattern-5-custom-properties-for-governance)
  * [Agentforce-Specific Capabilities](#agentforce-specific-capabilities)
  * [Why GitHub Actions for Salesforce ALM?](#why-github-actions-for-salesforce-alm)
  * [Operating Modes](#operating-modes)
  * [Branch Strategies](#branch-strategies)
  * [Authentication Methods](#authentication-methods)
  * [Best Practices](#best-practices)
  * [Integration with Salesforce Skills](#integration-with-salesforce-skills)
  * [GitHub Enterprise](#github-enterprise)
  * [Troubleshooting](#troubleshooting)
  * [Development](#development)
  * [Evaluation Tests](#evaluation-tests)
  * [Contributing](#contributing)
  * [License](#license)

---

## What Problem Does This Solve?

When building enterprise Salesforce applications, especially with Agentforce and multi-cloud architectures, traditional declarative change management tools create bottlenecks:

1. **Manual Deployments**: Button-clicking deployments don't scale to multiple daily releases
2. **Sequential Testing**: Testing across multiple sandboxes takes hours of manual work
3. **Weak Governance**: No fine-grained control over who can deploy what to which environment
4. **Slow Deployments**: Running the same Apex tests multiple times wastes 30-45 minutes per deployment
5. **Siloed Systems**: No coordination between Salesforce, external APIs, and multi-cloud components
6. **Agentforce Security Gaps**: No automated scanning for prompt injection, PII exposure, or grounding issues

This skill provides **GitHub-native, API-first orchestration patterns** that enable teams to move from button-clicking to modular automation, essential for scaling toward Agentforce and multi-cloud integration.

---

## Solution Overview

The `salesforce-alm-github` skill provides comprehensive ALM modernization capabilities:

- **🚀 Five GitHub Actions Patterns**: Matrix Strategy, OIDC, Quick Deploy, Repository Dispatch, Custom Properties
- **🎯 Dual Operating Modes**: Advisory (workshop facilitation) + Implementation (actual workflow generation)
- **🤖 Agentforce-Ready**: Prompt injection scanning, PII detection, grounding validation
- **🏢 Enterprise Governance**: Custom Properties for SOX/HIPAA compliance, fine-grained access control
- **⚡ Speed Optimizations**: Parallel testing (3x faster), Quick Deploy (93% time savings)
- **🔄 Multi-Cloud Orchestration**: Cross-repository coordination via Repository Dispatch
- **📚 Comprehensive**: 500+ lines covering patterns, authentication, testing, troubleshooting
- **🎓 Workshop Materials**: Talking points, facilitation guides, and ROI analysis for team education

---

## The Five GitHub Actions Superpowers

These five patterns are unique capabilities that GitHub Enterprise provides over traditional declarative tools:

1. **Matrix Strategy**: Test across multiple Salesforce orgs simultaneously (3x faster than sequential)
2. **OIDC + Custom Properties**: Zero-trust authentication with automatic governance enforcement
3. **Quick Deploy Pattern**: Reuse validation results for 3-minute production deployments (vs 45 minutes)
4. **Repository Dispatch**: Cross-system testing coordination for multi-cloud architectures
5. **Custom Properties for Governance**: Centralized repository classification with conditional quality gates

---

## Key Features

✅ **Matrix Strategy Implementation**: Parallel testing across FSC, Health Cloud, Manufacturing Cloud, etc.  
✅ **OIDC Authentication**: Eliminate long-lived secrets with OpenID Connect  
✅ **Quick Deploy Workflows**: GitHub Artifacts pass validation Job IDs between workflows  
✅ **Custom Properties Governance**: SOX/HIPAA compliance tiers with automatic enforcement  
✅ **Agentforce Security Scanning**: Prompt injection detection, PII checks, grounding validation  
✅ **Workshop Facilitation**: Complete materials for conducting ALM modernization workshops  
✅ **Branch Strategy Guidance**: Trunk-based, GitHub Flow, or Gitflow based on team size  
✅ **Authentication Patterns**: SFDX Auth URL, JWT, and OIDC with setup guidance  
✅ **GitHub Enterprise Support**: Self-hosted GitHub with VPN, SSL, and proxy configuration  
✅ **Scratch Org Workflows**: Ephemeral test environments for feature development  

---

## Quick Start

### Installation

This skill is available for Claude Code. To verify it's installed:

```bash
# List available skills
claude skills list | grep salesforce-alm-github
```

If not installed, add the skill:

```bash
# Install from repository
claude skills install github.com/alansf/salesforce-alm-github
```

### Basic Usage

**Scenario: Set up CI/CD for Salesforce project**

Simply ask Claude Code:

```
"We're migrating off our current change management tool and need to set up GitHub Actions 
for our Salesforce project. We have a dev hub, three sandboxes (dev, qa, staging), and 
production. Can you help us get started with CI/CD? We want to validate changes on PRs 
and deploy to production when we merge to main."
```

Claude will:
1. Generate complete `.github/workflows/` directory with all workflow files
2. Create validate-pr.yml for PR validation with Apex tests
3. Create deploy-production.yml with quick deploy pattern
4. Provide authentication setup documentation (GitHub secrets, Connected Apps)
5. Include branch strategy and testing best practices
6. Generate troubleshooting guide and setup checklist

**Quick Pattern Implementation**:

```
"Show me how to implement the Matrix Strategy pattern to test my Apex changes 
across three different Salesforce orgs simultaneously"
```

Claude will generate a workflow with parallel testing configuration and time savings analysis.

---

## Common Salesforce ALM Scenarios

### 1. Set Up CI/CD for Salesforce Project

**Context**: Migrating from declarative tool, need automated PR validation and production deployment.

**What Claude Does**:
- Generates complete workflow files (validate-pr.yml, deploy-production.yml)
- Provides GitHub secrets configuration guide
- Includes authentication setup (SFDX Auth URLs or JWT)
- Recommends branch strategy based on team size
- Creates comprehensive setup documentation

**Example Workflows Generated**:
- Validate PRs with Apex tests and static analysis
- Deploy to sandboxes on merge to develop
- Quick deploy to production on merge to main
- Scratch org creation for feature testing

---

### 2. Test Across Multiple Orgs in Parallel

**Context**: Need to test changes against Financial Services Cloud, Health Cloud, and Enterprise Edition simultaneously.

**What Claude Does**:
- Generates Matrix Strategy workflow with parallel jobs
- Configures authentication for each org type
- Calculates time savings (sequential 45 min → parallel 15 min)
- Provides org-specific secret configuration guide
- Includes failure handling (fail-fast: false)

**Time Savings**: 66% reduction in deployment time

---

### 3. Implement Governance and Agentforce Security

**Context**: Compliance team requires SOX controls and Agentforce projects need prompt injection scanning.

**What Claude Does**:
- Generates Custom Properties setup guide (compliance-tier, agentforce-enabled)
- Creates conditional workflows based on repository classification
- Implements SOX compliance checks (separation of duties, audit trails, 7-year retention)
- Adds Agentforce security scanning (prompt injection, PII exposure, grounding validation)
- Provides GitHub Enterprise configuration documentation

**Governance Features**: Build-failing on critical security findings, automatic approval requirements

---

### 4. Speed Up Production Deployments

**Context**: Production deployments take 45 minutes because Apex tests run twice.

**What Claude Does**:
- Generates Quick Deploy workflows using GitHub Artifacts
- PR validation workflow captures Salesforce validation Job ID
- Production deployment reuses Job ID with `sf project deploy quick`
- Includes fallback to full deployment if artifact missing
- Provides ROI analysis showing annual time savings

**Time Savings**: 45 minutes → 3 minutes (93% reduction)

---

### 5. Conduct ALM Workshop

**Context**: Need to educate team on GitHub Actions, address nervousness about moving away from UI-driven tools.

**What Claude Does**:
- Generates comprehensive workshop materials (talking points, slides outline, facilitator guide)
- Provides business justifications for all five patterns with ROI calculations
- Addresses team concerns ("losing control", "don't know Git", "seems complicated")
- Emphasizes Agentforce benefits (rapid iteration, multi-cloud coordination, security)
- Includes success metrics and follow-up plan
- Avoids naming specific vendor tools (focuses on capabilities vs competition)

**Workshop Duration**: 90-minute structured presentation with hands-on exercises

---

## The Five Patterns Explained

### Pattern 1: Matrix Strategy

**Problem**: Testing across multiple Salesforce org types requires sequential manual deployments.

**Solution**: GitHub Actions matrix jobs run simultaneously.

**Value**: 
- Test FSC, Health Cloud, Manufacturing Cloud in parallel
- 3x faster than sequential (15 min vs 45 min)
- Single workflow handles all org types

**Use When**: Multiple Salesforce editions, multiple API versions, cross-org compatibility testing

---

### Pattern 2: OIDC with Custom Properties

**Problem**: Long-lived secrets are risky and don't provide governance based on repository metadata.

**Solution**: GitHub OIDC authenticates without secrets, Custom Properties enforce governance at identity layer.

**Value**:
- No secrets to rotate or leak
- Automatic enforcement: SOX-compliant repos can't access non-compliant orgs
- Fine-grained access control without manual management

**Use When**: Enterprise environments, compliance requirements (SOX/HIPAA/PCI), GitHub Enterprise

---

### Pattern 3: Quick Deploy Pattern

**Problem**: Salesforce deployments take 45+ minutes because Apex tests run during validation AND deployment.

**Solution**: GitHub Artifacts pass validation Job ID from PR to production deployment workflow.

**Value**:
- Reuse validation results instead of re-running tests
- 45 min → 3 min production deployments
- 434 hours saved per year (typical team)

**Use When**: Large test suites, frequent deployments, teams prioritizing speed

---

### Pattern 4: Repository Dispatch

**Problem**: Multi-cloud architectures need cross-system coordination (Salesforce + external APIs).

**Solution**: Repository Dispatch lets one repo trigger workflows in another repo.

**Value**:
- Coordinate deployments across systems
- Trigger Salesforce integration tests when external API changes
- Build "Integrated ALM" for Agentforce multi-cloud architectures

**Use When**: Multi-repository projects, external system dependencies, agent architectures

---

### Pattern 5: Custom Properties for Governance

**Problem**: Different project types need different quality gates without duplicating workflows.

**Solution**: Tag repositories with Custom Properties, use conditional workflow steps.

**Value**:
- Single workflow adapts to repository type
- Agentforce repos automatically get security scanning
- SOX repos automatically get enhanced compliance checks
- Scales to hundreds of repositories

**Use When**: Multiple project types, centralized governance, GitHub Enterprise

---

## Agentforce-Specific Capabilities

### Prompt Injection Scanning

Detects unsafe string concatenation and unsanitized user input in agent prompts:

```bash
# Scans for patterns like:
String prompt = userInput + "...";  # UNSAFE
```

### PII Exposure Detection

Checks agent metadata files for hardcoded PII (SSN, credit cards, passport numbers):

```bash
# Fails build if PII found in .agent-meta.xml files
```

### Grounding Validation

Ensures proper LLM grounding implementation:

```bash
# Checks for @AuraEnabled grounding methods
# Validates data source configuration
```

### Security Context Verification

Validates "with sharing" usage in Apex classes used by agents:

```bash
# Ensures proper field-level security (FLS) enforcement
```

---

## Why GitHub Actions for Salesforce ALM?

### vs. Declarative Tools

| Capability | Declarative Tools | GitHub Actions |
|------------|------------------|----------------|
| Parallel testing | ❌ Sequential only | ✅ Matrix Strategy |
| Secret management | ⚠️ Long-lived secrets | ✅ OIDC (no secrets) |
| Deployment speed | ⚠️ 45+ min | ✅ 3 min (Quick Deploy) |
| Cross-system coordination | ❌ Not supported | ✅ Repository Dispatch |
| Repository-based governance | ❌ Not supported | ✅ Custom Properties |
| Agentforce security | ❌ Manual checks | ✅ Automated scanning |
| Multi-cloud orchestration | ⚠️ Limited | ✅ Native support |

### Agentforce Requirements

Modern AI agent architectures require:
- **Rapid iteration**: 2+ deploys per day (vs 2 per week)
- **Multi-cloud coordination**: Salesforce + external LLMs + data clouds
- **Specialized security**: Prompt injection, PII leakage, grounding validation
- **Complete auditability**: Every change tracked with approval trails

GitHub Actions provides these capabilities natively; declarative tools were not designed for AI-first applications.

---

## Operating Modes

### 1. Workshop/Advisory Mode

Explain patterns, demonstrate value, provide migration guidance.

**Example**:
```
"I'm running an ALM workshop next week. What are the key talking points for the 
five GitHub Actions patterns?"
```

**Output**: Workshop materials, facilitation guide, ROI analysis, team concerns Q&A

---

### 2. Implementation Mode

Generate actual workflow files, configure repositories, set up authentication.

**Example**:
```
"Set up a CI/CD pipeline with the Quick Deploy pattern for my Salesforce project"
```

**Output**: Complete .github/workflows files, setup documentation, troubleshooting guide

---

## Branch Strategies

### Small Teams (1-5 developers)

**Trunk-Based Development**
- Direct commits to `main`
- Short-lived feature branches
- Validate on PR, quick deploy on merge

---

### Medium Teams (5-20 developers)

**GitHub Flow Variant**
- `main` (production)
- `develop` (integration)
- `feature/*` (features)
- `hotfix/*` (production fixes)

---

### Large Teams (20+ developers)

**Gitflow with Release Branches**
- `main` (production)
- `release/*` (release candidates)
- `develop` (integration)
- `feature/*` (features)
- `hotfix/*` (production fixes)

---

## Authentication Methods

### 1. SFDX Auth URL (Simplest)

```bash
sf org login web --alias myorg
sf org display --target-org myorg --verbose --json | jq -r '.result.sfdxAuthUrl'
# Store as GitHub secret: SFDX_AUTH_URL
```

**Pros**: Easy to set up, works immediately  
**Cons**: Long-lived secret, manual rotation

---

### 2. JWT (Recommended for Production)

```bash
sf org login jwt \
  --client-id ${{ secrets.SF_CONSUMER_KEY }} \
  --jwt-key-file jwt-key.txt \
  --username ${{ secrets.SF_USERNAME }}
```

**Pros**: More secure, certificate-based  
**Cons**: Requires Connected App setup

---

### 3. OIDC (Most Secure, GitHub Enterprise)

```yaml
permissions:
  id-token: write
steps:
  - name: Get OIDC Token
    run: curl -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
         "$ACTIONS_ID_TOKEN_REQUEST_URL" | jq -r .value
```

**Pros**: No secrets, automatic governance via Custom Properties  
**Cons**: Requires GitHub Enterprise, more complex setup

---

## Best Practices

### 1. Test After Every Change
- Deploy to scratch org
- Run Apex tests
- Verify code coverage (75%+ for production)
- Check static analysis (PMD, ESLint)

### 2. Use Specific Secrets per Environment
```
SFDX_AUTH_URL_DEV
SFDX_AUTH_URL_QA
SFDX_AUTH_URL_STAGING
SFDX_AUTH_URL_PROD
```

### 3. Protect Production Branch
- Require PR approvals (2+)
- Require status checks to pass
- Restrict who can push to main
- Enable environment protection with approvals

### 4. Monitor Deployment Speed
- Track validation time
- Track deployment time
- Set alerts for slowdowns
- Optimize test execution

### 5. Document Approval Trails
```bash
git commit -m "Deploy customer portal updates

Approved by: Platform Architect (Jira: SF-1234)
Tested in: QA sandbox (2026-05-06)
Deployment window: May 6, 10pm-11pm PST"
```

---

## Integration with Salesforce Skills

This skill works alongside other Salesforce development skills:

| Skill | Purpose | Integration Point |
|-------|---------|-------------------|
| **developing-agentforce** | Build Agentforce agents | Deploy agents via CI/CD |
| **testing-agentforce** | Test agent functionality | Integrate tests in workflows |
| **generating-apex** | Generate Apex classes | Deploy Apex via workflows |
| **deploying-ui-bundle** | Deploy React apps | Coordinate deployments |
| **generating-flow** | Create Salesforce Flows | Include flows in deployments |

**Workflow**:
```
1. Set up ALM with GitHub Actions (← THIS SKILL)
2. Develop Salesforce features (generating-apex, developing-agentforce)
3. Create workflows that test and deploy automatically
4. Monitor deployments and iterate
```

---

## GitHub Enterprise

Full support for self-hosted GitHub instances.

### Authentication
- SSH keys (recommended)
- Personal access tokens
- SAML/SSO authorization

### OIDC Configuration
- Custom identity provider setup
- Custom Properties as OIDC claims
- Fine-grained access control

### SSL Certificates
- Add to system trust store
- Configure git sslCAInfo
- Troubleshooting guidance

### Connectivity
- VPN requirement checks
- Proxy configuration
- DNS resolution verification

**Example**:
```bash
# Enterprise workflow
git remote add origin git@github.acme-corp.com:platform/salesforce-app.git
```

---

## Troubleshooting

### Workflow fails with "Resource not accessible by integration"
**Cause**: Missing workflow permissions  
**Solution**: Add `permissions: contents: read, pull-requests: write, id-token: write`

### SF CLI authentication fails
**Cause**: Invalid auth URL format  
**Solution**: Regenerate auth URL with `sf org display --verbose --json | jq -r '.result.sfdxAuthUrl'`

### Artifact not found in quick deploy
**Cause**: Artifact expired or validation workflow didn't complete  
**Solution**: Add fallback logic with `continue-on-error: true` and full deployment fallback

### Matrix strategy fails on one org but succeeds on others
**Cause**: Org-specific configuration or permissions  
**Solution**: Use `fail-fast: false` and investigate failed org separately

### OIDC authentication fails with custom properties
**Cause**: Custom properties not exposed as variables or OIDC provider misconfigured  
**Solution**: Verify Custom Properties in repo settings and OIDC trust relationship includes correct claims

---

## Development

### Project Structure

```
salesforce-alm-github/
├── SKILL.md          # Complete skill definition (500+ lines)
├── README.md         # This file
├── LICENSE           # MIT License
└── evals/
    └── evals.json    # 5 comprehensive test scenarios
```

### Testing the Skill

The skill includes evaluation tests covering:

1. Basic CI/CD setup for Salesforce project
2. Matrix strategy for parallel testing
3. Governance with Custom Properties and Agentforce security
4. Quick deploy pattern for speed optimization
5. Workshop facilitation for team education

**Run evaluations**:
```bash
claude skills eval salesforce-alm-github
```

---

## Evaluation Tests

The skill includes 5 comprehensive Salesforce ALM scenarios:

| Test ID | Scenario | Coverage |
|---------|----------|----------|
| 1 | Basic CI/CD setup | Workflows, authentication, testing, documentation |
| 2 | Matrix strategy | Parallel testing, multiple orgs, time savings |
| 3 | Governance & Agentforce | Custom Properties, SOX compliance, security scanning |
| 4 | Quick deploy | Artifact passing, Job ID reuse, fallback logic |
| 5 | Workshop facilitation | All 5 patterns, ROI analysis, team concerns |

**Expected Outcomes**: Each test validates pattern implementation, documentation completeness, and Salesforce-specific guidance.

---

## Contributing

Contributions are welcome from the Salesforce developer community!

**To contribute**:
1. Fork this repository
2. Create a feature branch (`git checkout -b feature/alm-improvement`)
3. Make your changes
4. Test with evaluation suite
5. Commit (`git commit -m 'Add matrix strategy enhancement'`)
6. Push to your fork
7. Open a Pull Request

**Guidelines**:
- Maintain Salesforce ALM focus
- Include GitHub Actions context
- Add tests for new scenarios
- Keep explanations practical and actionable

---

## License

MIT License - See [LICENSE](LICENSE) file for details.

---

## Resources

- 🚀 [GitHub Actions Documentation](https://docs.github.com/en/actions)
- ⚡ [Salesforce CLI](https://developer.salesforce.com/tools/salesforcecli)
- 📚 [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/)
- 🤖 [Agentforce Documentation](https://help.salesforce.com/s/articleView?id=sf.agents_overview.htm)
- 💼 [GitHub Enterprise Server](https://docs.github.com/en/enterprise-server)
- 🔐 [OpenID Connect in GitHub Actions](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- 🤖 [Claude Code Skills](https://docs.anthropic.com/claude/docs/claude-code)

---

<p align="center">
  <strong>Built for Salesforce Teams Modernizing Their ALM</strong>
</p>

<p align="center">
  <a href="https://github.com/alansf/salesforce-alm-github">GitHub</a> •
  <a href="https://www.salesforce.com/">Salesforce</a> •
  <a href="https://claude.ai">Claude AI</a>
</p>
