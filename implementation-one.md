# Complete DevSecOps Implementation Guide: Secrets Detection & Prevention

## Overview

This guide covers implementing a comprehensive secrets detection and prevention strategy using native Git hooks, Gitleaks, and GitHub Actions. This prevents sensitive data like passwords, API keys, and tokens from being committed to your repository.

## 📁 Project Structure

```
your-project/
├── .gitignore
├── .git/
│   └── hooks/
│       └── pre-commit (custom script)
├── .pre-commit-config.yaml
├── custom-rules.toml
├── .github/
│   └── workflows/
│       └── gitleaks.yml
├── .dockerignore
└── (your application files)
```

---

## 1️⃣ .gitignore Configuration

### **Purpose**: Prevents sensitive files from ever being staged for commit

### **Location**: Root of your repository (`/path/to/your-project/.gitignore`)

### **Complete Configuration**:

```gitignore
# Environment Variables - NEVER commit these!
.env
.env.*
.env.local
.env.production
.env.staging
.env.development

# Security Keys & Certificates
*.pem
*.key
*.crt
*.p12
*.pfx
*.der
*.cer
*.p7b
*.p7r

# SSH Keys
id_rsa
id_rsa.pub
id_dsa
id_ecdsa
id_ed25519
*.ppk

# Terraform State Files (contain secrets!)
terraform.tfstate
terraform.tfstate.*
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl

# Node.js (often contains .env files)
node_modules/
package-lock.json
yarn.lock
npm-debug.log
yarn-error.log

# Build Artifacts
dist/
build/
out/
*.log

# IDE Configurations (may contain secrets in settings)
.vscode/
.idea/
*.swp
*.swo
*~
.DS_Store

# Testing & Coverage
coverage/
.nyc_output/
.nyc_output/

# Docker sensitive files
docker-compose.override.yml
docker-compose.*.yml

# Cloud provider configs
.aws/
.gcloud/
.azure/
.terraform.d/

# Token files
*.token
*.secrets
secrets.json
secrets.yaml
credentials.json
service-account.json
```

### **Explanation**:
- **Environment files** contain database passwords, API keys
- **Key files** are private certificates and SSH keys
- **State files** store infrastructure secrets in plaintext
- Git ignores these patterns, preventing accidental staging

---

## 2️⃣ Native Git Pre-Commit Hook (Custom Script)

### **Purpose**: Quick, lightweight check before each commit

### **Location**: `/path/to/your-project/.git/hooks/pre-commit`

### **Complete Script**:

```bash
#!/bin/bash

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${YELLOW}🔍 Running native pre-commit hook...${NC}"

# Common secret patterns to detect
SECRET_PATTERNS=(
    "secret"
    "password"
    "api_key"
    "apikey"
    "token"
    "auth_token"
    "private_key"
    "-----BEGIN RSA PRIVATE KEY-----"
    "-----BEGIN OPENSSH PRIVATE KEY-----"
    "AWS_SECRET_ACCESS_KEY"
    "AWS_ACCESS_KEY_ID"
    "DATABASE_URL"
    "REDIS_URL"
    "JWT_SECRET"
    "ENCRYPTION_KEY"
)

# Track if any secret is found
FOUND_SECRETS=0

# Get list of staged files
STAGED_FILES=$(git diff --cached --name-only)

# Check each staged file
for FILE in $STAGED_FILES; do
    # Skip binary files and .gitignored files
    if [[ -f "$FILE" ]]; then
        # Check file extension to avoid binary files
        if file "$FILE" | grep -q "text"; then
            for PATTERN in "${SECRET_PATTERNS[@]}"; do
                if git diff --cached "$FILE" | grep -i "$PATTERN" > /dev/null 2>&1; then
                    echo -e "${RED}❌ Secret pattern '$PATTERN' detected in file: $FILE${NC}"
                    git diff --cached "$FILE" | grep -i "$PATTERN" --color=always
                    FOUND_SECRETS=1
                fi
            done
        fi
    fi
done

# Also check new files (not just changes)
for FILE in $STAGED_FILES; do
    if [[ -f "$FILE" ]] && file "$FILE" | grep -q "text"; then
        for PATTERN in "${SECRET_PATTERNS[@]}"; do
            if grep -i "$PATTERN" "$FILE" > /dev/null 2>&1; then
                # Check if file is newly added
                if git diff --cached --name-only --diff-filter=A | grep -q "^$FILE$"; then
                    echo -e "${RED}❌ Secret pattern '$PATTERN' found in new file: $FILE${NC}"
                    grep -i "$PATTERN" "$FILE" --color=always
                    FOUND_SECRETS=1
                fi
            fi
        done
    fi
done

if [ $FOUND_SECRETS -eq 1 ]; then
    echo -e "${RED}❌ Commit blocked: Potential secrets detected!${NC}"
    echo -e "${YELLOW}💡 Hint: Remove secrets or use environment variables${NC}"
    echo -e "${YELLOW}💡 To bypass (not recommended): git commit --no-verify${NC}"
    exit 1
fi

echo -e "${GREEN}✅ Commit passed security checks.${NC}"
exit 0
```

