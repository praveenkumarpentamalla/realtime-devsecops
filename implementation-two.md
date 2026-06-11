# Complete DevSecOps Implementation Guide (Step-by-Step)

The items you listed form a **multi-layer secret protection strategy**:

1. `.gitignore` → Prevent sensitive files from being tracked.
2. Native Git Hook → Block suspicious commits locally.
3. Pre-commit framework + Gitleaks → Scan before every commit.
4. Custom Gitleaks rules → Detect passwords and secrets.
5. GitHub Actions → Scan entire repository and history in CI/CD.
6. `.dockerignore` → Prevent secrets from entering Docker images.

---

# Architecture

```
Developer Machine
       |
       |
git add
       |
       ▼
Native pre-commit hook
       |
       ▼
Pre-commit Framework
       |
       ▼
Gitleaks Scan
       |
       ▼
git commit
       |
       ▼
Push to GitHub
       |
       ▼
GitHub Actions
       |
       ▼
Repository History Scan
```

---

# Project Structure

Suppose your project is:

```
my-project/
│
├── .git/
├── .github/
│     └── workflows/
│            gitleaks.yml
│
├── .gitignore
├── .dockerignore
├── .pre-commit-config.yaml
├── custom-rules.toml
├── .env
├── package.json
├── Dockerfile
├── src/
├── dist/
└── node_modules/
```

---

# Step 1: Configure `.gitignore`

### Purpose

Prevent sensitive files from being committed.

Create:

```
my-project/.gitignore
```

Put:

```gitignore
.env
.env.*
*.pem
*.key
id_rsa
terraform.tfstate
.terraform/
node_modules/
dist/
```

---

## Explanation

### Ignore environment files

```gitignore
.env
.env.*
```

Examples:

```
.env
.env.production
.env.local
.env.dev
```

These files usually contain:

```env
DB_PASSWORD=123456
AWS_SECRET_ACCESS_KEY=abcd
JWT_SECRET=xyz
```

Never commit them.

---

### Ignore private keys

```gitignore
*.pem
*.key
id_rsa
```

Examples:

```
CivilBook_Key_Pair.pem
private.key
id_rsa
```

These are SSH keys and AWS keys.

---

### Ignore Terraform state

```gitignore
terraform.tfstate
.terraform/
```

Terraform state may contain:

* RDS passwords
* IAM keys
* Resource metadata

Example:

```json
{
 "db_password":"admin123"
}
```

---

### Ignore dependencies

```gitignore
node_modules/
```

Avoid storing 1000s of packages in Git.

---

### Ignore build output

```gitignore
dist/
```

Generated files should not be version controlled.

---

# Step 2: Native Git Hook

Location:

```
.git/hooks/pre-commit
```

Inside:

```bash
#!/bin/bash

echo "🔍 Running native pre-commit hook..."

if git diff --cached | grep -i "secret"; then
  echo "❌ Secret detected. Commit blocked."
  exit 1
fi

echo "✅ Commit passed security checks."
exit 0
```

---

## What does it do?

When you run:

```bash
git commit -m "update"
```

Git automatically executes:

```
.git/hooks/pre-commit
```

before creating the commit.

---

## Meaning

This command:

```bash
git diff --cached
```

shows staged files.

Example:

```bash
API_SECRET=mysecret123
```

Then:

```bash
grep -i "secret"
```

finds:

```
SECRET
secret
Secret
```

and blocks commit.

---

## Make it executable

Linux:

```bash
chmod +x .git/hooks/pre-commit
```

Verify:

```bash
ls -l .git/hooks/pre-commit
```

Should show:

```bash
-rwxr-xr-x
```

---

# Step 3: Install Pre-Commit Framework

Pre-commit is installed on the developer machine.

Not inside GitHub.

Not inside Docker.

Not inside the project.

---

## Ubuntu

Install pip:

```bash
sudo apt update
sudo apt install python3-pip -y
```

Install pre-commit:

```bash
pip3 install pre-commit
```

Verify:

```bash
pre-commit --version
```

Example:

```bash
pre-commit 4.2.0
```

---

## Mac

```bash
brew install pre-commit
```

---

## Windows

```powershell
pip install pre-commit
```

---

# Step 4: Create `.pre-commit-config.yaml`

Location:

```
my-project/.pre-commit-config.yaml
```

