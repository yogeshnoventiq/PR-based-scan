# DevSecOps - PR Security Scanning
## Customer Demonstration Guide

**Prepared by:** Noventiq  
**Version:** 3.3  
**Date:** November 2026

---

## Executive Summary

This document demonstrates the **PR-Based Security Scanning** solution - an automated security scanning system that integrates seamlessly with your existing development workflow without requiring any changes to your 500+ pipelines.

### Key Value Propositions

| Benefit | Description |
|---------|-------------|
| 🚀 **Zero Pipeline Changes** | Works independently of existing CI/CD pipelines |
| ⚡ **Quick Account Onboarding** | Deploy CloudFormation template and done |
| 📝 **Simple Repository Onboarding** | Add one line to a JSON file |
| 🔒 **Shift-Left Security** | Catch vulnerabilities before code merges |
| 📊 **Developer Friendly** | Results posted directly as PR comments |
| 🏢 **Cross-Account Support** | Single solution for all AWS accounts |
| 💰 **Cost Optimized** | Shared SonarQube projects reduce licensing costs |

---

## How It Works

### Complete Flow

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    PR SECURITY SCANNING FLOW                                      │
│                                                                                                   │
│  ┌─────────────────────────────────────────┐    ┌─────────────────────────────────────────────┐  │
│  │        APPLICATION ACCOUNT(S)           │    │              CENTRAL ACCOUNT                 │  │
│  │                                         │    │              (CENTRAL_ACCOUNT_ID)                  │  │
│  │                                         │    │                                              │  │
│  │   ┌───────────────┐                     │    │    ┌─────────────────┐                      │  │
│  │   │  CodeCommit   │ ①                   │    │    │   Event Bus     │                      │  │
│  │   │  Repository   │ Developer           │    │    │  (pr-scan)      │                      │  │
│  │   │              ─┼─creates PR          │    │    └────────┬────────┘                      │  │
│  │   └───────┬───────┘                     │    │             │                               │  │
│  │           │                             │    │             │ ③ Triggers                    │  │
│  │           │ ② PR Event                  │    │             ▼                               │  │
│  │           ▼                             │    │    ┌─────────────────┐   ┌──────────────┐   │  │
│  │   ┌───────────────┐    Cross-Account    │    │    │  EventBridge    │   │ CodeCommit   │   │  │
│  │   │  EventBridge  │ ──────────────────────────▶  │  Rule           │   │ (buildspecs) │   │  │
│  │   │  Rule         │                     │    │    └────────┬────────┘   │              │   │  │
│  │   └───────────────┘                     │    │             │            │ onboarded-   │   │  │
│  │                                         │    │             │ ④ Starts   │ repos.json   │   │  │
│  │                                         │    │             ▼            └───────┬──────┘   │  │
│  │   ┌───────────────┐    ⑤ Clone Code     │    │    ┌─────────────────┐          │          │  │
│  │   │  IAM Role     │ ◀─────────────────────────── │    CodeBuild     │ ◀────────┘          │  │
│  │   │  (Access)     │                     │    │    │    Project      │  Reads config       │  │
│  │   └───────────────┘                     │    │    │                 │                     │  │
│  │                                         │    │    │  ┌───────────┐  │                     │  │
│  │                                         │    │    │  │ Run Scan  │──────▶ SonarQube      │  │
│  │   ┌───────────────┐    ⑧ Post Comment   │    │    │  │           │  │ ⑥   (EC2)          │  │
│  │   │   PR Comment  │ ◀─────────────────────────── │  │ Get Result│◀──────               │  │
│  │   │   Posted      │                     │    │    │  └───────────┘  │ ⑦                   │  │
│  │   └───────────────┘                     │    │    │                 │                     │  │
│  │                                         │    │    │  Upload to S3 ──────▶ S3 Bucket      │  │
│  └─────────────────────────────────────────┘    │    │  (Evidence)     │ ⑨   (Audit)        │  │
│                                                  │    └─────────────────┘                     │  │
│                                                  │                                              │  │
│                                                  └──────────────────────────────────────────────┘  │
│                                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Flow