### **Installation Instructions**:

```bash
# Make the hook executable
chmod +x .git/hooks/pre-commit

# Test the hook
echo "password=secret123" > test.txt
git add test.txt
# This should BLOCK the commit

# Clean up test
rm test.txt
```

### **How It Works**:
1. Runs automatically on `git commit`
2. Scans staged changes for secret patterns
3. Blocks commit if secrets found
4. Shows exactly where the secret was detected

---

## 3️⃣ Gitleaks Pre-Commit Hook (Professional Tool)

### **What is Gitleaks**?
Professional open-source tool for detecting secrets in Git repositories. Much more sophisticated than custom scripts.

### **Installation**:

**Option 1: Install pre-commit framework (Recommended)**

```bash
# On macOS
brew install pre-commit

# On Linux (Ubuntu/Debian)
pip install pre-commit
# OR
sudo apt install pre-commit

# On Windows (using Chocolatey)
choco install pre-commit
# OR using pip
pip install pre-commit

# Verify installation
pre-commit --version
```

**Option 2: Install Gitleaks directly**

```bash
# macOS
brew install gitleaks

# Linux
curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.24.2/gitleaks_8.24.2_linux_x64.tar.gz | tar xz
sudo mv gitleaks /usr/local/bin/

# Windows (using scoop)
scoop install gitleaks
```

### **Configuration File**: `.pre-commit-config.yaml`

### **Location**: `/path/to/your-project/.pre-commit-config.yaml`

### **Complete Configuration**:

```yaml
# .pre-commit-config.yaml
# Documentation: https://pre-commit.com/

# Minimum pre-commit version
minimum_pre_commit_version: '2.9.0'

# Default language version
default_language_version:
  python: python3
  node: system

# Repository definitions
repos:
  # Gitleaks for secrets detection
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.24.2  # Check for latest version: https://github.com/gitleaks/gitleaks/releases
    hooks:
      - id: gitleaks
        name: Gitleaks Secrets Scanner
        description: Detect hardcoded secrets like passwords, API keys, and tokens
        entry: gitleaks protect --staged --verbose
        language: system
        pass_filenames: false
        always_run: true
        stages: [commit]
        verbose: true

  # Additional security hooks (optional but recommended)
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-json
        name: Validate JSON files
      - id: check-yaml
        name: Validate YAML files
      - id: end-of-file-fixer
        name: Fix end of files
      - id: trailing-whitespace
        name: Trim trailing whitespace
      - id: detect-private-key
        name: Detect private keys
      - id: detect-aws-credentials
        name: Detect AWS credentials
      - id: check-added-large-files
        name: Check for large files
        args: ['--maxkb=500']

  # TruffleHog - Another secrets scanner (optional)
  - repo: https://github.com/trufflesecurity/trufflehog
    rev: v3.63.7
    hooks:
      - id: trufflehog
        name: TruffleHog Secrets Scanner
        description: Detect secrets with entropy checking
        entry: trufflehog git file://. --no-verification --only-verified
        language: system
        pass_filenames: false
        always_run: true
        stages: [commit]
```

### **Setup and Usage**:

```bash
# Navigate to your repository
cd /path/to/your-project

# Install the git hook scripts
pre-commit install

# Install the commit-msg hook (optional)
pre-commit install --hook-type commit-msg

# Run against all files manually
pre-commit run --all-files

# Run specific hook
pre-commit run gitleaks

# Update hooks to latest versions
pre-commit autoupdate

# Bypass hooks (not recommended)
git commit --no-verify -m "Emergency fix"
```

### **Testing Gitleaks**:

```bash
# Create a test file with a secret
echo "AWS_SECRET_ACCESS_KEY = 'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY'" > test_secret.txt

# Add to staging
git add test_secret.txt

# Try to commit
git commit -m "Test commit with secret"

# Gitleaks should block the commit and show:
# ❌ Detected secret: AWS Secret Access Key
# ✓ 1 commits scanned
# ○ 1 secret found
```

