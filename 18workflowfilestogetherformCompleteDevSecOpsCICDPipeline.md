These 18 workflow files together form a **Complete DevSecOps CI/CD Pipeline**. Each file has a specific responsibility. Think of them as security checkpoints that code must pass before reaching production.

---

# Complete Flow

```text
Developer
    ↓
Push Code
    ↓
01 Gitleaks
    ↓
02 CodeQL
    ↓
03 SonarQube
    ↓
04 npm audit
    ↓
05 Dependency Check
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
15 Push to ECR
    ↓
16 Deploy to EKS
    ↓
17 Smoke Test
    ↓
18 Slack Notification
```

---

# 01-gitleaks.yml

## Purpose

Detect secrets accidentally committed to Git.

### Finds

* AWS access keys
* Database passwords
* JWT secrets
* API tokens
* Private keys

Example:

```env
DB_PASSWORD=admin123
AWS_SECRET_ACCESS_KEY=abcxyz
```

### Why important?

Secrets inside Git repositories are one of the most common causes of cloud compromises.

### Runs

Immediately after push or pull request.

---

# 02-codeql.yml

## Purpose

Static Application Security Testing (SAST).

GitHub's CodeQL analyzes source code.

### Detects

* SQL Injection

```javascript
query = "SELECT * FROM users WHERE id=" + req.body.id
```

* Cross-Site Scripting (XSS)

```javascript
res.send(req.body.comment)
```

* Command Injection

```javascript
exec(req.body.command)
```

* Path Traversal
* Insecure Deserialization

### Why important?

Detects vulnerabilities before the application runs.

---

# 03-sonarqube.yml

## Purpose

Code quality and security analysis.

### Detects

#### Bugs

```javascript
if(a=b)
```

instead of

```javascript
if(a==b)
```

---

#### Code Smells

Large methods

Duplicate code

Unused variables

---

#### Security Issues

Hardcoded passwords

Weak cryptography

---

#### Coverage

Measures test coverage.

Example:

```
80% coverage
```

### Why important?

Improves maintainability.

---

# 04-npm-audit.yml

## Purpose

Checks Node.js dependencies.

Example:

```json
"lodash": "4.17.10"
```

### Detects

Known vulnerabilities from npm advisory database.

Example:

```
CVE-2019-10744
HIGH severity
```

### Why important?

Third-party packages are frequent attack vectors.

---

# 05-dependency-check.yml

## Purpose

OWASP Dependency Check.

### Detects

Vulnerable libraries.

Example:

```json
express
axios
lodash
moment
```

Compares them against:

* NVD database
* CVE database

### Generates

HTML reports.

### Why important?

Protects against vulnerable dependencies.

---

# 06-unit-test.yml

## Purpose

Runs automated tests.

Example:

```javascript
describe("login", ()=>{
})
```

### Checks

* Business logic
* APIs
* Functions

### Why important?

Prevents broken code from reaching production.

---

# 07-build-image.yml

## Purpose

Build Docker image.

Example:

```dockerfile
FROM node:22
```

Builds:

```text
myapp:latest
```

### Why important?

Creates deployable artifact.

---

# 08-trivy-filesystem.yml

## Purpose

Scan repository files.

### Detects

* Secrets
* Vulnerabilities
* Misconfigurations

Scans:

```text
package.json
requirements.txt
pom.xml
```

### Why important?

Find issues before Docker image creation.

---

# 09-trivy-image.yml

## Purpose

Scan Docker image.

Example:

```text
myapp:latest
```

### Detects

OpenSSL vulnerabilities

Node vulnerabilities

Alpine vulnerabilities

### Example

```
HIGH
openssl CVE-2025-XXXX
```

### Why important?

Protects containers.

---

# 10-trivy-config.yml

## Purpose

Configuration scan.

### Scans

Dockerfiles

Terraform

Kubernetes manifests

### Detects

Container running as root:

```dockerfile
USER root
```

Privileged mode:

```yaml
privileged: true
```

Public ports

Weak permissions

### Why important?

Prevents insecure configurations.

---

# 11-checkov.yml

## Purpose

Infrastructure as Code (IaC) security.

### Scans

Terraform

CloudFormation

Kubernetes YAML

### Detects

Public S3 bucket

```hcl
acl = "public-read"
```

Security group open to world

```hcl
0.0.0.0/0
```

Unencrypted RDS

### Why important?

Prevents cloud misconfigurations.

---

# 12-k8s-scan.yml

## Purpose

Kubernetes manifest scanning.

### Scans

```yaml
deployment.yaml
service.yaml
ingress.yaml
```

### Detects

Root containers

Missing resource limits

Privileged containers

Latest tags

```yaml
image: nginx:latest
```

### Why important?

Secures Kubernetes workloads.

---

# 13-sbom.yml

## Purpose

Generate Software Bill of Materials.

### Lists

```text
express
mongoose
axios
jsonwebtoken
```

### Why important?

Supply chain visibility.

Useful for compliance and auditing.

---

# 14-cosign.yml

## Purpose

Digitally sign container images.

### Ensures

Image integrity.

Verifies:

```text
This image came from our pipeline.
```

### Prevents

Tampered images.

### Why important?

Supply chain security.

---

# 15-push-ecr.yml

## Purpose

Push Docker image to AWS ECR.

Example:

```text
123456789.dkr.ecr.us-east-1.amazonaws.com/app:v1
```

### Why important?

Stores deployable images.

Acts as container registry.

---

# 16-deploy-eks.yml

## Purpose

Deploy application to Kubernetes.

Updates:

```yaml
Deployment
Service
Ingress
ConfigMap
Secret
```

### Why important?

Automates production deployment.

---

# 17-smoke-test.yml

## Purpose

Verify deployment success.

Example:

Checks:

```bash
curl https://app.example.com/health
```

Expected:

```json
{
 "status":"UP"
}
```

### Why important?

Ensures application is actually working.

---

# 18-slack-notify.yml

## Purpose

Send pipeline notifications.

### Success

```text
Deployment Successful
Version v1.2.0
```

### Failure

```text
Trivy scan failed
HIGH vulnerabilities detected
```

### Why important?

Provides visibility to developers and DevOps teams.

---

# Categories

## Security

| Workflow | Tool                   |
| -------- | ---------------------- |
| 01       | Gitleaks               |
| 02       | CodeQL                 |
| 03       | SonarQube              |
| 04       | npm audit              |
| 05       | OWASP Dependency Check |
| 08       | Trivy FS               |
| 09       | Trivy Image            |
| 10       | Trivy Config           |
| 11       | Checkov                |
| 12       | K8s Scan               |

---

## Quality

| Workflow | Tool       |
| -------- | ---------- |
| 06       | Unit Tests |
| 03       | SonarQube  |

---

## Supply Chain Security

| Workflow | Tool   |
| -------- | ------ |
| 13       | SBOM   |
| 14       | Cosign |

---

## CI/CD

| Workflow | Tool         |
| -------- | ------------ |
| 07       | Build Image  |
| 15       | Push ECR     |
| 16       | Deploy EKS   |
| 17       | Smoke Test   |
| 18       | Slack Notify |

---

# Overall Goal

These workflows collectively protect:

```text
Source Code
     ↓
Dependencies
     ↓
Docker Images
     ↓
Terraform
     ↓
Kubernetes
     ↓
Supply Chain
     ↓
AWS Infrastructure
     ↓
Production Environment
```

This architecture is very close to what is used in enterprise DevSecOps environments for AWS + Docker + Kubernetes platforms.
