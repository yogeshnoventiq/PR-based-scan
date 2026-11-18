# DevSecOps - PR-Based Security Scanning
## Solution Design Document

**Version:** 3.4  
**Author:** Noventiq  
**Date:** November 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Business Requirements](#2-business-requirements)
3. [Solution Architecture](#3-solution-architecture)
4. [Component Details](#4-component-details)
5. [Cross-Account Design](#5-cross-account-design)
6. [Security Scanning Flow](#6-security-scanning-flow)
7. [Data Flow Diagram](#7-data-flow-diagram)
8. [IAM Permissions](#8-iam-permissions)
9. [Configuration Management](#9-configuration-management)
10. [Onboarding Process](#10-onboarding-process)
11. [Security Considerations](#11-security-considerations)
12. [Evidence Storage](#12-evidence-storage)
13. [Cost Analysis](#13-cost-analysis)
14. [Troubleshooting](#14-troubleshooting)
15. [Appendix](#15-appendix)

---

## 1. Executive Summary

### 1.1 Overview

This document describes the PR-Based Security Scanning solution for DevSecOps implementation. The solution provides automated security scanning when Pull Requests are created or updated, without requiring changes to existing pipelines.

### 1.2 Key Benefits

| Benefit | Description |
|---------|-------------|
| **Zero Pipeline Changes** | Completely independent of existing 500+ pipelines |
| **Shift-Left Security** | Catch vulnerabilities before code is merged |
| **Developer Friendly** | Results posted as PR comments for easy review |
| **Centralized Management** | Single CodeBuild project scans all repositories |
| **Cross-Account Support** | Works with 200+ repos across multiple AWS accounts |
| **Audit Ready** | All scan evidence stored in S3 for compliance |
| **Unified Security Platform** | SonarQube Enterprise handles SAST, SCA, Secrets, SBOM |
| **Cost Optimized** | Shared projects across accounts (LOC counted once) |
| **Minimal Dependencies** | No pip install needed, only 4 domains to whitelist |

### 1.3 Approach: Lift & Shift

The solution uses a **Lift & Shift** approach - security scanning is performed when PR is created/updated, NOT during the build pipeline. This ensures:

- No modifications to existing pipelines
- No delays to deployment workflows
- Independent scaling of security scanning
- Admin controls merge decisions based on scan results

### 1.4 Version History

| Version | Key Changes |
|---------|-------------|
| **3.4** | Configurable logging with `LOG_LEVEL` (ERROR/INFO/DEBUG) |
| **3.3** | Auto-Approval when Quality Gate passes, Approval Rule Template |
| **3.2** | Branch filtering in EventBridge, Language in metadata (`lang`) |
| **3.1** | Removed CycloneDX (SonarQube Advanced Security handles SBOM) |
| **3.0** | Centralized metadata onboarding, dynamic Quality Gate |
| 2.6 | Opt-in model with `.pr-scan.yml` |
| 2.5 | Self-hosted SonarQube, per-environment thresholds |

### 1.5 v3.4 Key Features

| Feature | Description |
|---------|-------------|
| **LOG_LEVEL** | Configurable logging: ERROR (minimal) | INFO | DEBUG (verbose) |
| **Cleaner Logs** | Suppressed git progress, S3 progress, API response JSON |
| **Auto-Approval** | PRs automatically approved when Quality Gate passes |
| **Approval Rule Template** | Created in app account (associate with repos) |
| **Branch Filtering** | EventBridge filters by allowed branches (cost optimization) |
| **Centralized Onboarding** | Single `onboarded-repos.json` file controls all repos |
| **Quality Gate Control** | 100% controlled by SonarQube (no buildspec thresholds) |
| **Minimal Whitelisting** | Only 4 domains needed |
| **Merged Output** | Single `pr-scan-report.json` file in S3 |

---

## 2. Business Requirements

### 2.1 Functional Requirements

| ID | Requirement | Status |
|----|-------------|--------|
| FR-01 | Scan PR when created or source branch updated | ✅ Implemented |
| FR-02 | Post scan results as PR comment | ✅ Implemented |
| FR-03 | Support multiple destination branches | ✅ Configurable via SSM |
| FR-04 | Generate SBOM (Software Bill of Materials) | ✅ SonarQube Enterprise |
| FR-05 | Scan for vulnerabilities (SCA) | ✅ SonarQube Enterprise |
| FR-06 | Perform SAST analysis | ✅ SonarQube Enterprise |
| FR-07 | Detect hardcoded secrets | ✅ SonarQube Enterprise |
| FR-08 | Store scan evidence | ✅ S3 bucket |
| FR-09 | Cross-account repository access | ✅ AssumeRole |
| FR-10 | Support AWS Organization accounts | ✅ Org ID policy |
| FR-11 | Support external (non-org) accounts | ✅ Explicit account list |
| FR-12 | Shared project across accounts (cost) | ✅ `sonarProject` config |
| FR-13 | Centralized repository onboarding | ✅ `onboarded-repos.json` |
| FR-14 | Language configuration in metadata | ✅ `lang` field |
| FR-15 | Branch filtering in EventBridge | ✅ Cost optimization |
| FR-16 | Auto-approval when Quality Gate passes | ✅ v3.3 |

### 2.2 Non-Functional Requirements

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-01 | Scan completion time | < 5 minutes |
| NFR-02 | Scan evidence retention | 90 days |
| NFR-03 | No changes to existing pipelines | 0 changes |
| NFR-04 | Onboarding time per account | < 30 minutes |
| NFR-05 | Multi-language support | Node.js, Java, Python, Go, .NET |
| NFR-06 | Domains to whitelist | 4 domains only |

---

## 3. Solution Architecture

### 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           APPLICATION ACCOUNTS                                   │
│                        (200+ repositories, multiple accounts)                    │
│                                                                                  │
│   ┌──────────────┐   PR Created    ┌──────────────────────────────────┐         │
│   │              │   or Updated    │                                  │         │
│   │  CodeCommit  │ ─────────────▶  │  EventBridge Rule                │         │
│   │  Repository  │                 │  (pr-scan-event-forwarder)       │         │
│   │              │                 │                                  │         │
│   └──────────────┘                 └──────────────┬───────────────────┘         │
│                                                   │                              │
│   ┌──────────────┐                               │                              │
│   │ IAM Role     │◀──────────────────────────────┤                              │
│   │ (CodeCommit  │   Assumed by Central          │                              │
│   │  Access)     │   CodeBuild                   │                              │
│   └──────────────┘                               │                              │
│                                                   │                              │
└───────────────────────────────────────────────────┼──────────────────────────────┘
                                                    │
                                                    │ PutEvents (Cross-Account)
                                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              CENTRAL ACCOUNT                                     │
│                            (CENTRAL_ACCOUNT_ID)                                        │
│                                                                                  │
│   ┌──────────────────┐                                                          │
│   │   Event Bus      │  Receives events from all App Accounts                   │
│   │ (pr-scan-events) │                                                          │
│   └────────┬─────────┘                                                          │
│            │                                                                     │
│            │ Matches PR Events                                                   │
│            ▼                                                                     │
│   ┌──────────────────┐        ┌───────────────────────────────────────┐         │
│   │  EventBridge     │───────▶│         CodeBuild Project             │         │
│   │  Rule            │        │     (pr-scan-security-scan)           │         │
│   │  (pr-scan-       │        │                                       │         │
│   │   trigger)       │        │  ┌─────────────────────────────────┐  │         │
│   └──────────────────┘        │  │ 1. Check onboarded-repos.json   │  │         │
│                               │  │ 2. Assume role in App Account   │  │         │
│   ┌──────────────────┐        │  │ 3. Clone source code            │  │         │
│   │  CodeCommit      │◀───────│  │ 4. Run SonarQube Scanner        │  │         │
│   │  (buildspecs)    │ Fetch  │  │ 5. Fetch results from API       │  │         │
│   │  ├─buildspec/    │        │  │ 6. Post PR comment              │  │         │
│   │  └─onboarded-    │        │  │ 7. Upload evidence to S3        │  │         │
│   │    repos.json    │        │  │ 8. Emit EventBridge event       │  │         │
│   └──────────────────┘        │  └─────────────────────────────────┘  │         │
│                               └───────────────────────────────────────┘         │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                          │
│   │ S3 Evidence  │  │ SSM Params   │  │ Secrets Mgr  │                          │
│   │ Bucket       │  │ (branches)   │  │ (SonarQube)  │                          │
│   └──────────────┘  └──────────────┘  └──────────────┘                          │
│                                                                                  │
│   ┌──────────────────────────────────────────────────┐                          │
│   │  VPC Configuration                                │                          │
│   │  ├─ Private Subnets (CodeBuild runs here)        │                          │
│   │  ├─ NAT Gateway (internet access)                │                          │
│   │  └─ Security Group (egress to SonarQube)         │                          │
│   └──────────────────────────────────────────────────┘                          │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ HTTPS/HTTP (port 9000)
                                    ▼
                  ┌───────────────────────────────────────────┐
                  │      SONARQUBE ENTERPRISE (EC2)           │
                  │     (Self-Hosted - Advanced Security)     │
                  │                                           │
                  │   ┌────────────┐   ┌────────────┐        │
                  │   │    SAST    │   │    SCA     │        │
                  │   │ Code Bugs  │   │ Dependency │        │
                  │   │ Code Smells│   │ Vulns      │        │
                  │   └────────────┘   └────────────┘        │
                  │                                           │
                  │   ┌────────────┐   ┌────────────┐        │
                  │   │  Secrets   │   │   SBOM     │        │
                  │   │ Detection  │   │ (Advanced) │        │
                  │   └────────────┘   └────────────┘        │
                  │                                           │
                  │   ┌────────────┐   ┌────────────┐        │
                  │   │  Security  │   │  Quality   │        │
                  │   │  Hotspots  │   │   Gates    │        │
                  │   └────────────┘   └────────────┘        │
                  │                                           │
                  │   URL: http://YOUR_SONARQUBE_IP:9000          │
                  └───────────────────────────────────────────┘
```

### 3.2 Component Summary

| Component | Account | Purpose |
|-----------|---------|---------|
| CodeCommit Repositories | App Accounts | Source code (200+ repos) |
| EventBridge Rule | App Accounts | Catch PR events, forward to Central |
| Event Bus | Central | Receive events from all accounts |
| EventBridge Rule | Central | Trigger CodeBuild on PR events |
| CodeBuild Project | Central | Execute security scans (VPC enabled) |
| CodeCommit (buildspecs) | Central | Store buildspec + onboarded-repos.json |
| S3 Bucket | Central | Store scan evidence |
| SSM Parameter | Central | Configurable allowed branches |
| Secrets Manager | Central | SonarQube credentials (token + host_url) |
| SonarQube Enterprise | EC2 (Self-Hosted) | SAST, SCA, Secrets, SBOM analysis |

### 3.3 Security Scanning Capabilities

| Scan Type | Tool | Description |
|-----------|------|-------------|
| **SAST** | SonarQube | Bugs, Vulnerabilities, Code Smells |
| **SCA** | SonarQube Enterprise | Dependency vulnerability scanning |
| **Secrets** | SonarQube | Hardcoded credentials detection |
| **Security Hotspots** | SonarQube | Potential security issues requiring review |
| **SBOM** | SonarQube Advanced Security | Software Bill of Materials |
| **Quality Gate** | SonarQube | Customizable pass/fail criteria |

### 3.4 SonarQube Severity Model

| Severity | Impact Level | Description |
|----------|--------------|-------------|
| BLOCKER | Critical | Must be fixed immediately |
| CRITICAL | High | High impact security/reliability issue |
| MAJOR | Medium | Significant issue that should be fixed |
| MINOR | Low | Minor issue, fix when possible |
| INFO | Informational | Informational finding |

**Important Notes:**
- SonarQube uses `severities` facet (not `impactSeverities` like SonarCloud)
- SonarQube does NOT use `organization` parameter (unlike SonarCloud)
- Quality Gate is 100% controlled by SonarQube (no buildspec thresholds)

---

## 4. Component Details

### 4.1 EventBridge Event Bus (Central Account)

**Resource:** `pr-scan-events`

**Purpose:** Receives PR events from all Application Accounts

**Policy:** Allows PutEvents from:
- AWS Organization (all accounts via Org ID)
- Explicit external accounts (for non-org accounts)

### 4.2 EventBridge Rule (Central Account)

**Resource:** `pr-scan-trigger`

**Event Pattern:**
```json
{
  "source": ["aws.codecommit"],
  "detail-type": ["CodeCommit Pull Request State Change"],
  "detail": {
    "event": ["pullRequestCreated", "pullRequestSourceBranchUpdated"]
  }
}
```

**InputTransformer Variables:**
| Variable | Description |
|----------|-------------|
| `PR_ID` | Pull Request ID |
| `REPO_NAME` | Repository name |
| `SOURCE_BRANCH` | Source branch reference |
| `DEST_BRANCH` | Destination branch reference |
| `SOURCE_COMMIT` | Source commit SHA |
| `DEST_COMMIT` | Destination commit SHA |
| `SOURCE_ACCOUNT_ID` | Application Account ID |
| `SOURCE_REGION` | Application Account region |

### 4.3 CodeBuild Project (Central Account)

**Resource:** `pr-scan-security-scan`

**Configuration:**
| Setting | Value |
|---------|-------|
| Image | `aws/codebuild/amazonlinux2-x86_64-standard:5.0` |
| Compute | `BUILD_GENERAL1_MEDIUM` |
| Timeout | 30 minutes |
| Source | CodeCommit (buildspec repository) |
| VPC | Yes (private subnets with NAT Gateway) |
| Artifacts | None |

**Environment Variables:**
| Variable | Source | Description |
|----------|--------|-------------|
| `SONAR_SECRET_ARN` | CloudFormation | ARN of SonarQube secret |
| `EVIDENCE_BUCKET` | CloudFormation | S3 bucket name |
| `ALLOWED_BRANCHES_PARAM` | CloudFormation | SSM parameter path |
| `CENTRAL_ACCOUNT_ID` | CloudFormation | Central account ID |
| `LOG_LEVEL` | Buildspec | Logging verbosity: `ERROR` (default), `INFO`, `DEBUG` |
| `ENABLE_AUTO_APPROVAL` | Buildspec | Enable PR auto-approval: `true` (default), `false` |
| `APPROVAL_SESSION_NAME` | Buildspec | Session name for approvals: `SecurityScanner` (default) |

### 4.4 Secrets Manager Structure

**Secret Name:** `pr-scan/sonarqube`

```json
{
  "token": "squ_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "host_url": "http://YOUR_SONARQUBE_IP:9000"
}
```

**Token Type Requirements:**

| Token Type | Prefix | Works with Scanner 8.x |
|------------|--------|------------------------|
| **User Token** ✅ | `squ_` | Yes (uses v2 API) |
| Global Analysis Token ❌ | `sqa_` | No |
| Project Analysis Token ❌ | `sqp_` | No |

> **Important:** SonarScanner 8.x requires User Tokens (`squ_`). Global Analysis Tokens (`sqa_`) do not work with the v2 API.

### 4.5 Network Configuration

**VPC Requirements:**

| Component | Purpose |
|-----------|---------|
| Private Subnets | CodeBuild instances run here |
| NAT Gateway | Internet access for package downloads |
| Security Group | Egress rules for SonarQube and internet |

**Security Group Egress Rules:**

| Port | Destination | Purpose |
|------|-------------|---------|
| 9000 | SonarQube IP CIDR | SonarQube API |
| 443 | 0.0.0.0/0 | HTTPS (packages via NAT) |
| 80 | 0.0.0.0/0 | HTTP (packages via NAT) |

**Domains to Whitelist (if proxy):**

1. `binaries.sonarsource.com` - SonarScanner download
2. `github.com` - yq tool download
3. `objects.githubusercontent.com` - GitHub releases
4. Your SonarQube host IP/domain

---

## 5. Cross-Account Design

### 5.1 Account Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AWS ORGANIZATION                                │
│                                                                      │
│   ┌────────────────────┐                                            │
│   │  Central Account   │                                            │
│   │  CENTRAL_ACCOUNT_ID      │                                            │
│   │                    │                                            │
│   │  - Event Bus       │                                            │
│   │  - CodeBuild       │                                            │
│   │  - S3 Evidence     │                                            │
│   └────────────────────┘                                            │
│            ▲                                                         │
│            │ Events + AssumeRole                                     │
│            │                                                         │
│   ┌────────┴───────┬───────────────┬───────────────┐                │
│   │                │               │               │                │
│   ▼                ▼               ▼               ▼                │
│ ┌─────────┐  ┌─────────┐   ┌─────────┐   ┌─────────┐               │
│ │ App     │  │ App     │   │ App     │   │ App     │               │
│ │ Account │  │ Account │   │ Account │   │ Account │   ...         │
│ │ #1      │  │ #2      │   │ #3      │   │ #N      │               │
│ │ 3976... │  │ xxxx... │   │ xxxx... │   │ xxxx... │               │
│ └─────────┘  └─────────┘   └─────────┘   └─────────┘               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Cross-Account AssumeRole

**From:** Central CodeBuild (`pr-scan-codebuild-role`)  
**To:** App Account (`PR-Scan-CodeCommit-Access-Role`)

```bash
ROLE_ARN="arn:aws:iam::${SOURCE_ACCOUNT_ID}:role/PR-Scan-CodeCommit-Access-Role"

CREDENTIALS=$(aws sts assume-role \
  --role-arn "$ROLE_ARN" \
  --role-session-name "pr-scan-${PR_ID}" \
  --external-id "pr-scan-cross-account" \
  --duration-seconds 3600)
```

**Trust Policy (App Account Role):**
```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::CENTRAL_ACCOUNT_ID:role/pr-scan-codebuild-role"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "pr-scan-cross-account"
    }
  }
}
```

---

## 6. Security Scanning Flow

### 6.1 Buildspec Phases

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INSTALL PHASE                                 │
│                                                                      │
│   1. Fetch SonarQube credentials from Secrets Manager                │
│   2. Test SonarQube connectivity (HTTP 200)                          │
│   3. Validate required environment variables                         │
│   4. Check allowed branches (SSM Parameter)                          │
│   5. Install tools:                                                  │
│      - yq (YAML parser)                                              │
│      - SonarScanner 8.1.0 (for SonarQube 10.x+)                     │
│                                                                      │
│   Note: No pip install needed (uses AWS credential-helper)           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       PRE-BUILD PHASE                                │
│                                                                      │
│   1. Load onboarded-repos.json                                       │
│   2. Check if repository is onboarded                                │
│      ├── If NOT onboarded → EXIT with clear message                  │
│      └── If onboarded → Continue                                     │
│   3. Load configuration from metadata:                               │
│      - appName, sonarProject, lang, bu, sbu, qualityGate             │
│   4. Assume role in App Account                                      │
│   5. Clone repository using AWS credential-helper                    │
│   6. Set Quality Gate (if specified in metadata)                     │
│   7. Print configuration summary                                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         BUILD PHASE                                  │
│                                                                      │
│   1. Run SonarQube Scanner                                           │
│      sonar-scanner \                                                 │
│        -Dsonar.projectKey=$PROJECT_KEY \                             │
│        -Dsonar.host.url=$SONAR_HOST_URL \                            │
│        -Dsonar.token=$SONAR_TOKEN \                                  │
│        -Dsonar.qualitygate.wait=true \                               │
│        -Dsonar.branch.name=$SOURCE_BRANCH                            │
│                                                                      │
│   2. Wait for analysis to complete (20 seconds)                      │
│                                                                      │
│   3. Fetch results from SonarQube API:                               │
│      - Quality Gate status                                           │
│      - Bugs (with severity breakdown)                                │
│      - Vulnerabilities (with severity breakdown)                     │
│      - Secrets (vulnerability + secret tag)                          │
│      - Code Smells (with severity breakdown)                         │
│      - Security Hotspots                                             │
│      - Dependencies count (Enterprise)                               │
│                                                                      │
│   4. Save raw API responses to evidence files                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       POST-BUILD PHASE                               │
│                                                                      │
│   1. Determine overall status based on Quality Gate:                 │
│      - OK → PASSED (SAFE_TO_MERGE)                                   │
│      - ERROR → FAILED (REVIEW_REQUIRED)                              │
│      - Other → WARNING (REVIEW_RECOMMENDED)                          │
│                                                                      │
│   2. Generate PR comment (Markdown)                                  │
│                                                                      │
│   3. Generate pr-scan-report.json (merged insights + comment)        │
│                                                                      │
│   4. Post PR comment to CodeCommit                                   │
│                                                                      │
│   5. Upload evidence to S3:                                          │
│      - pr-scan-report.json                                           │
│      - sast/bugs-report.json                                         │
│      - sast/vulnerabilities-report.json                              │
│      - sast/code-smells-report.json                                  │
│      - sast/hotspots-report.json                                     │
│      - secrets/secrets-report.json                                   │
│                                                                      │
│   6. Emit EventBridge event (scan completion)                        │
│                                                                      │
│   7. AUTO-APPROVAL (v3.3):                                           │
│      - If Quality Gate OK → Approve PR with SecurityScanner          │
│      - If Quality Gate ERROR/OTHER → No approval                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 Quality Gate Decision Flow

```
                    SonarQube Quality Gate
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
           ┌─────┐      ┌───────┐     ┌───────┐
           │ OK  │      │ ERROR │     │ OTHER │
           └──┬──┘      └───┬───┘     └───┬───┘
              │             │             │
              ▼             ▼             ▼
        ┌─────────┐   ┌─────────┐   ┌─────────┐
        │ PASSED  │   │ FAILED  │   │ WARNING │
        │         │   │         │   │         │
        │ SAFE_TO │   │ REVIEW_ │   │ REVIEW_ │
        │ _MERGE  │   │ REQUIRED│   │ RECOMM. │
        └─────────┘   └─────────┘   └─────────┘
```

**Important:** The buildspec does NOT apply any additional thresholds. Quality Gate pass/fail is 100% controlled by SonarQube.

---

## 7. Data Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              DATA FLOW                                        │
│                                                                               │
│  ┌────────────┐   PR Event    ┌────────────┐   Forward    ┌────────────┐    │
│  │            │ ───────────▶  │            │ ───────────▶ │            │    │
│  │ CodeCommit │               │ EventBridge│              │ Event Bus  │    │
│  │ (App Acct) │               │ Rule       │              │ (Central)  │    │
│  │            │               │ (App Acct) │              │            │    │
│  └────────────┘               └────────────┘              └─────┬──────┘    │
│                                                                  │           │
│                                                                  │ Trigger   │
│                                                                  ▼           │
│  ┌────────────┐   Clone       ┌────────────┐   Fetch      ┌────────────┐    │
│  │            │ ◀──────────── │            │ ◀──────────  │            │    │
│  │ CodeCommit │               │ CodeBuild  │              │ EventBridge│    │
│  │ (App Acct) │               │ (Central)  │              │ Rule       │    │
│  │            │               │            │              │ (Central)  │    │
│  └────────────┘               └─────┬──────┘              └────────────┘    │
│                                     │                                        │
│                                     │ Scan                                   │
│                                     ▼                                        │
│  ┌────────────┐   Results     ┌────────────┐                                │
│  │            │ ◀──────────── │            │                                │
│  │ SonarQube  │               │ SonarQube  │                                │
│  │ API        │ ───────────▶  │ Scanner    │                                │
│  │            │   Upload      │            │                                │
│  └────────────┘               └─────┬──────┘                                │
│                                     │                                        │
│                    ┌────────────────┼────────────────┐                      │
│                    │                │                │                      │
│                    ▼                ▼                ▼                      │
│             ┌────────────┐   ┌────────────┐   ┌────────────┐                │
│             │            │   │            │   │            │                │
│             │ S3 Bucket  │   │ CodeCommit │   │ EventBridge│                │
│             │ (Evidence) │   │ PR Comment │   │ (Default)  │                │
│             │            │   │            │   │            │                │
│             └────────────┘   └────────────┘   └────────────┘                │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. IAM Permissions

### 8.1 Central Account Roles

**CodeBuild Service Role:** `pr-scan-codebuild-role`

| Permission | Resource |
|------------|----------|
| `logs:CreateLogStream`, `logs:PutLogEvents` | CloudWatch Logs |
| `ec2:CreateNetworkInterface`, `ec2:Describe*` | VPC networking |
| `ec2:CreateNetworkInterfacePermission` | VPC (with ArnLike condition) |
| `ssm:GetParameter` | SSM Parameters `/pr-scan/*` |
| `secretsmanager:GetSecretValue` | SonarQube secret |
| `s3:PutObject`, `s3:GetObject`, `s3:ListBucket` | Evidence bucket |
| `codecommit:GitPull`, `codecommit:GetRepository` | Buildspec repository |
| `sts:AssumeRole` | App Account CodeCommit roles |
| `events:PutEvents` | Default Event Bus |

**EventBridge Role:** `pr-scan-eventbridge-role`

| Permission | Resource |
|------------|----------|
| `codebuild:StartBuild` | CodeBuild project |

### 8.2 App Account Roles

**EventBridge Forwarder:** `pr-scan-eventbridge-forwarder-role`

| Permission | Resource |
|------------|----------|
| `events:PutEvents` | Central Event Bus |

**CodeCommit Access:** `PR-Scan-CodeCommit-Access-Role`

| Permission | Resource |
|------------|----------|
| `codecommit:GitPull` | All CodeCommit repos |
| `codecommit:GetRepository`, `codecommit:GetBranch` | All CodeCommit repos |
| `codecommit:GetPullRequest` | All CodeCommit repos |
| `codecommit:PostCommentForPullRequest` | All CodeCommit repos |
| `codecommit:UpdatePullRequestApprovalState` | All CodeCommit repos (v3.3) |

---

## 8.3 Auto-Approval (v3.3)

### How Auto-Approval Works

When a PR passes the SonarQube Quality Gate, the buildspec automatically approves the PR using the same `PR-Scan-CodeCommit-Access-Role`.

```
┌───────────────────────────────────────────────────────────────────┐
│                    AUTO-APPROVAL FLOW (v3.3)                       │
│                                                                    │
│   ┌──────────────┐        ┌──────────────┐        ┌────────────┐  │
│   │   SonarQube  │───────▶│   CodeBuild  │───────▶│ CodeCommit │  │
│   │   Scanner    │        │   (Check QG) │        │    PR      │  │
│   └──────────────┘        └──────┬───────┘        └────────────┘  │
│                                  │                                 │
│                    ┌─────────────┼─────────────┐                  │
│                    │             │             │                  │
│                    ▼             ▼             ▼                  │
│              ┌─────────┐   ┌─────────┐   ┌─────────┐              │
│              │   OK    │   │  ERROR  │   │  OTHER  │              │
│              │ (PASS)  │   │ (FAIL)  │   │ (WARN)  │              │
│              └────┬────┘   └────┬────┘   └────┬────┘              │
│                   │             │             │                   │
│                   ▼             ▼             ▼                   │
│              ┌─────────┐   ┌─────────┐   ┌─────────┐              │
│              │ APPROVE │   │ NO      │   │ NO      │              │
│              │ PR ✅   │   │ ACTION  │   │ ACTION  │              │
│              └─────────┘   └─────────┘   └─────────┘              │
│                                                                    │
└───────────────────────────────────────────────────────────────────┘
```

### Approval Rule Template

The CloudFormation template creates an **Approval Rule Template** that defines who can approve PRs:

```json
{
  "Version": "2018-11-08",
  "Statements": [{
    "Type": "Approvers",
    "NumberOfApprovalsNeeded": 1,
    "ApprovalPoolMembers": [
      "arn:aws:sts::ACCOUNT:assumed-role/PR-Scan-CodeCommit-Access-Role/SecurityScanner"
    ]
  }]
}
```

**Key Points:**
- Uses fixed session name `SecurityScanner` (not wildcard)
- Same role used for clone, comment, and approval
- Template must be associated with each repository

### Session Name Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  CodeBuild assumes role with session name "SecurityScanner":        │
│                                                                      │
│  aws sts assume-role \                                               │
│    --role-arn "arn:aws:iam::APP_ACCOUNT:role/PR-Scan-CodeCommit-... │
│    --role-session-name "SecurityScanner" \                           │
│    --external-id "pr-scan-cross-account"                             │
│                                                                      │
│  Resulting principal ARN:                                            │
│  arn:aws:sts::APP_ACCOUNT:assumed-role/PR-Scan-CodeCommit.../        │
│                                         SecurityScanner              │
│                                                                      │
│  This matches ApprovalPoolMembers in the Approval Rule Template      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Associate Template with Repositories

The CloudFormation stack automatically creates the Approval Rule Template via Lambda Custom Resource.

After deployment, associate the template with repositories:

```bash
# Single repository
aws codecommit associate-approval-rule-template-with-repository \
  --approval-rule-template-name "pr-scan-SecurityScanApproval" \
  --repository-name "your-repo-name"

# Multiple repositories
aws codecommit batch-associate-approval-rule-template-with-repositories \
  --approval-rule-template-name "pr-scan-SecurityScanApproval" \
  --repository-names "repo1" "repo2" "repo3"

# List all repositories
aws codecommit list-repositories --query 'repositories[].repositoryName'
```

> **Note:** The Approval Rule Template is created automatically during stack deployment. Only the association step is manual.

### Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `EnableAutoApproval` | `true` | Create approval rule template (CloudFormation) |
| `ApprovalSessionName` | `SecurityScanner` | Session name for approvals (CloudFormation) |
| `ENABLE_AUTO_APPROVAL` | `true` | Enable approval in buildspec |
| `APPROVAL_SESSION_NAME` | `SecurityScanner` | Session name in buildspec |

### Disable Auto-Approval

**Option 1: CloudFormation** - Set `EnableAutoApproval=false` (no template created)  
**Option 2: Buildspec** - Set `ENABLE_AUTO_APPROVAL=false` (template exists but not used)

---

## 9. Configuration Management

### 9.1 SSM Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `/pr-scan/allowed-branches` | `main,uat,master` | Branches that trigger scans |

### 9.2 Secrets Manager

**Secret:** `pr-scan/sonarqube`

| Key | Description |
|-----|-------------|
| `token` | SonarQube User Token (must be `squ_` prefix) |
| `host_url` | SonarQube server URL (e.g., `http://YOUR_SONARQUBE_IP:9000`) |

---

## 10. Onboarding Process

### 10.1 Repository Onboarding

Repositories are onboarded via `onboarded-repos.json` in the buildspec repository:

```json
{
  "version": "3.2",
  "description": "PR Security Scan - Onboarded Repositories",
  "repositories": {
    "super-app-be": {
      "appName": "super-app-be Application",
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
    }
  }
}
```

### 10.2 Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `appName` | No | Display name in PR comments |
| `sonarProject` | No | SonarQube project key (cost optimization) |
| `lang` | **Recommended** | Language: `nodejs`, `java-maven`, `java-gradle`, `python`, `go`, `dotnet`, `generic` |
| `bu` | No | Business Unit |
| `sbu` | No | Sub-Business Unit |
| `qualityGate` | No | SonarQube Quality Gate name |

### 10.3 Onboarding Steps

1. Clone the buildspec repository
2. Edit `onboarded-repos.json`
3. Add the repository entry with configuration
4. Commit and push
5. Create a PR in the target repository to trigger scan

### 10.4 Not Onboarded Behavior

If a repository is not in `onboarded-repos.json`, the scan exits with a clear message:

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

## 11. Security Considerations

### 11.1 Credential Management

| Credential | Storage | Access |
|------------|---------|--------|
| SonarQube Token | Secrets Manager | CodeBuild role only |
| Cross-Account Access | IAM Role + External ID | Scoped to specific actions |
| Temporary Credentials | STS AssumeRole | 1-hour validity |

### 11.2 Network Security

| Control | Implementation |
|---------|----------------|
| VPC Isolation | CodeBuild runs in private subnets |
| Egress Control | Security Group limits outbound traffic |
| No Inbound | CodeBuild has no inbound rules |
| NAT Gateway | All internet traffic routed through NAT |

### 11.3 Data Protection

| Data | Protection |
|------|------------|
| Evidence in S3 | AES-256 encryption, versioning enabled |
| CloudWatch Logs | 30-day retention |
| SonarQube Communication | Token-based authentication |

---

## 12. Evidence Storage

### 12.1 S3 Structure

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

### 12.2 Main Output File: pr-scan-report.json

```json
{
  "schemaVersion": "3.4",
  "metadata": {
    "correlationId": "super-app-be-PR814-20261117T064701Z",
    "appId": "super-app-be",
    "appName": "super-app-be Application",
    "bu": "Technology",
    "sbu": "Platform Engineering",
    "repository": "super-app-be",
    "prId": "814",
    "sourceBranch": "pr-raise-request",
    "destBranch": "dev",
    "commit": "12f46f42aa26f2e75f351e3ab703e1c968c3ef89",
    "commitShort": "12f46f42",
    "language": "nodejs",
    "timestamp": "2026-11-17T06:47:58Z",
    "duration": "0m 56s",
    "sonarProject": "super-app-be",
    "qualityGate": "Sonar way"
  },
  "result": {
    "overallStatus": "PASSED",
    "mergeRecommendation": "SAFE_TO_MERGE",
    "qualityGateStatus": "OK"
  },
  "findings": {
    "bugs": {
      "total": 8,
      "blocker": 0,
      "critical": 0,
      "major": 8,
      "minor": 0
    },
    "vulnerabilities": {
      "total": 0,
      "blocker": 0,
      "critical": 0,
      "major": 0,
      "minor": 0
    },
    "secrets": {
      "total": 0,
      "blocker": 0,
      "critical": 0,
      "major": 0,
      "minor": 0
    },
    "codeSmells": {
      "total": 74,
      "blocker": 0,
      "critical": 3,
      "major": 15,
      "minor": 56
    },
    "securityHotspots": {
      "total": 4
    },
    "dependencies": {
      "count": 0,
      "sbom": "Available in SonarQube UI"
    }
  },
  "links": {
    "dashboard": "http://YOUR_SONARQUBE_IP:9000/dashboard?id=my-app&branch=feature-branch",
    "issues": "http://YOUR_SONARQUBE_IP:9000/project/issues?id=my-app&branch=feature-branch",
    "hotspots": "http://YOUR_SONARQUBE_IP:9000/security_hotspots?id=my-app&branch=feature-branch"
  },
  "prComment": "## 🔒 Security Scan Report\n\n..."
}
```

### 12.3 Evidence Retention

| Policy | Value |
|--------|-------|
| Expiration | 90 days |
| Noncurrent Version Expiration | 30 days |
| Versioning | Enabled |

---

## 13. Cost Analysis

### 13.1 AWS Costs

| Service | Estimated Monthly Cost |
|---------|------------------------|
| CodeBuild | ~$50-100 (depends on scan frequency) |
| S3 Evidence | ~$5-10 |
| EventBridge | ~$1-2 |
| Secrets Manager | ~$1 |
| **Total AWS** | **~$60-115/month** |

### 13.2 SonarQube Cost Optimization

Using `sonarProject` to share projects across accounts:

| Without Sharing | With Sharing |
|-----------------|--------------|
| payment-api-111111 (UAT) | payment-api |
| payment-api-222222 (PROD) | payment-api |
| LOC counted twice | LOC counted once |

---

## 14. Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Repo not scanned | Not in onboarded-repos.json | Add entry to metadata file |
| "NOT ONBOARDED" message | Repo name mismatch | Ensure exact repo name match |
| Quality Gate not applied | QG name doesn't exist | Verify QG exists in SonarQube |
| PR comment not posted | IAM role trust issue | Check PR-Scan-CodeCommit-Access-Role |
| SonarQube connection failed | Network issue | Check Security Group egress rules |
| Authentication failed (401) | Wrong token type | Use User Token (`squ_`), not `sqa_` |
| Dependencies = 0 | Community Edition | Normal - SCA is Enterprise feature |
| Scanner fails | Token expired | Regenerate User Token in SonarQube |

---

## 15. Appendix

### 15.1 Accounts Reference

| Account | ID | Purpose |
|---------|-----|---------|
| Central | `YOUR_CENTRAL_ACCOUNT_ID` | CodeBuild, EventBridge, S3 |
| Application | `YOUR_APP_ACCOUNT_ID` | CodeCommit repositories |

### 15.2 SonarQube Servers

| Environment | IP | Port | Edition |
|-------------|-----|------|---------|
| Production | `YOUR_SONARQUBE_IP` | 9000 | Enterprise |

### 15.3 Key URLs

| Resource | URL |
|----------|-----|
| SonarQube Dashboard | `http://YOUR_SONARQUBE_IP:9000/dashboard?id={project}` |
| SonarQube Issues | `http://YOUR_SONARQUBE_IP:9000/project/issues?id={project}` |
| SonarQube Hotspots | `http://YOUR_SONARQUBE_IP:9000/security_hotspots?id={project}` |

### 15.4 Tool Versions

| Tool | Version |
|------|---------|
| SonarScanner | 8.1.0.6389 |
| yq | 4.40.5 |
| CodeBuild Image | amazonlinux2-x86_64-standard:5.0 |

---

**Author:** Noventiq  
**Version:** 3.4  
**Last Updated:** November 2026