---

## 4️⃣ Custom Gitleaks Rules

### **Purpose**: Add organization-specific secret patterns

### **Location**: `/path/to/your-project/custom-rules.toml`

### **Complete Custom Rules Configuration**:

```toml
# custom-rules.toml
# Gitleaks custom rules file
# Documentation: https://github.com/gitleaks/gitleaks/tree/master/config

# Title for the ruleset
title = "Custom Security Rules for My Organization"

# Global configuration
[extend]
# Use default Gitleaks rules as base
useDefault = true

# Custom rule 1: Generic password detection
[[rules]]
id = "generic-password-detection"
description = "Detects any password assignment in code"
regex = '''(?i)password\s*[:=]\s*["'][^"']{8,}["']'''
tags = ["password", "credentials", "custom"]
severity = "HIGH"
keywords = ["password", "passwd", "pwd"]

# Custom rule 2: API key patterns
[[rules]]
id = "custom-api-key"
description = "Detects custom API key patterns"
regex = '''(?i)(api[_-]?key|apikey|api_token|token)\s*[:=]\s*["'][a-zA-Z0-9]{16,64}["']'''
tags = ["api_key", "token", "custom"]
severity = "CRITICAL"
keywords = ["api_key", "apikey", "api-token"]

# Custom rule 3: Database connection strings
[[rules]]
id = "database-connection-string"
description = "Detects database connection strings with credentials"
regex = '''(postgresql|mysql|mongodb|redis)://[a-zA-Z0-9]+:[^@\s]+@'''
tags = ["database", "credentials", "connection_string"]
severity = "CRITICAL"
keywords = ["postgresql://", "mysql://", "mongodb://", "redis://"]

# Custom rule 4: JWT tokens
[[rules]]
id = "jwt-token-detection"
description = "Detects JWT tokens in code"
regex = '''eyJ[a-zA-Z0-9_-]{10,}\.eyJ[a-zA-Z0-9_-]{10,}\.[a-zA-Z0-9_-]{10,}'''
tags = ["jwt", "token", "authentication"]
severity = "HIGH"
keywords = ["eyJ"]

# Custom rule 5: Internal service secrets
[[rules]]
id = "internal-service-secret"
description = "Detects our internal service secret format"
regex = '''SVC_[A-Z]{3,10}_SECRET\s*=\s*["'][a-f0-9]{32}["']'''
tags = ["internal", "service_secret", "custom"]
severity = "CRITICAL"
keywords = ["SVC_", "_SECRET"]

# Custom rule 6: Slack webhook URLs
[[rules]]
id = "slack-webhook"
description = "Detects Slack webhook URLs"
regex = '''https://hooks\.slack\.com/services/[A-Z0-9]+/[A-Z0-9]+/[a-zA-Z0-9]+'''
tags = ["slack", "webhook", "integration"]
severity = "HIGH"
keywords = ["hooks.slack.com"]

# Custom rule 7: Generic high-entropy strings
[[rules]]
id = "high-entropy-string"
description = "Detects high-entropy strings (potential secrets)"
regex = '''[a-zA-Z0-9+/]{40,}={0,2}'''
tags = ["entropy", "potential_secret"]
severity = "MEDIUM"

# Allowlist - files to always ignore
[[allowlist]]
description = "Ignore test files and documentation"
paths = [
    '''.*_test\.go$''',
    '''.*_test\.py$''',
    '''.*\.md$''',
    '''.*\.txt$''',
    '''test/.*''',
    '''examples/.*''',
]

# Allowlist - commits to ignore
[[allowlist]]
description = "Known false positives in specific commits"
commits = [
    "abc123def456",  # Example commit hash
]

# Allowlist - regex patterns to ignore
[[allowlist]]
description = "Ignore example values in comments"
regexes = [
    '''password\s*=\s*["']example["']''',
    '''api_key\s*=\s*["']your-api-key-here["']''',
]
```

### **Using Custom Rules**:

```bash
# Test with custom rules on current directory
gitleaks detect --source . --config custom-rules.toml --verbose

# Test with custom rules on specific branch
gitleaks detect --source . --config custom-rules.toml --branch main

# Generate a report
gitleaks detect --source . --config custom-rules.toml --report-path leak-report.json

# Pre-commit hook with custom rules (modify .pre-commit-config.yaml)
# Add: args: ['--config', 'custom-rules.toml', '--verbose']
```

