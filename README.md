# Security Scan GitHub Actions Workflow

This repository contains a **Security Scan** GitHub Actions workflow that runs multiple security and quality tools on pull requests and posts a detailed report and inline comments directly on the pull request. [web:5][web:18]

## Workflow Overview

The workflow file should be placed at:

##.github/workflows/security-scan.yml


It is triggered on pull request events (`opened`, `synchronize`, `reopened`) when: [web:5]

- A `feature/*` branch targets the `develop` branch  
- The `develop` branch targets the `main` branch

The `security-scan` job runs on `ubuntu-latest` and performs: [web:5][web:18]

- SonarQube static code analysis using **SonarScanner CLI**
- **Bandit** scan for Python security issues
- **Trivy** Dockerfile misconfiguration scan (HIGH/CRITICAL severities)
- **Hadolint** Dockerfile linting
- **Terraform + Checkov** infrastructure security scanning (if a `Terraform/` folder exists)
- Aggregation of results into:
  - A human-readable `report.md`
  - A machine-readable `all_findings.json` for inline PR comments

## Tools Used in the Pipeline

The workflow installs and runs the following tools:

- **SonarQube (SonarScanner CLI)**  
  Performs static code analysis to detect bugs, vulnerabilities, and code smells, integrated with SonarQube or SonarCloud. [web:5][web:20]

- **Bandit**  
  A security linter for Python that inspects code for common security issues. [web:6]

- **Trivy**  
  A scanner that analyzes Dockerfiles and other targets for vulnerabilities and misconfigurations, here focused on Dockerfile misconfigurations of HIGH and CRITICAL severity. [web:9][web:15]

- **Hadolint**  
  A linter for Dockerfiles enforcing best practices and avoiding common pitfalls. [web:12]

- **Terraform + Checkov**  
  - Runs `terraform init` and `terraform plan` (if `Terraform/` directory exists)  
  - Converts the plan to JSON (`tfplan.json`)  
  - Uses **Checkov** to scan the plan for infrastructure‑as‑code (IaC) security misconfigurations. [web:18]

Each tool produces a JSON report (`sonar_issues.json`, `bandit-report.json`, `trivy-report.json`, `hadolint-report.json`, `Terraform/checkov-report.json`), which is parsed by `jq` to build aggregated results.

## SonarQube Configuration (`sonar-project.properties`)

The workflow expects a `sonar-project.properties` file at the root of the repository. This file configures how SonarScanner connects to SonarQube and identifies the project. [web:5][web:11]

Create a file named:

sonar-project.properties


with at least the following keys:

Unique project key in SonarQube
sonar.projectKey=your-project-key

Human-readable project name
sonar.projectName=Your Project Name

URL of your SonarQube server (or SonarCloud endpoint)
sonar.host.url=https://your-sonarqube-server

Authentication token generated in SonarQube
sonar.token=your-sonar-token


### Required Keys Explained

- **`sonar.projectKey`**  
  A stable, unique identifier for the project in SonarQube; used to associate analyses and history with the same project. [web:11]

- **`sonar.projectName`**  
  Display name shown in the SonarQube UI for easier identification by humans. [web:3]

- **`sonar.host.url`**  
  Base URL of the SonarQube instance (for example, `https://sonar.mycompany.com` or the SonarCloud URL). [web:5]

- **`sonar.token`**  
  An access token generated from your SonarQube user account or project settings, used by the CLI instead of username/password. [web:2][web:8]

The workflow installs **SonarScanner CLI**, adds it to the `PATH`, and simply runs: [web:5][web:20]

1. Start from main
git checkout main
git pull origin main

2. Create UAT from main
git checkout -b UAT
git push origin UAT

3. Create develop from UAT
git checkout UAT
git pull origin UAT
git checkout -b develop
git push origin develop

4. Create a feature branch from develop
git checkout develop
git pull origin develop
git checkout -b feature/your-feature-name
git push origin feature/your-feature-name


### Branch Flow Diagram
```
┌─────────────┐
│   feature/  │  ← Developers work here
│  your-feat  │
└──────┬──────┘
       │ PR (with security scans)
       ↓
┌─────────────┐
│   develop   │  ← Integration & Development Environment
└──────┬──────┘
       │ PR (with security scans)
       ↓
┌─────────────┐
│     uat     │  ← User Acceptance Testing Environment
└──────┬──────┘
       │ PR (with security scans)
       ↓
┌─────────────┐
│    main     │  ← Production Environment
└─────────────┘
```

This ensures that security and quality checks run both at the **feature** level and at the **release** level.

## Reports and PR Feedback

### Generated Files

During the workflow run, the following key files are generated:

- **`report.md`**  
  - High-level summary section listing counts of failed checks for:
    - Checkov
    - SonarQube
    - Bandit
    - Hadolint
    - Trivy  
  - Detailed sections:
    - Checkov Terraform Scan Detailed Report  
    - SonarQube Detailed Report  
    - Bandit Detailed Report  
    - Hadolint Dockerfile Scan Detailed Report  
    - Trivy Dockerfile Scan Detailed Report  
  - Each detailed section lists issues with:
    - Severity  
    - File and line information  
    - Human-readable message  
    - Direct GitHub links to the relevant lines in the repository

- **`all_findings.json`**  
  - A merged array of findings from:
    - Checkov (Terraform)  
    - SonarQube  
    - Bandit  
    - Hadolint  
    - Trivy  
  - Each entry has:
    - `path`: File path relative to the repository root  
    - `line`: Line number in the file  
    - `body`: Markdown message summarizing the issue (tool, severity, ID, description, and in some cases resolution tips)

### Inline PR Comments

The **“Post inline review comments with batching and retry”** step uses `actions/github-script` to: [web:5][web:18]

1. Load `all_findings.json`.  
2. Fetch the list of files and their diffs for the pull request.  
3. Compute valid line numbers from the diff hunks (added/context lines only).  
4. Filter findings so that:
   - Only files present in the PR are considered.  
   - Only lines that exist in the PR diff are commented on.  
5. Batch comments in chunks (e.g., 20 comments at a time) and post them as PR reviews with retry logic for rate limits and transient errors.

### Summary Comment

The **“Post Security Scan Summary to PR (with update/reuse)”** step: [web:5]

- Reads `report.md`.  
- Wraps it with a marker:  

<!-- security-scan-full-report -->


- Searches existing PR comments for this marker.  
- If a comment exists:
- Updates the existing comment with the new report content.  
- If no comment exists:
- Creates a new comment containing the full `report.md`.

This behavior ensures that each push to the pull request **refreshes a single summary comment** instead of filling the PR with multiple duplicate reports, keeping the discussion clean and focused.

## Prerequisites and Notes

- **AWS Credentials**  
The workflow assumes an IAM role (e.g., `arn:aws:iam::302263040839:role/Githubactions`) is available and accessible via GitHub OIDC for Terraform actions. [web:18]

- **Permissions**  
The workflow uses:
- `contents: read`
- `pull-requests: write`
- `id-token: write`  
to allow reading code, posting comments, and assuming the AWS role. [web:5]

- **Dependencies in the Runner**  
The workflow installs the following via `apt`/`pip`:
- `jq`, `curl`, `wget`
- `bandit`
- `trivy`
- `hadolint` (downloaded binary)
- `checkov`  
Ensure that your runner has internet access to fetch these tools from their respective sources. [web:9][web:15][web:18]

With this setup, every eligible pull request is automatically scanned for security and quality issues, and developers receive both a consolidated report and precise inline feedback tied to the changed lines.
