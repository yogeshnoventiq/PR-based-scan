# DevSecOps - PR-Based Security Scanning

> Automated security scanning for Pull Requests using self-hosted SonarQube Enterprise

[![Version](https://img.shields.io/badge/Version-3.3-blue)]()
[![AWS](https://img.shields.io/badge/AWS-CodeBuild%20|%20EventBridge-orange)]()
[![SonarQube](https://img.shields.io/badge/SonarQube-Enterprise-green)]()

---

## Overview

Centralized security scanning for Pull Requests across multiple AWS accounts. When a PR is created or updated, CodeBuild scans the code using SonarQube Enterprise and posts results as PR comments.

**Scan Capabilities:**
- **SAST** - Static Application Security Testing (Bugs, Vulnerabilities, Code Smells)
- **SCA** - Software Composition Analysis (Enterprise feature)
- **Secrets** - Hardcoded credentials detection
- **SBOM** - Software Bill of Materials (Enterprise Advanced Security)
- **Security Hotspots** - Potential security issues requiring review
- **Auto-Approval** - PRs automatically approved when Quality Gate passes (v3.3)

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                      APPLICATION ACCOUNT(S)                        │
│                                                                    │
│   ┌──────────────┐         ┌─────────────────┐                     │
│   │  CodeCommit  │ ──PR──▶ │  EventBridge    │ ────────────┐       │
│   │  Repository  │         │  Rule           │             │       │
│   └──────────────┘         └─────────────────┘             │       │
└────────────────────────────────────────────────────────────┼───────┘
                                                             │
                                                             ▼
┌────────────────────────────────────────────────────────────────────┐
│                        CENTRAL ACCOUNT                             │
│                                                                    │
│   ┌─────────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│   │  Event Bus      │───▶│  EventBridge │───▶│    CodeBuild     │  │
│   │  (pr-scan)      │    │  Rule        │    │                  │  │
│   └─────────────────┘    └──────────────┘    │ 1. Check metadata│  │
│                                              │ 2. Clone repo    │  │
│   ┌─────────────────────────────────────┐    │ 3. Run SonarQube │  │
│   │  pr-scan-buildspecs (CodeCommit)    │    │ 4. Post comment  │  │
│   │  ├── buildspec/pr-scan-buildspec.yaml│   │ 5. Upload to S3  │  │
│   │  └── onboarded-repos.json  ◄────────────┘                   │  │
│   └─────────────────────────────────────┘    └──────────────────┘  │
│                                                                    │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│   │ S3 Evidence  │  │ SSM Params   │  │ Secrets Mgr  │             │
│   └──────────────┘  └──────────────┘  └──────────────┘             │
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                  ┌───────────────────────────┐
                  │   SonarQube Enterprise    │
                  │   (Self-Hosted EC2)       │
                  └───────────────────────────┘
```

---

## Onboarding Model

Repositories are onboarded via a centralized metadata file (`onboarded-repos.json`) stored in the buildspec repository.

| Repo in Metadata? | Behavior |
|-------------------|----------|
| ✅ Yes | Scanned |
| ❌ No | Skipped with clear message |

**To onboard a repository:** Add an entry to `onboarded-repos.json` and push.

---

## Quick Start

### Step 1: Deploy Central Account

```bash
aws cloudformation deploy \
  --template-file cloudformation/central-account-resources.yaml \
  --stack-name pr-scan-central \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    VpcId="vpc-xxxxxxxx" \
    PrivateSubnetIds="subnet-xxx,subnet-yyy" \
    SonarQubeToken="squ_your-token" \
    SonarQubeHostUrl="http://YOUR_SONARQUBE_IP:9000" \
    SonarQubeIpCidr="10.x.x.x/32"
```

### Step 2: Push Buildspec + Metadata

```bash
# Clone the buildspec repository
git clone codecommit::ap-south-1://pr-scan-buildspecs

# Copy files
cp buildspec/pr-scan-buildspec.yaml pr-scan-buildspecs/buildspec/
cp onboarded-repos.json pr-scan-buildspecs/

# Push
cd pr-scan-buildspecs
git add .
git commit -m "Add PR scan buildspec v3.2"
git push origin master
```

### Step 3: Deploy App Account(s)

```bash
aws cloudformation deploy \
  --template-file cloudformation/app-account-resources.yaml \
  --stack-name pr-scan-app \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    CentralAccountId="CENTRAL_ACCOUNT_ID" \
    CentralEventBusArn="arn:aws:events:ap-south-1:CENTRAL_ACCOUNT_ID:event-bus/pr-scan-events" \
    CentralCodeBuildRoleArn="arn:aws:iam::CENTRAL_ACCOUNT_ID:role/pr-scan-codebuild-role"
```

### Step 4: Onboard Repositories

Edit `onboarded-repos.json` in the buildspec repository:

```json
{
  "version": "3.2",
  "repositories": {
    "my-application": {
      "appName": "My Application",
      "sonarProject": "my-application",
      "lang": "nodejs",
      "bu": "Technology",
      "sbu": "Platform Engineering",
      "qualityGate": "Sonar way"
    }
  }
}
```

Commit and push. The repository will be scanned on next PR.

---

## Metadata File Reference

### onboarded-repos.json

```json
{
  "version": "3.2",
  "description": "Onboarded repositories for PR security scanning",
  "repositories": {
    "REPO_NAME": {
      "appName": "Display Name (optional)",
      "sonarProject": "SonarQube project key (optional, defaults to REPO_NAME)",
      "lang": "nodejs | java-maven | java-gradle | python | go | dotnet | generic",
      "bu": "Business Unit (optional)",
      "sbu": "Sub Business Unit (optional)",
      "qualityGate": "SonarQube Quality Gate name (optional, uses project default)"
    }
  }
}
```

### Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `appName` | No | Display name shown in PR comments |
| `sonarProject` | No | SonarQube project key (cost optimization for shared projects) |
| `lang` | **Recommended** | Programming language: `nodejs`, `java-maven`, `java-gradle`, `python`, `go`, `dotnet`, `generic` |
| `bu` | No | Business unit for reporting |
| `sbu` | No | Sub-business unit for reporting |
| `qualityGate` | No | SonarQube Quality Gate name (e.g., "Sonar way", "Strict") |

### Language Support

| `lang` Value | Detected By | Notes |
|--------------|-------------|-------|
| `nodejs` | `package.json` | Node.js/JavaScript/TypeScript |
| `java-maven` | `pom.xml` | Java with Maven |
| `java-gradle` | `build.gradle` | Java with Gradle |
| `python` | `requirements.txt` | Python |
| `go` | `go.mod` | Go/Golang |
| `dotnet` | `*.csproj`, `*.sln` | .NET/C# |
| `generic` | (default) | Any other language |

### Quality Gate Behavior

Quality Gate pass/fail is **100% controlled by SonarQube**. The buildspec does not apply any additional thresholds.

| `qualityGate` Value | Behavior |
|---------------------|----------|
| `"Sonar way"` | Uses "Sonar way" Quality Gate |
| `"My Custom Gate"` | Uses your custom QG (must exist in SonarQube) |
| Not specified | Uses SonarQube project's default QG |

---

## Cost Optimization

### Shared SonarQube Projects

If the same repository exists in multiple accounts (UAT/PROD), use `sonarProject` to share a single SonarQube project:

```json
{
  "repositories": {
    "payment-api": {
      "sonarProject": "payment-api",
      "lang": "java-gradle"
    }
  }
}
```

| Without `sonarProject` | With `sonarProject` |
|------------------------|---------------------|
| UAT: `payment-api-111111111111` | `payment-api` |
| PROD: `payment-api-222222222222` | `payment-api` |
| **LOC counted twice** | **LOC counted once** |

---

## Configuration

### SSM Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `/pr-scan/allowed-branches` | `main,uat,master` | Destination branches that trigger scans |

### Secrets Manager

**Secret:** `pr-scan/sonarqube`

```json
{
  "token": "squ_your_user_token_here",
  "host_url": "http://YOUR_SONARQUBE_IP:9000"
}
```

**Important Token Types:**

| Token Type | Prefix | Works with Scanner 8.x |
|------------|--------|------------------------|
| **User Token** ✅ | `squ_` | Yes |
| Global Analysis Token ❌ | `sqa_` | No |
| Project Analysis Token ❌ | `sqp_` | No |

> **Note:** SonarScanner 8.x requires User Tokens (`squ_`). Global Analysis Tokens (`sqa_`) do not work with the v2 API.

---

## PR Comment

After scanning, a report is posted as a PR comment including:

- **Overall Status** - PASSED / WARNING / FAILED
- **Merge Recommendation** - Based on Quality Gate result
- **Security Summary:**
  - Bugs (with severity breakdown)
  - Vulnerabilities (with severity breakdown)
  - Secrets detected
  - Code Smells (with severity breakdown)
  - Security Hotspots
  - Dependencies (Enterprise only)
- **Quality Gate Result** - From SonarQube
- **Links** - Direct links to SonarQube dashboard

---

## Evidence Storage (S3)

All scan evidence is stored in S3 with the following structure:

```
s3://pr-scan-evidence-{account}-{region}/{source-account}/{app-id}/pr-{pr-id}/
├── pr-scan-report.json              ← Combined insights + PR comment
├── sast/
│   ├── bugs-report.json             ← Raw SonarQube API response
│   ├── vulnerabilities-report.json
│   ├── code-smells-report.json
│   └── hotspots-report.json
└── secrets/
    └── secrets-report.json
```

### Output File: pr-scan-report.json

The main output file contains:

```json
{
  "schemaVersion": "3.3",
  "metadata": {
    "correlationId": "app-id-PR123-20261117T064701Z",
    "appId": "my-application",
    "appName": "My Application",
    "repository": "my-application",
    "prId": "123",
    "sourceBranch": "feature-branch",
    "destBranch": "main",
    "language": "nodejs",
    "timestamp": "2026-11-17T06:47:58Z",
    "qualityGate": "Sonar way"
  },
  "result": {
    "overallStatus": "PASSED",
    "mergeRecommendation": "SAFE_TO_MERGE",
    "qualityGateStatus": "OK"
  },
  "findings": {
    "bugs": { "total": 8, "blocker": 0, "critical": 0, "major": 8, "minor": 0 },
    "vulnerabilities": { "total": 0, "blocker": 0, "critical": 0, "major": 0, "minor": 0 },
    "secrets": { "total": 0, "blocker": 0, "critical": 0, "major": 0, "minor": 0 },
    "codeSmells": { "total": 74, "blocker": 0, "critical": 3, "major": 15, "minor": 56 },
    "securityHotspots": { "total": 4 },
    "dependencies": { "count": 0, "sbom": "Available in SonarQube UI" }
  },
  "links": {
    "dashboard": "http://sonarqube:9000/dashboard?id=my-app&branch=feature",
    "issues": "http://sonarqube:9000/project/issues?id=my-app&branch=feature",
    "hotspots": "http://sonarqube:9000/security_hotspots?id=my-app&branch=feature"
  },
  "prComment": "## 🔒 Security Scan Report..."
}
```

---

## Project Structure

```
PR-based-scan/
├── cloudformation/
│   ├── central-account-resources.yaml    ← Central account (CodeBuild, EventBridge, S3)
│   └── app-account-resources.yaml        ← App accounts (EventBridge rule, IAM roles)
├── buildspec/
│   └── pr-scan-buildspec.yaml            ← Main buildspec file
├── docs/
│   └── CONFIG_SCHEMA.md                  ← Detailed configuration reference
├── README.md                             ← This file
└── SOLUTION_DESIGN.md                    ← Detailed solution design document
```

---

## Network Requirements

CodeBuild runs in a VPC with the following requirements:

| Requirement | Purpose |
|-------------|---------|
| **Private Subnets** | CodeBuild instances run here |
| **NAT Gateway** | Internet access for package downloads |
| **SonarQube Access** | Network path to SonarQube (VPC Peering/Transit Gateway/Direct Connect) |

### Security Group Egress Rules

| Port | Destination | Purpose |
|------|-------------|---------|
| 9000 | SonarQube IP | SonarQube API and scanner uploads |
| 443 | 0.0.0.0/0 | Package downloads (npm, pip, maven) via NAT |
| 80 | 0.0.0.0/0 | Some package registries (via NAT) |

### Domains to Whitelist (if using proxy)

Only 4 domains needed:
1. `binaries.sonarsource.com` - SonarScanner download
2. `github.com` - yq tool download
3. `objects.githubusercontent.com` - GitHub releases
4. Your SonarQube host

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Repo not being scanned | Add repo to `onboarded-repos.json` |
| "NOT ONBOARDED" message | Repo name must match exactly in metadata file |
| Quality Gate not applied | Verify QG name exists in SonarQube |
| PR comment not posted | Check `PR-Scan-CodeCommit-Access-Role` trust policy |
| SonarQube connection failed | Verify Security Group allows CodeBuild → SonarQube:9000 |
| Authentication failed (401) | Use User Token (`squ_`), not Global Analysis Token (`sqa_`) |
| Dependencies = 0 | Normal for Community Edition (SCA is Enterprise feature) |

---

## Version History

| Version | Changes |
|---------|---------|
| **3.3** | Auto-Approval when Quality Gate passes, Approval Rule Template, `EnableAutoApproval` parameter |
| **3.2** | Branch filtering in EventBridge (cost optimization), Language in metadata (`lang` field), merged output file |
| **3.1** | Removed CycloneDX (SonarQube Advanced Security handles SBOM) |
| **3.0** | Centralized metadata onboarding, dynamic Quality Gate, removed per-repo config |
| 2.6 | Opt-in model with `.pr-scan.yml` |
| 2.5 | Self-hosted SonarQube, per-environment thresholds |

---

## Auto-Approval (v3.3)

When a PR passes the SonarQube Quality Gate, it is automatically approved by the security scanner.

### How It Works

1. **Quality Gate PASSED** → PR auto-approved by `SecurityScanner`
2. **Quality Gate FAILED/WARNING** → PR NOT approved, developer must fix issues

### Setup

1. Deploy `app-account-resources.yaml` with `EnableAutoApproval=true` (default)
   - This automatically creates the Approval Rule Template via Lambda Custom Resource
2. Associate the template with repositories:

```bash
# Single repository
aws codecommit associate-approval-rule-template-with-repository \
  --approval-rule-template-name "pr-scan-SecurityScanApproval" \
  --repository-name "your-repo-name"

# Multiple repositories
aws codecommit batch-associate-approval-rule-template-with-repositories \
  --approval-rule-template-name "pr-scan-SecurityScanApproval" \
  --repository-names "repo1" "repo2" "repo3"
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `EnableAutoApproval` | `true` | Create approval rule template and enable approval |
| `ApprovalSessionName` | `SecurityScanner` | Session name for approvals |

### Disable Auto-Approval

To disable, set `EnableAutoApproval=false` in CloudFormation or `ENABLE_AUTO_APPROVAL=false` in buildspec environment variables.

---

## Accounts Reference

| Account | ID | Purpose |
|---------|-----|---------|
| Central | `YOUR_CENTRAL_ACCOUNT_ID` | CodeBuild, EventBridge, S3 Evidence |
| Application | `YOUR_APP_ACCOUNT_ID` | CodeCommit repositories |

---

**Author:** Noventiq  
**Version:** 3.3