---

## 5️⃣ GitHub Actions Integration (CI/CD)

### **Purpose**: Scan all commits, PRs, and branches in CI pipeline

### **Location**: `/.github/workflows/gitleaks.yml`

### **Complete GitHub Actions Workflow**:

```yaml
# .github/workflows/gitleaks.yml
name: Security Secrets Scan

# Trigger on important events
on:
  # Run on push to any branch
  push:
    branches:
      - main
      - master
      - develop
      - 'release/**'
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.gitignore'
  
  # Run on pull requests
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches:
      - main
      - master
      - develop
  
  # Allow manual trigger
  workflow_dispatch:
  
  # Schedule weekly full scan (Sunday at 2 AM)
  schedule:
    - cron: '0 2 * * 0'

# Permissions (minimum required)
permissions:
  contents: read
  security-events: write  # For SARIF report upload
  pull-requests: write    # For PR comments

# Environment variables
env:
  GO_VERSION: '1.21'

jobs:
  # Job 1: Quick scan for staged changes (fast)
  quick-scan:
    name: Quick Scan (Staged Changes)
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 2  # Only last 2 commits for speed
          
      - name: Gitleaks Quick Scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          args: --redact --verbose --source .
          
  # Job 2: Full repository scan (comprehensive)
  full-scan:
    name: Full Repository Scan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code (full history)
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full git history
          
      - name: Run Gitleaks Full Scan
        id: gitleaks
        uses: gitleaks/gitleaks-action@v2
        continue-on-error: true  # Don't fail immediately
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}  # Required for orgs
        with:
          args: |
            --verbose
            --redact
            --log-level debug
            --config custom-rules.toml
            --report-path gitleaks-report.json
            --report-format json
          
      - name: Upload scan results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: gitleaks-report
          path: gitleaks-report.json
          retention-days: 30
          
      - name: Generate SARIF report (for GitHub Advanced Security)
        if: always()
        run: |
          # Convert Gitleaks report to SARIF format
          docker run --rm -v $(pwd):/code gitleaks/gitleaks:latest \
            report --input=/code/gitleaks-report.json --output=/code/gitleaks.sarif --report-format sarif
          
      - name: Upload SARIF to GitHub Security tab
        if: always()
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: gitleaks.sarif
          category: gitleaks
          
      - name: Comment on PR with findings
        if: github.event_name == 'pull_request' && steps.gitleaks.outcome == 'failure'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = JSON.parse(fs.readFileSync('gitleaks-report.json', 'utf8'));
            
            let comment = '## 🔒 Gitleaks Secrets Scan Results\n\n';
            comment += '❌ **Secrets detected!** Please fix before merging.\n\n';
            comment += '### Findings:\n\n';
            
            report.forEach(leak => {
              comment += `- **${leak.Description}**\n`;
              comment += `  - File: \`${leak.File}\`\n`;
              comment += `  - Line: ${leak.StartLine}\n`;
              comment += `  - Secret: \`${leak.Secret.substring(0, 20)}...\`\n\n`;
            });
            
            comment += '### How to fix:\n';
            comment += '1. Remove the hardcoded secret\n';
            comment += '2. Use environment variables or secrets manager\n';
            comment += '3. If secret is real, rotate it immediately\n';
            comment += '4. Use `git filter-branch` to remove from history\n';
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
          
      - name: Fail workflow if secrets found
        if: steps.gitleaks.outcome == 'failure'
        run: |
          echo "❌ Secrets detected in repository!"
          echo "Please check the scan report above."
          exit 1
          
  # Job 3: Deep historical scan (slow but thorough)
  historical-scan:
    name: Historical Scan (Weekly)
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
    steps:
      - name: Checkout complete history
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Deep historical scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          args: |
            --verbose
            --redact
            --no-git  # Scan all historical commits
            --config custom-rules.toml
            --report-path historical-report.json
            
      - name: Upload historical report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: gitleaks-historical-report
          path: historical-report.json
          retention-days: 90
          
  # Job 4: Remediation guidance (for detected secrets)
  remediation:
    name: Provide Remediation Steps
    needs: full-scan
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - name: Output remediation steps
        run: |
          echo "========================================="
          echo "🔴 SECRETS DETECTED - REMEDIATION REQUIRED"
          echo "========================================="
          echo ""
          echo "IMMEDIATE ACTIONS:"
          echo "1. Revoke any exposed secrets immediately"
          echo "2. Rotate all affected credentials"
          echo "3. Check for unauthorized access"
          echo ""
          echo "CODE REMEDIATION:"
          echo "1. Remove secrets from code:"
          echo "   git filter-branch --force --index-filter \\"
          echo "   'git rm --cached --ignore-unmatch <file-with-secret>' \\"
          echo "   --prune-empty --tag-name-filter cat -- --all"
          echo ""
          echo "2. Use BFG Repo-Cleaner (easier):"
          echo "   java -jar bfg.jar --replace-text passwords.txt ."
          echo ""
          echo "3. Use environment variables instead:"
          echo "   // Bad: const API_KEY = 'abc123'"
          echo "   // Good: const API_KEY = process.env.API_KEY"
          echo ""
          echo "4. Use secrets manager (AWS Secrets Manager, HashiCorp Vault)"
          echo ""
          echo "PREVENTION:"
          echo "1. Keep .gitignore updated"
          echo "2. Use pre-commit hooks locally"
          echo "3. Never commit .env files"
          echo "4. Use CI/CD scans like this one"
          
      - name: Create GitHub Issue for tracking
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `🔒 Secrets Detected - ${new Date().toISOString()}`,
              body: 'A Gitleaks scan has detected hardcoded secrets in the repository.\n\nPlease check the workflow run for details and remediate immediately.',
              labels: ['security', 'critical', 'secrets-detected']
            });
