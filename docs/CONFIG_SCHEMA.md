# PR Scan Configuration Schema v3.3

This document describes the `onboarded-repos.json` configuration file schema for the DevSecOps PR Security Scanning solution.

## Overview

Starting from v3.0, repository onboarding is managed through a **centralized metadata file** (`onboarded-repos.json`) stored in the buildspec repository (`pr-scan-buildspecs`). This replaces the previous per-repository `.pr-scan.yml` approach.

### Benefits of Centralized Onboarding

| Benefit | Description |
|---------|-------------|
| **Single Point of Control** | One file manages all repositories |
| **No Repository Changes** | Developers don't need to add config files |
| **Easy Onboarding** | Add/remove repos by editing one JSON file |
| **Consistent Configuration** | Standard format across all repositories |
| **Audit Trail** | Git history shows all onboarding changes |

---

## Schema Reference

### File Location

```
pr-scan-buildspecs (CodeCommit)
├── buildspec/
│   └── pr-scan-buildspec.yaml
└── onboarded-repos.json          ← Configuration file
```

### Root Level Structure

```json
{
  "version": "3.2",
  "description": "PR Security Scan - Onboarded Repositories",
  "repositories": {
    "repository-name": {
      "appName": "Display Name",
      "sonarProject": "sonar-project-key",
      "lang": "nodejs",
      "bu": "Business Unit",
      "sbu": "Sub Business Unit",
      "qualityGate": "Sonar way"
    }
  }
}
```

### Root Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `version` | string | Yes | Schema version (current: `"3.2"`) |
| `description` | string | No | Human-readable description |
| `repositories` | object | Yes | Map of repository configurations |

### Repository Configuration

Each key in `repositories` must be the **exact CodeCommit repository name**.

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `appName` | string | No | Repository name | Human-readable application name for PR comments |
| `sonarProject` | string | No | Repository name | SonarQube project key (enables cost optimization) |
| `lang` | string | **Recommended** | `"generic"` | Programming language for reporting |
| `bu` | string | No | - | Business Unit for reporting |
| `sbu` | string | No | - | Sub-Business Unit for reporting |
| `qualityGate` | string | No | Project default | SonarQube Quality Gate name |

---

## Property Details

### lang (Language)

The `lang` property specifies the repository's primary programming language. This is used for:
- Accurate reporting in PR comments
- Metadata in scan evidence files
- Future language-specific scan configurations

**Supported Values:**

| Value | Description | Detection Indicators |
|-------|-------------|---------------------|
| `nodejs` | Node.js/JavaScript/TypeScript | `package.json` |
| `java-maven` | Java with Maven | `pom.xml` |
| `java-gradle` | Java with Gradle | `build.gradle` |
| `python` | Python | `requirements.txt` |
| `go` | Go/Golang | `go.mod` |
| `dotnet` | .NET/C# | `*.csproj`, `*.sln` |
| `generic` | Any other language | Default fallback |

**Example:**
```json
{
  "my-nodejs-app": {
    "lang": "nodejs"
  },
  "payment-service": {
    "lang": "java-gradle"
  }
}
```

### sonarProject (Cost Optimization)

The `sonarProject` property enables **shared SonarQube projects** across multiple AWS accounts.

**How SonarQube Billing Works:**
- SonarQube bills based on **Lines of Code (LOC)** per project
- Same project with multiple branches = LOC counted **once**
- Different projects = LOC counted **separately**

**Cost Optimization Example:**

```
WITHOUT sonarProject:
┌─────────────────────────────────────────────────────────────┐
│  UAT Account:   payment-api-APP_ACCOUNT_ID  →  50,000 LOC    │
│  PROD Account:  payment-api-CENTRAL_ACCOUNT_ID  →  50,000 LOC    │
│  ─────────────────────────────────────────────────────────  │
│  TOTAL: 100,000 LOC billed                                  │
└─────────────────────────────────────────────────────────────┘

WITH sonarProject: "payment-api"
┌─────────────────────────────────────────────────────────────┐
│  UAT Account:   branch "feature/xyz"  ─┐                    │
│  PROD Account:  branch "main"         ─┴→  50,000 LOC      │
│  ─────────────────────────────────────────────────────────  │
│  TOTAL: 50,000 LOC billed (50% savings!)                   │
└─────────────────────────────────────────────────────────────┘
```

**Recommendation:** Always specify `sonarProject` if the same codebase exists in multiple accounts.

### qualityGate (Quality Gate)

The `qualityGate` property specifies which SonarQube Quality Gate to apply.

**Important:** Quality Gate pass/fail is **100% controlled by SonarQube**. The buildspec does NOT apply any additional thresholds.

