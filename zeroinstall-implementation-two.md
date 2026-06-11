If you want a **real enterprise DevSecOps pipeline**, my recommendation is:

> **Avoid requiring developers to install anything locally.**
>
> Put almost everything inside **GitHub Actions** so every push and pull request is automatically scanned.

This gives you:

```text
Developer
    ↓
git push
    ↓
GitHub Actions
    ↓
Gitleaks (Secrets)
    ↓
Dependency Scan
    ↓
SAST Scan
    ↓
Docker Build
    ↓
Trivy Container Scan
    ↓
Terraform Scan
    ↓
SBOM Generation
    ↓
Artifact Upload
    ↓
Deploy
```

---

# Recommended Repository Structure

```text
project/
│
├── .github/
│     └── workflows/
│            gitleaks.yml
│            dependency-check.yml
│            sonarqube.yml
│            trivy.yml
│            checkov.yml
│            sbom.yml
│
├── src/
├── Dockerfile
├── package.json
├── package-lock.json
├── sonar-project.properties
├── .gitignore
└── .dockerignore
```

---

# 1. Gitleaks (Secret Detection)

Purpose:

Detect:

* AWS keys
* Passwords
* JWT secrets
* Database passwords
* API keys

---

## .github/workflows/gitleaks.yml

```yaml
name: Secret Scan

on:
  push:
  pull_request:

jobs:
  gitleaks:
    runs-on: ubuntu-latest

    permissions:
      contents: read

    steps:

      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

# 2. Dependency Vulnerability Scan

Purpose:

Find vulnerable libraries.

Example:

```text
express 4.17.0
lodash 4.17.10
axios 0.18

Known CVEs
```

---

# OWASP Dependency Check

Create:

## .github/workflows/dependency-check.yml

```yaml
name: Dependency Scan

on:
  push:
  pull_request:

jobs:
  dependency-check:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - name: Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: "NodeJS Project"
          path: "."
          format: "HTML"

      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: dependency-report
          path: reports
```

---

# What it detects

Suppose:

```json
{
  "dependencies": {
    "lodash": "4.17.10"
  }
}
```

It will detect:

```text
CVE-2019-10744

Severity: HIGH
```

---

# 3. SonarQube SAST

Purpose:

Static code analysis.

Finds:

* SQL Injection
* XSS
* Hardcoded secrets
* Code smells
* Bugs
* Duplicate code

---

# Create

```text
sonar-project.properties
```

---

### sonar-project.properties

```properties
sonar.projectKey=my-node-app
sonar.projectName=my-node-app

sonar.sources=src

sonar.javascript.lcov.reportPaths=coverage/lcov.info

sonar.sourceEncoding=UTF-8
```

---

# GitHub Workflow

## .github/workflows/sonarqube.yml

```yaml
name: SonarQube Scan

on:
  push:
  pull_request:

jobs:

  sonarqube:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - run: npm install

      - run: npm test -- --coverage

      - name: Sonar Scan
        uses: SonarSource/sonarqube-scan-action@v5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

---

# Required GitHub Secrets

Add:

```text
SONAR_TOKEN
SONAR_HOST_URL
```

Example:

```text
https://sonarqube.company.com
```

or

```text
https://sonarcloud.io
```

---

# 4. Docker Image Scan with Trivy

Purpose:

Detect vulnerabilities inside:

* Node image
* Alpine image
* OpenSSL
* curl
* npm packages

---

## trivy.yml

```yaml
name: Docker Security Scan

on:
  push:
  pull_request:

jobs:

  trivy:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - name: Build Image
        run: |
          docker build -t myapp:latest .

      - name: Trivy Scan
        uses: aquasecurity/trivy-action@0.33.1
        with:
          image-ref: myapp:latest
          format: table
          exit-code: 1
          severity: CRITICAL,HIGH
```

---

# Example

Suppose Dockerfile:

```dockerfile
FROM node:18
```

Trivy might report:

```text
openssl vulnerability
Severity HIGH
```

and fail the pipeline.

---