```

### **Setting Up GitHub Actions Secret**:

```bash
# For Organization accounts only (personal accounts don't need this)
# Add to GitHub repository: Settings -> Secrets and variables -> Actions

# Secret name: GITLEAKS_LICENSE
# Value: Your Gitleaks license key from https://gitleaks.io
```

---

## 6️⃣ .dockerignore Configuration

### **Purpose**: Prevent secrets from being copied into Docker images

### **Location**: `/path/to/your-project/.dockerignore`

### **Complete Configuration**:

```dockerignore
# .dockerignore
# Prevents sensitive files from being included in Docker image

# Environment files
.env
.env.*
*.env

# Git files
.git
.gitignore
.gitattributes
.gitmodules

# Security keys
*.pem
*.key
*.crt
*.p12
*.pfx
id_rsa*
*.p8
*.p10

# Node.js
node_modules/
npm-debug.log
yarn-debug.log
yarn-error.log
package-lock.json
yarn.lock

# Build directories
dist/
build/
out/
target/
*.egg-info/

# Docker files
Dockerfile*
docker-compose*.yml
.dockerignore

# CI/CD
.github/
.gitlab/
.azure-pipelines/
circle.yml
.travis.yml

# IDE
.vscode/
.idea/
*.swp
*.swo
*~
.DS_Store

# Logs
*.log
logs/
*.pid

# Testing
coverage/
.nyc_output/
test/
__tests__/
jest.config.js

# Documentation (not needed in production)
README.md
CHANGELOG.md
CONTRIBUTING.md
docs/
*.md