Content:

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.24.2
    hooks:
      - id: gitleaks
```

---

## Install hooks

Run:

```bash
pre-commit install
```

Output:

```text
pre-commit installed at .git/hooks/pre-commit
```

---

## Test

Commit:

```bash
git add .
git commit -m "test"
```

Automatically:

```
pre-commit
     ↓
gitleaks
     ↓
allow/block
```

---

# Step 5: Create Custom Rules

### Location

Create in project root:

```
my-project/custom-rules.toml
```

Example:

```
my-project/
│
├── custom-rules.toml
├── .pre-commit-config.yaml
├── package.json
└── src/
```

---

### Content

```toml
[[rules]]
id = "generic-password"
description = "Detect any PASSWORD assignment"
regex = '''(?i)password\s*=\s*["'][^"']+["']'''
tags = ["password", "custom"]
```

---

## Meaning

It catches:

```javascript
password="admin123"
```

or

```python
PASSWORD='root'
```

or

```env
password="123456"
```

---

# Step 6: Run Gitleaks Manually

Install Gitleaks.

### Ubuntu

```bash
brew install gitleaks
```

or

```bash
wget https://github.com/gitleaks/gitleaks/releases/download/v8.24.2/gitleaks_8.24.2_linux_x64.tar.gz
```

Extract:

```bash
tar -xzf gitleaks_8.24.2_linux_x64.tar.gz
```

Move:

```bash
sudo mv gitleaks /usr/local/bin/
```

Verify:

```bash
gitleaks version
```

---

## Scan current repository

```bash
gitleaks detect
```

---

## Scan complete Git history

```bash
gitleaks detect --source . --verbose
```

---

## Using custom rules

```bash
gitleaks detect \
--source . \
--config custom-rules.toml
```

---

# Step 7: GitHub Actions

Folder:

```
.github/workflows/
```

Create:

```
.github/workflows/gitleaks.yml
```

Structure:

```
my-project
│
├── .github
│     └── workflows
│             gitleaks.yml
```

---

# Content

```yaml
name: gitleaks

on:
  push:
  pull_request:
  workflow_dispatch:

jobs:
  scan:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

# Why fetch-depth=0?

Normally:

```yaml
fetch-depth: 1
```

downloads only latest commit.

But:

```yaml
fetch-depth: 0
```

downloads entire history.

Gitleaks can detect secrets hidden in older commits.

---

# Step 8: Add Custom Rules to GitHub Actions

Update:

```yaml
- name: Run Gitleaks
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  with:
    config-path: custom-rules.toml
```

Now GitHub Actions uses your custom rules.

---

# Step 9: `.dockerignore`

Location:

```
my-project/.dockerignore
```

Content:

```dockerignore
.git
.gitignore
node_modules
.env
Dockerfile*
README.md
```

---

## Why?

Suppose Dockerfile has:

```dockerfile
COPY . .
```

Without `.dockerignore`, Docker copies:

```
.git/
.env
node_modules/
README.md
```

into image.

This may expose:

* passwords
* API keys
* Git history

---

# Example

Bad:

```
Image
│
├── .env
├── AWS_SECRET_ACCESS_KEY
├── .git
└── node_modules
```

Good:

```
Image
│
├── src/
├── package.json
└── dist/
```

---

# Complete DevSecOps Workflow

### Developer Side

```text
Code
 ↓
git add
 ↓
Native Hook
 ↓
Pre-commit Framework
 ↓
Gitleaks
 ↓
Commit
 ↓
Push
```

---

### CI/CD Side

```text
Push
 ↓
GitHub Actions
 ↓
Checkout entire history
 ↓
Run Gitleaks
 ↓
Secret Detection
 ↓
Pass/Fail Pipeline
```

---

# Production-Level Recommendation

A mature DevSecOps pipeline usually includes:

```
Developer
    ↓
.gitignore
    ↓
Pre-commit
    ↓
Gitleaks
    ↓
GitHub Actions
    ↓
Trivy (Docker Scan)
    ↓
SonarQube (SAST)
    ↓
Dependency Check
    ↓
Terraform Checkov
    ↓
Docker Image Signing
    ↓
Kubernetes Admission Controller
```

This creates a complete **Shift-Left DevSecOps pipeline**, where secrets and vulnerabilities are detected before they reach production.