| Step | Component | What Happens |
|------|-----------|--------------|
| ① | **Developer** | Creates or updates Pull Request |
| ② | **EventBridge Rule** (App) | Catches PR event, forwards to Central |
| ③ | **Event Bus** (Central) | Receives event, triggers CodeBuild |
| ④ | **CodeBuild** | Checks if repo is onboarded in JSON |
| ⑤ | **CodeBuild** | Clones repository from App Account |
| ⑥ | **SonarQube** | Performs SAST, SCA, Secrets scan |
| ⑦ | **CodeBuild** | Gets scan results from SonarQube |
| ⑧ | **CodeBuild** | Posts security report as PR comment |
| ⑨ | **S3 Bucket** | Stores scan evidence for audit |

---

## How to Onboard

### Account Onboarding (One-Time per Account)

Deploy CloudFormation template in the Application Account:

```bash
aws cloudformation deploy \
  --template-file app-account-resources.yaml \
  --stack-name pr-scan-app \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    CentralAccountId="YOUR_CENTRAL_ACCOUNT_ID" \
    CentralEventBusArn="arn:aws:events:ap-south-1:YOUR_CENTRAL_ACCOUNT_ID:event-bus/pr-scan-events"
```

**Creates:**
- EventBridge Rule (forwards PR events to Central)
- IAM Role (allows Central to clone code and post comments)

✅ Account is ready! All repos in this account can now be onboarded.

---

### Repository Onboarding (Per Repository)

**Just add one entry to `onboarded-repos.json`:**

```json
{
  "version": "3.2",
  "repositories": {
    "my-new-repo": {
      "appName": "My Application",
      "sonarProject": "my-new-repo",
      "lang": "nodejs",
      "bu": "Technology",
      "qualityGate": "Sonar way"
    }
  }
}
```

**Then commit and push:**

```bash
git add onboarded-repos.json
git commit -m "Onboard my-new-repo"
git push
```

✅ Repository onboarded! Next PR will be automatically scanned.

---

### Repository Configuration Options

| Field | Required | Description | Example |
|-------|----------|-------------|---------|
| `appName` | No | Display name in PR comments | `"Payment API"` |
| `sonarProject` | No | SonarQube project key (for cost optimization) | `"payment-api"` |
| `lang` | Recommended | Programming language | `"nodejs"`, `"java-maven"`, `"python"` |
| `bu` | No | Business Unit | `"Technology"` |
| `sbu` | No | Sub-Business Unit | `"Platform"` |
| `qualityGate` | No | Quality Gate profile | `"Sonar way"` |

### Supported Languages

| Language | Value |
|----------|-------|
| Node.js/JavaScript | `nodejs` |
| Java (Maven) | `java-maven` |
| Java (Gradle) | `java-gradle` |
| Python | `python` |
| Go | `go` |
| .NET/C# | `dotnet` |

---

## What Developers See