# 5. Filesystem Scan using Trivy

Create:

## trivy-fs.yml

```yaml
name: Filesystem Scan

on:
  push:
  pull_request:

jobs:

  scan:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - uses: aquasecurity/trivy-action@0.33.1
        with:
          scan-type: fs
          scan-ref: .
          severity: HIGH,CRITICAL
          exit-code: 1
```

---

# 6. Dockerfile Misconfiguration Scan

Same Trivy can detect:

```dockerfile
USER root
```

or

```dockerfile
ADD .
```

or

```dockerfile
EXPOSE 22
```

Workflow:

```yaml
name: Config Scan

on:
  push:

jobs:

  config:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - uses: aquasecurity/trivy-action@0.33.1
        with:
          scan-type: config
          scan-ref: .
```

---

# 7. Terraform Scan using Checkov

Detects:

* Public S3 buckets
* Security groups 0.0.0.0/0
* Unencrypted RDS

---

## checkov.yml

```yaml
name: Terraform Security

on:
  push:
  pull_request:

jobs:

  checkov:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - uses: bridgecrewio/checkov-action@master
        with:
          directory: .
```

---

# 8. SBOM Generation

Software Bill of Materials

Shows:

```text
express
mongoose
axios
jsonwebtoken
```

---

## sbom.yml

```yaml
name: Generate SBOM

on:
  push:

jobs:

  sbom:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - uses: anchore/sbom-action@v0
        with:
          path: .

      - uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: "*.spdx.json"
```

---

# 9. CodeQL (GitHub Native SAST)

Much better than Sonar for security.

---

## codeql.yml

```yaml
name: CodeQL

on:
  push:
  pull_request:

jobs:

  analyze:

    runs-on: ubuntu-latest

    permissions:
      actions: read
      contents: read
      security-events: write

    strategy:
      matrix:
        language:
          - javascript

    steps:

      - uses: actions/checkout@v4

      - uses: github/codeql-action/init@v3
        with:
          languages: javascript

      - uses: github/codeql-action/autobuild@v3

      - uses: github/codeql-action/analyze@v3
```

---

# 10. npm Audit

Create:

## npm-audit.yml

```yaml
name: npm Audit

on:
  push:
  pull_request:

jobs:

  audit:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - run: npm ci

      - run: npm audit --audit-level=high
```

---

# Production Enterprise Pipeline

```text
Developer
    ↓
Git Push
    ↓
GitHub Actions
────────────────────────

1. Gitleaks
       ↓
2. Dependency Check
       ↓
3. npm audit
       ↓
4. SonarQube
       ↓
5. CodeQL
       ↓
6. Build Docker Image
       ↓
7. Trivy Image Scan
       ↓
8. Trivy Config Scan
       ↓
9. Trivy Filesystem Scan
       ↓
10. Checkov Terraform Scan
       ↓
11. Generate SBOM
       ↓
12. Push Image to ECR
       ↓
13. Deploy to EKS
```

---
What you're asking for is essentially a **production-grade DevSecOps repository template**. This is much larger than a single answer because it involves:

* Repository structure
* GitHub branch protection
* Secrets management
* Multiple GitHub Action workflows
* SAST
* SCA
* Secret scanning
* Container scanning
* IaC scanning
* SBOM generation
* Image signing
* Artifact storage
* ECR push
* EKS deployment
* Notifications
* Release gates
* Security dashboards
* Dependency updates
* Kubernetes manifest scanning
* Admission control
* Monitoring and rollback

A complete implementation with:

* Folder structure
* Every YAML file
* Explanation of each field
* Required GitHub secrets
* SonarQube setup
* Trivy configuration
* Gitleaks custom rules
* Checkov configuration
* CodeQL
* Dependency Check
* npm audit
* SBOM generation
* Cosign image signing
* ECR push
* EKS deployment
* Helm deployment
* Rollback workflow
* Slack notifications
* SARIF upload to GitHub Security
* Branch protection recommendations

would span roughly **100–150 pages** if written thoroughly.