**Common Quality Gates:**

| Gate Name | Description |
|-----------|-------------|
| `Sonar way` | SonarQube default (recommended) |
| `Strict` | Zero tolerance for issues |
| `Custom Gate` | Your organization's custom gate |

**Behavior:**

| Configuration | Behavior |
|---------------|----------|
| `"qualityGate": "Sonar way"` | Uses "Sonar way" Quality Gate |
| `"qualityGate": "My Custom Gate"` | Uses custom gate (must exist) |
| Not specified | Uses SonarQube project's default |

---

## Complete Examples

### Minimal Configuration

```json
{
  "version": "3.2",
  "repositories": {
    "my-app": {}
  }
}
```

This will:
- Scan `my-app` repository
- Use repo name as SonarQube project key
- Use SonarQube project's default Quality Gate
- Set language to `generic`

### Standard Configuration

```json
{
  "version": "3.2",
  "description": "PR Security Scan - Onboarded Repositories",
  "repositories": {
    "super-app-be": {
      "appName": "Super App Backend",
      "sonarProject": "super-app-be",
      "lang": "nodejs",
      "bu": "Technology",
      "sbu": "Platform Engineering",
      "qualityGate": "Sonar way"
    }
  }
}
```

### Multi-Repository Configuration

```json
{
  "version": "3.2",
  "description": "DevSecOps - Onboarded Repositories",
  "repositories": {
    "super-app-be": {
      "appName": "Super App Backend",
      "sonarProject": "super-app-be",
      "lang": "nodejs",
      "bu": "Technology",
      "sbu": "Platform Engineering",
      "qualityGate": "Sonar way"
    },
    "payment-api": {
      "appName": "Payment Gateway API",
      "sonarProject": "payment-api",
      "lang": "java-gradle",
      "bu": "Payments",
      "sbu": "Digital Payments"
    },
    "claims-service": {
      "appName": "Health Claims Service",
      "sonarProject": "claims-service",
      "lang": "java-maven",
      "bu": "Claims",
      "sbu": "Health Claims",
      "qualityGate": "Strict"
    },
    "data-pipeline": {
      "appName": "Data Processing Pipeline",
      "sonarProject": "data-pipeline",
      "lang": "python",
      "bu": "Analytics",
      "sbu": "Data Engineering"
    },
    "mobile-gateway": {
      "appName": "Mobile API Gateway",
      "sonarProject": "mobile-gateway",
      "lang": "go",
      "bu": "Mobile",
      "sbu": "Mobile Platform"
    }
  }
}
```

---

## Onboarding Workflow

### Adding a New Repository

1. **Clone the buildspec repository:**
   ```bash
   git clone codecommit::ap-south-1://pr-scan-buildspecs
   cd pr-scan-buildspecs
   ```

2. **Edit onboarded-repos.json:**
   ```bash
   # Add your repository entry
   vi onboarded-repos.json
   ```

3. **Commit and push:**
   ```bash
   git add onboarded-repos.json
   git commit -m "Onboard my-new-app repository"
   git push origin master
   ```

4. **Create a PR in the target repository** to trigger the first scan.

### Removing a Repository

1. Remove the repository entry from `onboarded-repos.json`
2. Commit and push
3. Future PRs in that repository will be skipped with message:
   ```
   [SKIP] Repository 'repo-name' is NOT onboarded
   ```

---

## Not Onboarded Behavior

When a repository is not in `onboarded-repos.json`, the scan exits gracefully with a helpful message:

```
========================================================
[SKIP] Repository 'my-repo' is NOT onboarded
========================================================

This repository is not listed in onboarded-repos.json.

To onboard this repository:
  1. Edit onboarded-repos.json in pr-scan-buildspecs repo
  2. Add entry for 'my-repo' with required fields
  3. Commit and push

Example entry:
  "my-repo": {
    "appName": "Application Display Name",
    "sonarProject": "my-repo",
    "lang": "nodejs",
    "bu": "Business Unit",
    "sbu": "Sub Business Unit",
    "qualityGate": "Sonar way"
  }

========================================================
```

---

## Output in PR Comment

The configuration values appear in the PR comment:

```markdown
## 🔒 Security Scan Report

**Application:** Super App Backend
**Repository:** super-app-be
**Language:** nodejs
**Quality Gate:** Sonar way ✅ PASSED

| Category | Total | Blocker | Critical | Major | Minor |
|----------|-------|---------|----------|-------|-------|
| Bugs | 8 | 0 | 0 | 8 | 0 |
| Vulnerabilities | 0 | 0 | 0 | 0 | 0 |
...
```