### PR Comment - Quality Gate PASSED

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   ## 🔒 Security Scan Report                                                 │
│                                                                              │
│   **Application:** Payment Gateway API                                       │
│   **PR:** #814 (`feature/new-flow` → `main`)                                │
│                                                                              │
│   ### Result: ✅ PASSED                                                      │
│                                                                              │
│   | Metric               | Status          |                                 │
│   |----------------------|-----------------|                                 │
│   | Quality Gate         | ✅ OK           |                                 │
│   | Merge Recommendation | 🟢 SAFE TO MERGE|                                 │
│                                                                              │
│   | Category         | Total | Blocker | Critical | Major | Minor |         │
│   |------------------|-------|---------|----------|-------|-------|         │
│   | Bugs             |   2   |    0    |    0     |   2   |   0   |         │
│   | Vulnerabilities  |   0   |    0    |    0     |   0   |   0   |         │
│   | Secrets          |   0   |    0    |    0     |   0   |   0   |         │
│   | Code Smells      |  15   |    0    |    1     |   8   |   6   |         │
│                                                                              │
│   🔗 [View in SonarQube](http://sonarqube:9000/dashboard?id=payment-api)     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### PR Comment - Quality Gate FAILED

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   ## 🔒 Security Scan Report                                                 │
│                                                                              │
│   **Application:** Customer Portal                                           │
│   **PR:** #256 (`feature/login-update` → `main`)                            │
│                                                                              │
│   ### Result: ❌ FAILED                                                      │
│                                                                              │
│   | Metric               | Status              |                             │
│   |----------------------|---------------------|                             │
│   | Quality Gate         | ❌ FAILED           |                             │
│   | Merge Recommendation | 🔴 REVIEW REQUIRED  |                             │
│                                                                              │
│   | Category         | Total | Blocker | Critical | Major | Minor |         │
│   |------------------|-------|---------|----------|-------|-------|         │
│   | Bugs             |   3   |    1    |    0     |   2   |   0   |         │
│   | Vulnerabilities  |   5   |    1    |    2     |   2   |   0   |         │
│   | Secrets          |   2   |    2    |    0     |   0   |   0   |         │
│                                                                              │
│   ⚠️ This PR has security issues that must be resolved before merging       │
│                                                                              │
│   🔗 [View Details in SonarQube](http://sonarqube:9000/dashboard?id=...)     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Important Scenarios

### Repository NOT Onboarded

When PR is created in a repository **not listed** in `onboarded-repos.json`:

| What Happens | Result |
|--------------|--------|
| SonarQube | ❌ NOT called (saves resources) |
| PR Comment | ❌ NOT posted (no noise) |
| Build Status | ✅ SUCCESS (graceful skip) |
| Logs | Clear instructions on how to onboard |

**CodeBuild Log Message:**
```
[SKIP] Repository 'unknown-repo' is NOT onboarded

To onboard, add entry to onboarded-repos.json:
  "unknown-repo": {
    "appName": "Display Name",
    "lang": "nodejs",
    "qualityGate": "Sonar way"
  }
```

---

### Branch Filtering

Only PRs to **allowed branches** are scanned (default: `main, uat, master`):

| PR Target | Scanned? |
|-----------|----------|
| `feature/login` → `main` | ✅ Yes |
| `feature/payment` → `uat` | ✅ Yes |
| `hotfix/urgent` → `master` | ✅ Yes |
| `feature/login` → `develop` | ❌ No (skipped) |

**Why?** Security scans happen only at critical merge points (to main branches), not for every feature-to-feature merge.

---

### Cost Optimization - Same Repo in Multiple Accounts

When same repository name exists in multiple accounts (UAT & PROD), use `sonarProject` to consolidate:

```json
{
  "payment-api": {
    "appName": "Payment Gateway API",
    "sonarProject": "payment-api",    // ← SHARED PROJECT KEY
    "lang": "java-gradle"
  }
}
```

**Result:**
- Single entry works for ANY account with repo named `payment-api`
- Lines of Code (LOC) counted once, not per account
- **50% reduction in SonarQube licensing cost**

---

## What Gets Scanned

| Scan Type | What It Detects |
|-----------|-----------------|
| **SAST** | Code bugs, vulnerabilities, code smells |
| **SCA** | Vulnerable dependencies, CVEs |
| **Secrets** | Hardcoded passwords, API keys, tokens |
| **Security Hotspots** | Potential issues needing review |
| **Quality Gate** | Overall code quality (controlled by Security Team) |

---

## Key Benefits

| For | Benefits |
|-----|----------|
| **Developers** | Zero workflow changes, instant feedback in PR, clear guidance on what to fix |
| **DevOps** | Easy onboarding (just JSON edit), centralized control, no pipeline changes |
| **Security** | Shift-left security, Quality Gate control, audit trail in S3 |
| **Management** | Cost optimized, scalable to 500+ repos, compliance ready |

---

## Quick Reference

### Onboard a Repository

```json
"my-repo": {
  "appName": "My Application",
  "sonarProject": "my-repo",
  "lang": "nodejs",
  "bu": "Technology",
  "qualityGate": "Sonar way"
}
```

### Useful Commands

```bash
# Clone config repo
git clone codecommit::ap-south-1://pr-scan-buildspecs

# List onboarded repos
jq '.repositories | keys[]' onboarded-repos.json

# Check if repo is onboarded
jq '.repositories["my-repo"]' onboarded-repos.json
```

---

**Prepared by:** Noventiq  
**Version:** 3.3

---

*For technical details, refer to:*
- [README.md](../README.md)
- [SOLUTION_DESIGN.md](SOLUTION_DESIGN.md)
- [CONFIG_SCHEMA.md](CONFIG_SCHEMA.md)