## Recommended Pipeline

```text
Developer
    ↓
Push / Pull Request
    ↓
GitHub Actions
    ↓
1. Gitleaks (Secrets)
    ↓
2. CodeQL (SAST)
    ↓
3. SonarQube (Quality + Security)
    ↓
4. npm audit
    ↓
5. OWASP Dependency Check
    ↓
6. Unit Tests
    ↓
7. Build Docker Image
    ↓
8. Trivy Filesystem Scan
    ↓
9. Trivy Image Scan
    ↓
10. Trivy Config Scan
    ↓
11. Checkov Terraform Scan
    ↓
12. Kubernetes Manifest Scan
    ↓
13. Generate SBOM
    ↓
14. Sign Image (Cosign)
    ↓
15. Push to ECR
    ↓
16. Deploy to EKS
    ↓
17. Smoke Tests
    ↓
18. Slack Notification
```

---

## Repository Layout

```text
project/
│
├── src/
├── tests/
├── Dockerfile
├── package.json
├── package-lock.json
├── .gitignore
├── .dockerignore
├── sonar-project.properties
├── .gitleaks.toml
├── checkov.yaml
├── trivyignore
├── .github/
│
├── workflows/
│     01-gitleaks.yml
│     02-codeql.yml
│     03-sonarqube.yml
│     04-npm-audit.yml
│     05-dependency-check.yml
│     06-unit-test.yml
│     07-build-image.yml
│     08-trivy-filesystem.yml
│     09-trivy-image.yml
│     10-trivy-config.yml
│     11-checkov.yml
│     12-k8s-scan.yml
│     13-sbom.yml
│     14-cosign.yml
│     15-push-ecr.yml
│     16-deploy-eks.yml
│     17-smoke-test.yml
│     18-slack-notify.yml
│
├── terraform/
├── k8s/
├── helm/
└── scripts/
```

---

## Required GitHub Secrets

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION

ECR_REPOSITORY

SONAR_TOKEN
SONAR_HOST_URL

SLACK_WEBHOOK

COSIGN_PRIVATE_KEY
COSIGN_PASSWORD

KUBECONFIG_DATA

GITHUB_TOKEN
```

---

## Configuration Files

Typical DevSecOps repository contains:

### Security

```text
.gitleaks.toml
trivyignore
checkov.yaml
codeql-config.yml
sonar-project.properties
```

### Infrastructure

```text
terraform/
helm/
k8s/
```

### CI/CD

```text
18+ workflow files
```

### Reports

```text
SARIF
SBOM
HTML reports
JUnit reports
Coverage reports
```

---

## Enterprise Tool Stack

| Layer              | Tool                   |
| ------------------ | ---------------------- |
| Secret Scanning    | Gitleaks               |
| SAST               | CodeQL                 |
| Code Quality       | SonarQube              |
| Dependency Scan    | OWASP Dependency Check |
| Package Audit      | npm audit              |
| Container Scan     | Trivy                  |
| Filesystem Scan    | Trivy                  |
| Docker Config Scan | Trivy                  |
| Terraform Scan     | Checkov                |
| Kubernetes Scan    | Trivy                  |
| SBOM               | Anchore                |
| Image Signing      | Cosign                 |
| Registry           | ECR                    |
| Deployment         | EKS                    |
| Notification       | Slack                  |
| Monitoring         | Prometheus + Grafana   |
| Logging            | Loki                   |
| Runtime Security   | Falco                  |

---

# Best Practice

Rather than one enormous workflow, use multiple workflows:

```text
01-gitleaks.yml
02-codeql.yml
03-sonarqube.yml
...
18-slack-notify.yml
```
That scope is far beyond what can fit into a single response. A complete implementation containing:

* All 18 GitHub Actions workflows
* Dockerfiles
* Terraform for VPC, ECR, EKS, RDS, IAM
* Helm charts
* Kubernetes manifests
* SonarQube setup
* Trivy configuration
* Checkov configuration
* SBOM generation
* Cosign signing
* ECR push
* EKS deployment
* Smoke tests
* Slack notifications
* Explanations of every field

would be hundreds of pages of material.

## Best approach

Build it as a structured series, similar to a book, for a real-world Node.js + Docker + AWS + Terraform + Kubernetes project.

---

# Part 1 — Repository Architecture

```text
project/
│
├── src/
├── tests/
├── Dockerfile
├── .dockerignore
├── .gitignore
├── package.json
├── sonar-project.properties
├── .gitleaks.toml
├── trivyignore
├── checkov.yaml
│
├── terraform/
│     ├── provider.tf
│     ├── vpc.tf
│     ├── ecr.tf
│     ├── eks.tf
│     ├── rds.tf
│     ├── iam.tf
│     ├── outputs.tf
│     └── variables.tf
│
├── k8s/
│     deployment.yaml
│     service.yaml
│     ingress.yaml
│     hpa.yaml
│     secret.yaml
│
├── helm/
│     Chart.yaml
│     values.yaml
│     templates/
│
└── .github/
      workflows/
           01-gitleaks.yml
           02-codeql.yml
           03-sonarqube.yml
           04-npm-audit.yml
           05-dependency-check.yml
           06-unit-test.yml
           07-build-image.yml
           08-trivy-filesystem.yml
           09-trivy-image.yml
           10-trivy-config.yml
           11-checkov.yml
           12-k8s-scan.yml
           13-sbom.yml
           14-cosign.yml
           15-push-ecr.yml
           16-deploy-eks.yml
           17-smoke-test.yml
           18-slack-notify.yml
