# Security Scan Pipeline - Complete Setup Guide

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

## Advanced Usage & Customization

### Customizing the Workflow
- Add or remove tools by editing the workflow YAML.
- Change scan severity thresholds (e.g., Trivy: `--severity MEDIUM,HIGH,CRITICAL`).
- Upload reports to S3, GCS, or other storage for compliance.

### Example: Enforcing PR Approval
Configure branch protection rules to require at least one approval before merging. This ensures code review and security validation.

---

## Best Practices for Secure Development

- Always address all security findings before merging.
- Use secrets managers for sensitive data (never hardcode passwords).
- Keep dependencies up to date.
- Use least privilege for AWS IAM roles.
- Regularly review and update branch protection rules.
- Document all exceptions and risk acceptances.
- Run scans locally before pushing large changes.
- Use descriptive commit messages for traceability.
- Rotate credentials and tokens regularly.
- Monitor workflow runs and address failures promptly.

---

## Example SonarQube Quality Gate Setup

1. Log in to SonarQube.
2. Go to **Quality Gates**.
3. Create a new gate (e.g., "Strict Security Gate").
4. Add conditions:
   - No new critical vulnerabilities
   - Coverage on new code > 80%
   - Maintainability rating: A
5. Assign the gate to your project.
6. The workflow will fail if the gate is not passed.

---

## Example Checkov Policy Suppression

If you need to suppress a false positive in Checkov, add a comment in your Terraform code:
```hcl
resource "aws_s3_bucket" "example" {
  # checkov:skip=CKV_AWS_18: S3 logging not required for this bucket
  bucket = "my-bucket"
}
```
Document all suppressions in your PR description.

---

## Example Bandit Configuration

You can customize Bandit with a `.bandit` config file:
```ini
[bandit]
skips: B101,B102
exclude: tests/*
```
Run with: `bandit -c .bandit -r .`

---

## Example Trivy Ignore File

Create `.trivyignore` to suppress known issues:
```
GHSA-xxxx-xxxx-xxxx
CVE-2022-12345
```
Place in the root directory.

---

## Example Hadolint Ignore File

Create `.hadolint.yaml`:
```yaml
ignored:
  - DL3008
  - DL3016
```
Place in the root directory.

---

## Example PR Review Process

1. Developer pushes feature branch.
2. PR is created to `develop`.
3. Security scan runs automatically.
4. Inline comments and summary posted.
5. Developer fixes issues and pushes updates.
6. Reviewer approves PR.
7. PR is merged.
8. Repeat for `develop → uat` and `uat → main`.

---

---

## Example Inline Comment

```
🐍 [Bandit][HIGH][B101] Use of assert detected. Assert statements should not be used in production code.
File: app/main.py:42
Link: https://github.com/org/repo/blob/feature/your-feature/app/main.py#L42
```

---

## Example Summary Comment

```
<!-- security-scan-full-report -->
# :shield: Security Scan Report

1️⃣ **Checkov Report**
   - 🟢 **PASSED Checks:** 45
   - 🔴 **FAILED Checks:** 3
   - ⚠️  **SKIPPED Checks:** 2

2️⃣ **SonarQube Report**
   - 🔴 **FAILED Checks:** 12

[... detailed reports ...]
```

---

## Example AWS IAM Policy for Least Privilege

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::your-bucket/reports/*"
    }
  ]
}
```

---

## Example Terraform Plan Command

```bash
terraform plan -var-file=./environments/dev.tfvars -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json
```

---

## Example S3 Upload Step

```yaml
- name: Upload Reports to S3
  run: |
    aws s3 cp report.md s3://your-bucket/reports/${{ github.repository }}/${{ github.run_number }}/report.md
    aws s3 cp sonar_issues.json s3://your-bucket/reports/${{ github.repository }}/${{ github.run_number }}/sonar_issues.json
    aws s3 cp bandit-report.json s3://your-bucket/reports/${{ github.repository }}/${{ github.run_number }}/bandit-report.json
    aws s3 cp trivy-report.json s3://your-bucket/reports/${{ github.repository }}/${{ github.run_number }}/trivy-report.json
    aws s3 cp hadolint-report.json s3://your-bucket/reports/${{ github.repository }}/${{ github.run_number }}/hadolint-report.json
    if [ -f Terraform/checkov-report.json ]; then
      aws s3 cp Terraform/checkov-report.json s3://your-bucket/reports/${{ github.repository }}/${{ github.run_number }}/checkov-report.json
    fi
```

---

## Example Local Scan Script

Create a script `local_scan.sh`:
```bash
#!/bin/bash
set -e
echo "Running Bandit..."
bandit -r . -f json -o bandit-report.json
echo "Running Hadolint..."
hadolint Dockerfile > hadolint-report.json
echo "Running Trivy..."
trivy config --severity HIGH,CRITICAL --format json --output trivy-report.json Dockerfile
echo "Running Checkov..."
checkov -d Terraform/ -o json > Terraform/checkov-report.json
echo "Done. See JSON reports for details."
```

---

## Example PR Template

Create `.github/pull_request_template.md`:
```markdown
    # Pull Request
    
    ## Description
    Describe your changes and why they are needed.
    
    ## Security Checklist
    - [ ] All security scan issues addressed
    - [ ] No hardcoded secrets
    - [ ] All dependencies up to date
    - [ ] Terraform plan reviewed
    - [ ] AWS IAM roles use least privilege
    
    ## Screenshots (if applicable)
    
    ## Additional Notes

```

---

## References & Further Reading

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [GitHub Actions Security Best Practices](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [Terraform Security Best Practices](https://learn.hashicorp.com/tutorials/terraform/security)
- [SonarQube Rules](https://rules.sonarsource.com/)
- [Bandit Test IDs](https://bandit.readthedocs.io/en/latest/plugins.html)
- [Trivy Misconfiguration Docs](https://aquasecurity.github.io/trivy/v0.18.0/docs/misconfiguration/)
- [Hadolint Rules](https://github.com/hadolint/hadolint/wiki/DL3000)
- [Checkov Policies](https://www.checkov.io/1.Policy%20Index.html)

---
