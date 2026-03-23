<p align="center">
  <img src="assets/logo.svg" alt="TerraGuard" width="160"/>
</p>

<h1 align="center">TerraGuard</h1>
<p align="center">
  <strong>AI-powered security analysis for Terraform / HCL code</strong><br/>
  Detect misconfigurations, overpermissioned IAM, exposed resources, and hardcoded secrets — via a simple REST API.
</p>

<p align="center">
  <a href="https://rapidapi.com/terrycrews99/api/terraguard"><img src="https://img.shields.io/badge/RapidAPI-TerraGuard-blue?style=flat-square&logo=rapid" alt="RapidAPI"/></a>
  <img src="https://img.shields.io/badge/python-3.12-blue?style=flat-square"/>
  <img src="https://img.shields.io/badge/fastapi-0.115-green?style=flat-square"/>
  <img src="https://img.shields.io/badge/license-MIT-lightgrey?style=flat-square"/>
</p>

<p align="center">
  <a href="https://rapidapi.com/terrycrews99/api/terraguard"><strong>→ Get API Access on RapidAPI</strong></a>
</p>

---

## What it does

TerraGuard analyzes Terraform HCL code or diffs and returns structured JSON identifying security risks before they reach production.

**What it detects:**
- Overpermissioned IAM (wildcards, admin roles)
- Security Groups open to `0.0.0.0/0`
- Unencrypted storage (S3, RDS, EBS)
- Public access on private resources
- Missing logging and monitoring
- Hardcoded secrets, API keys, passwords

---

## Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | API status |
| POST | `/analyze` | Security analysis of HCL code |
| POST | `/secrets` | Hardcoded secrets detection |

**Request body (both POST endpoints):**
```json
{ "hcl": "<terraform code or diff content>" }
```

**Limits:** 8,000 chars max (auto-truncated), 100KB body, 10 req/s per IP.

---

## Quick Start

### 1. Get your API key

Subscribe on [RapidAPI](https://rapidapi.com/terrycrews99/api/terraguard) — free tier includes 30 requests/month.

### 2. Analyze your Terraform

```bash
HCL=$(cat main.tf | jq -Rs .)

curl -X POST https://terraguard.p.rapidapi.com/analyze \
  -H "Content-Type: application/json" \
  -H "X-RapidAPI-Key: YOUR_API_KEY" \
  -H "X-RapidAPI-Host: terraguard.p.rapidapi.com" \
  -d "{\"hcl\": $HCL}"
```

---

## Examples

### `POST /analyze`

**Request:**
```json
{
  "hcl": "resource \"aws_security_group\" \"web\" {\n  ingress {\n    from_port   = 0\n    to_port     = 0\n    protocol    = \"-1\"\n    cidr_blocks = [\"0.0.0.0/0\"]\n  }\n}"
}
```

**Response:**
```json
{
  "summary": "Security group allows unrestricted inbound traffic from any IP, posing a critical network exposure risk.",
  "risk_level": "CRITICAL",
  "issues": [
    {
      "severity": "CRITICAL",
      "category": "NETWORK",
      "title": "Overly Permissive Security Group Ingress",
      "description": "Allows all traffic (protocol -1, ports 0-0) from 0.0.0.0/0, exposing resources to the entire internet.",
      "resource": "aws_security_group.web",
      "recommendation": "Restrict to specific ports and trusted CIDR ranges."
    }
  ],
  "passed_checks": [],
  "total_issues": 1
}
```

### `POST /secrets`

**Request:**
```json
{
  "hcl": "resource \"aws_db_instance\" \"db\" {\n  password = \"MySuperSecret123\"\n  username = \"admin\"\n}"
}
```

**Response:**
```json
{
  "secrets_found": true,
  "risk_level": "CRITICAL",
  "findings": [
    {
      "severity": "CRITICAL",
      "type": "PASSWORD",
      "description": "Hardcoded database password found in aws_db_instance resource.",
      "location": "aws_db_instance.db.password",
      "recommendation": "Use var.db_password with sensitive=true, or retrieve from AWS Secrets Manager / SSM Parameter Store."
    }
  ],
  "total_findings": 1,
  "remediation_summary": "Move all secrets to environment variables or a secrets manager."
}
```

---

## Integrations

### Pre-commit hook

Save as `.git/hooks/pre-commit` and make it executable:

```bash
#!/bin/bash
TF_FILES=$(git diff --cached --name-only | grep '\.tf$')
if [ -z "$TF_FILES" ]; then exit 0; fi

HCL=$(git diff --cached -- $TF_FILES | jq -Rs .)

RESULT=$(curl -s -X POST https://terraguard.p.rapidapi.com/analyze \
  -H "Content-Type: application/json" \
  -H "X-RapidAPI-Key: $RAPIDAPI_KEY" \
  -H "X-RapidAPI-Host: terraguard.p.rapidapi.com" \
  -d "{\"hcl\": $HCL}")

RISK=$(echo $RESULT | jq -r '.risk_level')
if [ "$RISK" = "CRITICAL" ] || [ "$RISK" = "HIGH" ]; then
  echo "TerraGuard: $RISK risk detected. Review before committing."
  echo $RESULT | jq '.issues[] | "[" + .severity + "] " + .title'
  exit 1
fi
```

### GitHub Actions

```yaml
name: TerraGuard Security Scan
on: [pull_request]

jobs:
  terraguard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Get changed Terraform files
        id: tf
        run: |
          FILES=$(git diff HEAD~1 --name-only | grep '\.tf$' | xargs cat 2>/dev/null | jq -Rs .)
          echo "hcl=$FILES" >> $GITHUB_OUTPUT

      - name: TerraGuard analyze
        run: |
          curl -s -X POST https://terraguard.p.rapidapi.com/analyze \
            -H "Content-Type: application/json" \
            -H "X-RapidAPI-Key: ${{ secrets.RAPIDAPI_KEY }}" \
            -H "X-RapidAPI-Host: terraguard.p.rapidapi.com" \
            -d "{\"hcl\": ${{ steps.tf.outputs.hcl }}}" | jq .
```

### Python

```python
import subprocess
import requests

with open("main.tf") as f:
    hcl = f.read()

response = requests.post(
    "https://terraguard.p.rapidapi.com/analyze",
    headers={
        "Content-Type": "application/json",
        "X-RapidAPI-Key": "YOUR_API_KEY",
        "X-RapidAPI-Host": "terraguard.p.rapidapi.com",
    },
    json={"hcl": hcl}
)

result = response.json()
print(f"Risk level: {result['risk_level']}")
for issue in result.get("issues", []):
    print(f"[{issue['severity']}] {issue['title']} — {issue['resource']}")
```

---

## Use Cases

- **CI/CD gates** — block pipelines when CRITICAL issues are found before `terraform apply`
- **Pre-commit hooks** — catch misconfigurations before they enter version control
- **PR review bots** — post security findings as comments on Terraform PRs
- **Scheduled audits** — scan your entire IaC codebase on a schedule
- **IDE integrations** — surface findings directly in your editor

---

## Pricing

Available on [RapidAPI](https://rapidapi.com/terrycrews99/api/terraguard):

| Plan | Price | Requests/month |
|------|-------|----------------|
| Free | $0 | 30 |
| Pro ⭐ | $19 | 500 |
| Ultra | $59 | 5,000 |

---

## License

MIT