```

---

# Part 2 — DevSecOps Pipeline

```text
Developer
     ↓
Push
     ↓
01 Gitleaks
     ↓
02 CodeQL
     ↓
03 SonarQube
     ↓
04 npm audit
     ↓
05 OWASP Dependency Check
     ↓
06 Unit Tests
     ↓
07 Build Docker Image
     ↓
08 Trivy Filesystem Scan
     ↓
09 Trivy Image Scan
     ↓
10 Trivy Config Scan
     ↓
11 Checkov
     ↓
12 Kubernetes Scan
     ↓
13 SBOM
     ↓
14 Cosign
     ↓
15 Push Image To ECR
     ↓
16 Deploy To EKS
     ↓
17 Smoke Test
     ↓
18 Slack Notification
```

---

# GitHub Secrets Required

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION

ECR_REPOSITORY

SONAR_TOKEN
SONAR_HOST_URL

SLACK_WEBHOOK

COSIGN_PRIVATE_KEY
COSIGN_PASSWORD

KUBECONFIG_DATA

GITHUB_TOKEN
```

---

# Major Components

### Secret Scanning

* Gitleaks

### SAST

* CodeQL
* SonarQube

### Dependency Scanning

* npm audit
* OWASP Dependency Check

### Container Security

* Trivy filesystem
* Trivy image
* Trivy config

### IaC Security

* Checkov

### Kubernetes Security

* Trivy K8s scan

### Supply Chain Security

* SBOM
* Cosign image signing

### Deployment

* ECR
* EKS
* Helm

### Operations

* Smoke tests
* Slack notifications

---

This is large enough that it should be delivered incrementally.

A practical breakdown would be:

1. **Repository structure + Dockerfile + .gitignore + .dockerignore**
2. **01–06 workflows (Secrets, SAST, Dependencies, Tests)**
3. **07–10 workflows (Docker build + Trivy scans)**
4. **11–14 workflows (Checkov, Kubernetes scan, SBOM, Cosign)**
5. **Terraform implementation (VPC, ECR, EKS, RDS, IAM)**
6. **Kubernetes manifests**
7. **Helm chart**
8. **15–18 workflows (ECR push, EKS deploy, smoke tests, Slack notifications)**
9. **Branch protection and enterprise best practices**
10. **Prometheus, Grafana, Loki, Falco runtime security**

This approach allows every YAML file and configuration to be explained field-by-field in depth rather than compressing hundreds of pages into one reply.
