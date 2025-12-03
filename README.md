Here is the complete content of the `README.md` file in a single block.

```markdown
# 🛡️ Security Scan Pipeline - Complete Setup Guide

## 📋 Table of Contents

1. [Overview](#1-overview)
2. [Branching Strategy & Flow](#2-branching-strategy--flow)
3. [Protected Branches & PR Triggers](#3-protected-branches--pr-triggers)
4. [SonarQube Configuration](#4-sonarqube-configuration)
5. [AWS OIDC Configuration](#5-aws-oidc-configuration)
6. [Security Scans Overview](#6-security-scans-overview)
7. [Terraform & Checkov Integration](#7-terraform--checkov-integration)
8. [PR Commenting Strategy](#8-pr-commenting-strategy)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Overview

This GitHub Actions workflow (`security-scan.yml`) establishes a robust **DevSecOps pipeline** for continuous security and quality validation. It integrates multiple leading security scanning tools directly into the Pull Request (PR) workflow, ensuring all code, container images, and infrastructure-as-code (IaC) are scanned before merging.

### Key Features

* **Multi-Tool Scanning**: Runs SonarQube, Bandit, Trivy, Hadolint, and Checkov.
* **Branch-Aware Execution**: The workflow only runs for specific, critical PR paths (`feature/*` to `develop`, and `develop` to `main`).
* **Contextual Reporting**: Posts a full summary report and **inline comments** directly on the lines of code that contain security findings.
* **OIDC Authentication**: Securely authenticates with AWS using IAM Role assumption (OIDC).
* **Terraform Validation**: Includes `terraform plan` and `checkov` scanning, selecting environment variables based on the PR target branch.

---

## 2. Branching Strategy & Flow

The pipeline is built around a restricted branching model to enforce quality gates at critical merge points.

### Branch Hierarchy

The repository structure follows a strict hierarchy, starting from **main** as the most stable environment.

```

main (Production)
↑
uat (User Acceptance Testing)
↑
develop (Development/Integration)
↑
feature/\* (Feature Development)

````

### ⚠️ Step-by-Step Branch Creation

It is **critical** to create the branches in the following order to ensure the correct base and history:

1.  **Create `uat` from `main`:**
    ```bash
    git checkout main
    git pull origin main
    git checkout -b uat
    git push origin uat
    ```
2.  **Create `develop` from `uat`:**
    ```bash
    git checkout uat
    git pull origin uat
    git checkout -b develop
    git push origin develop
    ```
3.  **Create `feature/` branches from `develop`:**
    ```bash
    git checkout develop
    git pull origin develop
    git checkout -b feature/my-new-feature
    git push origin feature/my-new-feature
    ```

### Pull Request Flow (Triggers)

The workflow is configured to run on the following specific PRs:

| Source Branch (`head_ref`) | Target Branch (`base_ref`) | Purpose | Terraform Vars Used |
| :--- | :--- | :--- | :--- |
| `feature/*` | `develop` | New feature integration & **Dev** testing | `dev.tfvars` |
| `develop` | `main` | Promotion to **Production** | `prod.tfvars` |

**Note**: To enable scanning for `develop` $\rightarrow$ `uat` or `uat` $\rightarrow$ `main`, you must update the `if` condition in the workflow file.

---

## 3. Protected Branches & PR Triggers

### Workflow Trigger Condition

The workflow uses an explicit `if` condition to limit runs to the designated critical paths:

```yaml
if: |
  (startsWith(github.head_ref, 'feature/') && github.base_ref == 'develop') || 
  (github.head_ref == 'develop' && github.base_ref == 'main')
````

### GitHub Protection Rules

To enforce this security gate, the following branches **must** be protected in your repository settings:

  * `main`
  * `uat`
  * `develop`

Ensure you enable **"Require status checks to pass before merging"** and select the `security-scan` job.

-----

## 4\. SonarQube Configuration

The workflow uses the `sonar-scanner` CLI to execute the scan, which reads its configuration from a file named `sonar-project.properties` in the repository root.

### `sonar-project.properties` File

You **must** create this file and configure it with your project's unique key, the SonarQube server URL, and an authentication token.

```properties
# sonar-project.properties

# --- REQUIRED SETTINGS ---

# 1. Unique key for your project in SonarQube
sonar.projectKey=CiCd-pipeline-checking-key

# 2. The URL of your SonarQube server
sonar.host.url=[http://13.126.220.187:9000](http://13.126.220.187:9000)

# 3. Authentication Token (GENERATED FROM SONARQUBE UI)
# **SECURITY WARNING**: This token is visible in the workflow logs.
# For production use, this value should be stored as a GitHub Secret.
# (E.g., sonar.token=${{ secrets.SONAR_TOKEN }})
sonar.token=squ_21245d46f6e715cdb9ecbe95acd899b66841e034

# --- OPTIONAL SETTINGS ---

# Source directory to scan (usually current directory)
sonar.sources=.

# Encoding of the source code
sonar.sourceEncoding=UTF-8
```

### Token Security Best Practice

While the example shows the token in the file, for production use, it is **highly recommended** to store your `sonar.token` in a GitHub secret (e.g., `SONAR_TOKEN`) and reference it in the `sonar-project.properties` file using `${{ secrets.SONAR_TOKEN }}`.

-----

## 5\. AWS OIDC Configuration

The workflow uses the `aws-actions/configure-aws-credentials@v4` action to assume an IAM role via **OpenID Connect (OIDC)**, eliminating the need to store AWS access keys in GitHub Secrets.

### Required IAM Role

| Parameter | Value |
| :--- | :--- |
| **Role ARN** | `arn:aws:iam::302263040839:role/Githubactions` |
| **AWS Region** | `ap-south-1` |

### Trust Relationship Update

The IAM Role **`Githubactions`** on account `302263040839` must have its **Trust Policy** updated to allow your GitHub repository to assume the role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::302263040839:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

**Replace `YOUR_ORG/YOUR_REPO` with your actual GitHub repository path.**

-----

## 6\. Security Scans Overview

The pipeline executes five different security scanners in sequence:

| Tool | Focus Area | Command/Action | Report File |
| :--- | :--- | :--- | :--- |
| **SonarQube** | SAST, Code Quality, Bugs | `sonar-scanner` | `sonar_issues.json` |
| **Bandit** | Python SAST | `bandit -r . -f json` | `bandit-report.json` |
| **Hadolint** | Dockerfile Linting | `hadolint --format json Dockerfile` | `hadolint-report.json` |
| **Trivy** | Dockerfile Vulnerabilities | `trivy config Dockerfile` | `trivy-report.json` |
| **Checkov** | IaC Security (Terraform) | `checkov -f tfplan.json` | `Terraform/checkov-report.json` |

-----

## 7\. Terraform & Checkov Integration

The Checkov scan is specifically configured to analyze your Terraform code's execution plan, which provides a more accurate view of the final deployed configuration.

### Folder Structure Requirement

The Terraform code must reside in a folder named `Terraform/` for the scan to be triggered.

```
repo-root/
└── Terraform/
    ├── environments/
    │   ├── dev.tfvars         <-- Used for 'feature/*' -> 'develop' PRs
    │   └── prod.tfvars        <-- Used for 'develop' -> 'main' PRs
    └── ... (your .tf files)
```

### Automatic `.tfvars` Selection

The workflow automatically sets the `TFVARS` environment variable based on the PR's target branch:

| PR Direction | `$TFVARS` Value | Rationale |
| :--- | :--- | :--- |
| `feature/*` $\rightarrow$ `develop` | `dev.tfvars` | Development environment config. |
| `develop` $\rightarrow$ `main` | `prod.tfvars` | Production environment config. |

This variable is then used in the `Terraform Plan` step:

```bash
terraform plan -var-file=./environments/${TFVARS} -out=tfplan.binary
```

-----

## 8\. PR Commenting Strategy

The workflow generates two types of comments to ensure all findings are clear and actionable.

### 1\. Security Scan Summary

A single, comprehensive comment is posted to the PR discussion with a full breakdown of all findings, including pass/fail counts and detailed reports for each tool.

  * **Update Logic**: The comment is tagged with a marker (\`\`). On subsequent runs, the workflow automatically **updates** the existing comment, avoiding comment spam.

### 2\. Inline Review Comments

Individual findings are posted as review comments directly on the line of code that triggered the issue.

  * **Batching**: Comments are sent in batches of **20** with a delay between each batch to prevent GitHub API rate-limiting.
  * **Filtering**: Comments are **filtered** to only appear on lines that are part of the current Pull Request's diff, reducing noise from legacy code issues.

-----

## 9\. Troubleshooting

### ❌ Issue: SonarQube Fails to Connect

  * **Action**: Double-check the `sonar.host.url` and `sonar.token` in your `sonar-project.properties` file.
  * **Action**: Ensure the SonarQube server is running and reachable from GitHub Actions runners.

### ❌ Issue: Checkov Scan Is Skipped

  * **Reason**: The `Terraform/` folder was not found.
  * **Action**: Verify the folder exists and is named exactly `Terraform` (case-sensitive).

### ❌ Issue: Workflow Does Not Run on PR

  * **Reason**: The PR flow does not match the `if` condition.
  * **Action**: Ensure your PR is either `feature/*` $\rightarrow$ `develop` or `develop` $\rightarrow$ `main`.

### ❌ Issue: AWS Authentication Fails

  * **Reason**: IAM Role Assumption failed.
  * **Action**: Verify the **Trust Policy** on the IAM Role `Githubactions` has been updated to include your repository's OIDC Subject (see [AWS OIDC Configuration](https://www.google.com/search?q=%235-aws-oidc-configuration)).

### ❌ Issue: Inline Comments Not Appearing

  * **Reason**: The findings are not on lines changed in the current PR diff (they were filtered out).
  * **Action**: Check the summary report. If the issue is critical, manually review the code line even if the inline comment was skipped.

<!-- end list -->

```
```