# Secrets patterns
**/*secret*
**/*key*
**/*credential*
**/*password*
**/*token*

# Temporary files
tmp/
temp/
*.tmp
*.temp
```

### **Verification**:

```bash
# Build image and check contents
docker build -t myapp:test .

# Check if .env files were copied
docker run --rm myapp:test ls -la

# Scan image for secrets (using trivy)
trivy image --security-checks secret myapp:test
```

---

## 🔄 Complete Workflow Integration

### **Local Development Workflow**:

```bash
# 1. Clone repository
git clone https://github.com/your-org/your-project.git
cd your-project

# 2. Set up pre-commit hooks
pre-commit install
pre-commit install --hook-type commit-msg

# 3. Set up custom hooks
chmod +x .git/hooks/pre-commit

# 4. Test the setup
echo "test" > test.txt
git add test.txt
git commit -m "test"  # Should pass

# 5. Test with secret (should be blocked)
echo "password=mysecret123" > secret.txt
git add secret.txt
git commit -m "secret"  # Should FAIL

# 6. Reset test
git reset HEAD secret.txt
rm secret.txt
```

### **Continuous Integration Workflow**:

```yaml
# Example of full CI pipeline with security stages
name: Full CI Pipeline

on: [push, pull_request]

jobs:
  security-scan:
    uses: ./.github/workflows/gitleaks.yml
  
  code-scan:
    needs: security-scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run CodeQL
        uses: github/codeql-action/init@v2
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
  
  build:
    needs: code-scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: docker build -t myapp .
      - name: Scan image for secrets
        run: trivy image --severity HIGH,CRITICAL myapp
      - name: Push image
        if: github.ref == 'refs/heads/main'
        run: docker push myapp
```

---

## 🚨 Incident Response for Exposed Secrets

If a secret is accidentally committed:

```bash
# IMMEDIATE (within minutes):
# 1. Revoke the exposed secret
# 2. Rotate all related credentials
# 3. Check logs for unauthorized access

# MEDIUM-TERM (within hours):
# 1. Remove from Git history
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/file-with-secret" \
  --prune-empty --tag-name-filter cat -- --all

# OR use BFG Repo-Cleaner (easier)
java -jar bfg.jar --delete-files "*.env" .
java -jar bfg.jar --replace-text passwords.txt .

# 2. Force push clean history
git push origin --force --all
git push origin --force --tags

# 3. Notify team members to rebase
echo "⚠️ SECURITY INCIDENT: Secrets exposed, please rebase your branches"

# 4. Enable GitHub's push protection
# Settings -> Code security -> Secret scanning -> Push protection

# LONG-TERM:
# 1. Post-mortem: How did it happen?
# 2. Add stronger prevention controls
# 3. Implement secrets manager
# 4. Regular security training
```

---

## 📊 Monitoring & Alerts

### **Setting up alerts in GitHub**:

```yaml
# .github/workflows/secret-alerts.yml
name: Secret Alerting

on:
  security_advisory:
    types: [published]
  secret_scanning_alert:
    types: [created, resolved]

jobs:
  alert:
    runs-on: ubuntu-latest
    steps:
      - name: Send Slack alert
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "🚨 SECRET ALERT! Exposed secret detected in ${{ github.repository }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*🚨 CRITICAL: Secret Exposed!*"
                  }
                },
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "Repository: ${{ github.repository }}\nAction: ${{ github.event.action }}\nAlert: ${{ github.event.alert.type }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_SECURITY_WEBHOOK }}
```

---

## 📈 Best Practices Summary

| Layer | Tool | Purpose | Run Time |
|-------|------|---------|----------|
| **Local Development** | .gitignore | Prevent staging | Instant |
| **Pre-Commit** | Native hook | Quick checks | < 1 sec |
| **Pre-Commit** | Gitleaks | Professional scan | 2-5 sec |
| **Pre-Push** | Gitleaks full | History scan | 10-30 sec |
| **CI/CD** | GitHub Action | All branches | 1-2 min |
| **Scheduled** | Weekly scan | Historical | 5-10 min |
| **Docker** | .dockerignore | Image security | Build time |

### **Key Metrics to Track**:
- Number of secrets detected (should trend to 0)
- Time from commit to detection
- False positive rate (aim for < 5%)
- Incident response time

### **Tools Comparison**:

| Feature | Native Hook | Gitleaks | TruffleHog | GitHub Secret Scanning |
|---------|-------------|----------|------------|------------------------|
| Speed | Very Fast | Fast | Medium | N/A (Cloud) |
| Entropy detection | ❌ | ✅ | ✅ | ✅ |
| Custom rules | Basic | ✅ | ✅ | Limited |
| History scanning | ❌ | ✅ | ✅ | ✅ |
| Cost | Free | Free | Free | Included with GitHub |
| False positives | Low | Medium | High | Low |

---

## 🎯 Quick Start Checklist

```bash
# ✅ Initial Setup (15 minutes)
□ 1. Create .gitignore with all patterns
□ 2. Install pre-commit framework
□ 3. Create .pre-commit-config.yaml
□ 4. Run: pre-commit install
□ 5. Create custom-rules.toml
□ 6. Test with dummy secret

# ✅ CI/CD Setup (10 minutes)
□ 7. Create .github/workflows/gitleaks.yml
□ 8. Add GITLEAKS_LICENSE (if org account)
□ 9. Push to GitHub
□ 10. Verify Actions run

# ✅ Docker Security (5 minutes)
□ 11. Create .dockerignore
□ 12. Test Docker build locally
□ 13. Add to CI pipeline

# ✅ Team Training (30 minutes)
□ 14. Share this guide with team
□ 15. Demonstrate live blocking
□ 16. Create incident response plan
□ 17. Schedule quarterly review
```

This complete implementation provides defense-in-depth for secrets detection, preventing leaks at multiple stages from local development to production deployment.