---

## Output in Evidence File

The configuration values are included in `pr-scan-report.json`:

```json
{
  "schemaVersion": "3.3",
  "metadata": {
    "correlationId": "super-app-be-PR814-20261117T064701Z",
    "appId": "super-app-be",
    "appName": "Super App Backend",
    "bu": "Technology",
    "sbu": "Platform Engineering",
    "repository": "super-app-be",
    "prId": "814",
    "language": "nodejs",
    "sonarProject": "super-app-be",
    "qualityGate": "Sonar way"
  },
  "result": {
    "overallStatus": "PASSED",
    "mergeRecommendation": "SAFE_TO_MERGE",
    "qualityGateStatus": "OK"
  }
}
```

---

## SonarQube Severity Mapping

SonarQube uses the following severity levels:

| Severity | Impact | Description |
|----------|--------|-------------|
| BLOCKER | Critical | Must be fixed immediately, blocks release |
| CRITICAL | High | High impact issue, fix as priority |
| MAJOR | Medium | Significant issue, should be addressed |
| MINOR | Low | Minor issue, fix when possible |
| INFO | Informational | Informational finding |

**Note:** Quality Gate thresholds are configured in SonarQube, not in the buildspec or metadata file.

---

## Migration Guide

### From v2.5 (.pr-scan.yml) to v3.2 (onboarded-repos.json)

The v3.0+ model removes per-repository configuration files:

**Before (v2.5):** Each repository had its own `.pr-scan.yml`:
```yaml
# .pr-scan.yml in each repo (DEPRECATED)
appId: "my-app"
appName: "My Application"
sonarProject: "my-app"
environment: "auto"
scan:
  type: "full"
policy:
  prod:
    failOnCritical: true
    maxHighAllowed: 0
```

**After (v3.2):** Single centralized file:
```json
{
  "version": "3.2",
  "repositories": {
    "my-app": {
      "appName": "My Application",
      "sonarProject": "my-app",
      "lang": "nodejs",
      "qualityGate": "Sonar way"
    }
  }
}
```

**Key Changes:**
| v2.5 Feature | v3.2 Status |
|--------------|-------------|
| `.pr-scan.yml` per repo | **Removed** - use centralized file |
| `environment` auto-detection | **Removed** - not needed |
| `scan.type` | **Removed** - always full scan |
| `policy.prod/uat/dev` thresholds | **Removed** - use SonarQube Quality Gate |
| `appType` | **Renamed** to `lang` |

---

## Validation

### JSON Syntax Validation

```bash
# Check JSON syntax
jq '.' onboarded-repos.json

# Pretty print
jq '.' onboarded-repos.json > temp.json && mv temp.json onboarded-repos.json
```

### Check if Repository is Onboarded

```bash
# Check if repo exists
jq -r '.repositories["my-app"]' onboarded-repos.json

# List all onboarded repos
jq -r '.repositories | keys[]' onboarded-repos.json
```

---

## Troubleshooting

### Repository Not Being Scanned

1. **Check exact repo name match:**
   ```bash
   jq -r '.repositories | keys[]' onboarded-repos.json
   ```

2. **Ensure JSON syntax is valid:**
   ```bash
   jq '.' onboarded-repos.json
   ```

3. **Check CodeBuild logs** for "NOT ONBOARDED" message

### Quality Gate Not Applied

1. **Verify Quality Gate exists in SonarQube:**
   ```bash
   curl -u "TOKEN:" "http://SONARQUBE:9000/api/qualitygates/list"
   ```

2. **Check exact name match** (case-sensitive)

### Language Shows as "generic"

1. **Add `lang` field** to repository configuration
2. **Use valid value:** `nodejs`, `java-maven`, `java-gradle`, `python`, `go`, `dotnet`, `generic`

---

## Reference

### File Paths

| File | Location | Purpose |
|------|----------|---------|
| `onboarded-repos.json` | `pr-scan-buildspecs/` | Repository configurations |
| `pr-scan-buildspec.yaml` | `pr-scan-buildspecs/buildspec/` | Scan buildspec |

### Related Documentation

- [README.md](../README.md) - Overview and quick start
- [SOLUTION_DESIGN.md](../SOLUTION_DESIGN.md) - Detailed architecture

### Accounts

| Account | ID | Purpose |
|---------|-----|---------|
| Central | `YOUR_CENTRAL_ACCOUNT_ID` | Buildspec repo, CodeBuild, S3 |
| Application | `YOUR_APP_ACCOUNT_ID` | CodeCommit repositories |

---

**Version:** 3.3  
**Author:** Noventiq  
**Last Updated:** November 2026
