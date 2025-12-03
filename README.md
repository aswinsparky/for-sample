# Security Scan Pipeline - Complete Setup Guide

## Table of Contents

1. Overview
2. Branching Strategy
3. Branch Setup Instructions
4. Protected Branches
5. Pull Request Workflow
6. Security Scans Overview
7. SonarQube Configuration
8. Terraform Structure & Configuration
9. Workflow Comments & Their Purpose
10. Report Generation & Storage
11. AWS Configuration
12. Troubleshooting

---

## Overview

This repository uses a robust DevSecOps CI/CD pipeline powered by GitHub Actions. Every Pull Request triggers automated security scans for code, Dockerfiles, and Terraform infrastructure, ensuring high code quality and security before merging.

**Tools Used:**
- Bandit (Python SAST)
- SonarQube (code quality & security)
- Trivy (container & config scanning)
- Hadolint (Dockerfile linting)
- Checkov (Terraform IaC scanning)
- AWS (OIDC authentication)

---

## Branching Strategy

We use a GitFlow-inspired model for environment isolation and quality gates:

```
main (Production)
  ↑
uat (User Acceptance Testing)
  ↑
develop (Development)
  ↑
feature/* (Feature Development)
```

---

## Branch Setup Instructions

**Branch Creation Order:**

1. **Main Branch (Production)**
   ```bash
   git checkout main
   git pull origin main
   ```
2. **UAT Branch (from Main)**
   ```bash
    git checkout main
    git checkout -b uat
    git push origin uat
   ```
3. **Develop Branch (from UAT)**
   ```bash
    git checkout uat
    git checkout -b develop
    git push origin develop
   ```
4. **Feature Branch (from Develop)**
   ```bash
    git checkout develop
    git checkout -b feature/your-feature-name
    git push origin feature/your-feature-name
   ```

**Feature branches** must start with `feature/`.

---

## Protected Branches

Branches `main`, `uat`, and `develop` are protected:
- No direct commits
- Require PRs and security scans
- Require code review and status checks

Configure these in GitHub repository settings under **Branches**.

---

## Pull Request Workflow

- **feature/* → develop**: For new features (uses `dev.tfvars`)
- **develop → uat**: For UAT testing (uses `dev.tfvars`)
- **uat → main**: For production (uses `prod.tfvars`)

Security scans run automatically on PRs between these branches.

---

## Security Scans Overview

**Scans performed:**
- **SonarQube**: Code quality and vulnerabilities
- **Bandit**: Python security issues
- **Trivy**: Dockerfile misconfigurations (HIGH/CRITICAL)
- **Hadolint**: Dockerfile linting
- **Checkov**: Terraform security (if `Terraform/` exists)

Each tool generates a JSON report and posts inline comments and a summary to the PR.

---

## SonarQube Configuration

Create a `sonar-project.properties` file in the repository root with:

```properties
sonar.projectKey=your-project-key
sonar.host.url=http://your-sonarqube-server:9000
sonar.token=your-sonarqube-token
sonar.sources=.
```

- **sonar.projectKey**: Unique key for your project
- **sonar.host.url**: SonarQube server URL
- **sonar.token**: Token from SonarQube UI (My Account → Security)
- **sonar.sources**: Source directory (usually `.`)

---

## Workflow Comments & Their Purpose

- **Inline Comments**: Posted on code lines with issues (severity, tool, description, link)
- **Summary Comment**: Full report with pass/fail counts and details, updated automatically (marker: `<!-- security-scan-full-report -->`)

---

## Report Generation & Storage

**Generated files:**
- `sonar_issues.json`
- `bandit-report.json`
- `trivy-report.json`
- `hadolint-report.json`
- `Terraform/checkov-report.json`
- `report.md`
- `all_findings.json` (temporary)

Reports are posted to PRs and available as workflow artifacts.

---

## AWS Configuration

**IAM Role ARN**: `arn:aws:iam::302263040839:role/Githubactions`
**Region**: `ap-south-1`

**Trust Policy**: Add your repo and branches to the OIDC trust relationship.

**Permissions**: Role should allow S3 upload and any resources managed by Terraform.

---

## Troubleshooting

- **SonarQube errors**: Check URL, token, and project key
- **Dockerfile issues**: Ensure file exists and is named correctly
- **Terraform skipped**: Ensure `Terraform/` folder exists
- **AWS auth errors**: Verify IAM role and trust policy
- **Rate limits**: Workflow retries automatically
- **Workflow not triggering**: Check PR direction and workflow file location

---

## Quick Reference

- Create branches in order: main → uat → develop → feature/*
- Configure `sonar-project.properties` with your key, name, and token
- Ensure all required files and folders exist
- Review PR comments and summary for security issues

---

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [SonarQube Documentation](https://docs.sonarqube.org/)
- [Bandit Documentation](https://bandit.readthedocs.io/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Hadolint Documentation](https://github.com/hadolint/hadolint)
- [Checkov Documentation](https://www.checkov.io/)

---

This repository follows best practices for secure, compliant, and maintainable infrastructure code.
