# AWS Security Agent Setup Guide for Online Boutique

This guide walks through setting up AWS Security Agent to perform security assessments on the Online Boutique microservices application.

---

## Prerequisites

- AWS Account with access to US East (N. Virginia) region
- GitHub repository with Online Boutique code
- (For penetration testing) Deployed instance of Online Boutique

---

## Step 1: Create Agent Space

1. Navigate to the [AWS Security Agent console](https://console.aws.amazon.com/security-agent)
2. Click **Set up AWS Security Agent**
3. Configure Agent Space:
   - **Agent Space name:** `online-boutique-security`
   - **Description:** `Security assessment for Online Boutique microservices demo`
4. Choose authentication method:
   - **SSO with IAM Identity Center** (recommended for teams)
   - **IAM-only access** (for quick setup)
5. Configure permissions - create default IAM role or select existing
6. Click **Set up AWS Security Agent**

---

## Step 2: Configure Security Requirements

### Enable Managed Requirements

1. Go to **Security requirements** in the navigation pane
2. Under **Managed security requirements**, enable:
   - Audit Logging Best Practices
   - Authentication Best Practices
   - Authorization Best Practices
   - Information Protection Best Practices
   - Secret Protection Best Practices
   - Secure by Default Best Practices

### Create Custom Requirements

Create these custom requirements for Online Boutique:

| Requirement Name | Description |
|-----------------|-------------|
| Service-to-Service mTLS Authentication | All internal service communication must use mutual TLS |
| Input Validation Best Practices | All user input must be validated before processing |
| XSS Prevention Controls | Dynamic HTML content must be properly encoded |
| SSRF Prevention Controls | Outbound requests must validate destination URLs |
| Path Traversal Prevention | File paths must be validated to prevent directory traversal |
| Command Injection Prevention | Shell commands must not include user input |

See `docs/aws-security-agent-requirements.md` for detailed requirement descriptions.

---

## Step 3: Design Review Setup

### Upload Design Documents

1. Go to your agent space and select **Design review** tab
2. Click **Start in web app** or **Admin access**
3. In the web application, click **Create design review**
4. Enter review name: `Online Boutique Architecture Review`
5. Upload these documents:
   - `docs/online-boutique-risk-assessment.md`
   - `docs/img/architecture-diagram.png`
   - `README.md`
6. Click **Start design review**

### Review Findings

After the review completes, examine:
- **Non-compliant** findings requiring design changes
- **Insufficient data** findings needing more documentation
- **Compliant** findings confirming security controls

---

## Step 4: Code Review Setup

### Connect GitHub Repository

1. Go to your agent space and select **Code review** tab
2. Click **Enable code review**
3. Click **Add** under GitHub connections
4. Authorize AWS Security Agent GitHub App
5. Select repositories:
   - `your-org/microservices-demo`
6. Enable code review for the repository
7. Select branch: `main` (or default)

### Configure PR Analysis

AWS Security Agent will automatically:
- Analyze new pull requests
- Check for OWASP Top Ten vulnerabilities
- Verify compliance with enabled security requirements
- Post findings as PR comments

### Test PRs Available

These vulnerability demonstration PRs are available for testing:

| PR | Vulnerability Type | Service |
|----|-------------------|---------|
| vuln/command-injection-shipping | Command Injection | shippingservice |
| vuln/hardcoded-secrets-payment | Hardcoded Secrets | paymentservice |
| vuln/path-traversal-catalog | Path Traversal | productcatalogservice |
| vuln/ssrf-recommendation | SSRF | recommendationservice |
| vuln/xss-search-frontend | XSS | frontend |

---

## Step 5: Penetration Testing Setup

### Configure Target Domain

1. Go to your agent space and select **Penetration test** tab
2. Click **Enable penetration test**
3. Under **Add and verify target domains**:
   - Enter your deployed Online Boutique domain
   - Select verification method (DNS TXT record)
4. Complete domain verification

### Configure VPC (for private endpoints)

If testing internal deployments:
1. Expand **VPC - Optional**
2. Select VPC, security group, and subnets
3. Ensure security group allows outbound traffic from Security Agent

### Add Context Sources

1. **GitHub repos:** Connect the microservices-demo repository
2. **S3 buckets:** Upload API specs, design docs
3. **CloudWatch:** Select log groups for application logs

### Create and Run Test

1. Access the Security Agent Web Application
2. Click **Create penetration test**
3. Configure:
   - **Name:** `Online Boutique Pentest - Jan 2026`
   - **Target URLs:** Your deployed frontend URL
   - **Authentication:** None required (demo app)
4. Click **Run penetration test**

### Review Results

The penetration test provides:
- **Findings** with severity ratings (Critical, High, Medium, Low)
- **Attack reasoning** explaining exploitation methods
- **Steps to reproduce** for validation
- **Remediation guidance** for fixes

---

## Expected Findings

### Design Review Expected Findings

| Requirement | Expected Status |
|------------|-----------------|
| Service-to-Service mTLS | Non-compliant |
| Encryption in Transit | Non-compliant (internal) |
| Network Segmentation | Insufficient data |
| Secret Protection | Compliant (with caveats) |

### Code Review Expected Findings (Vulnerability PRs)

| PR | Expected Finding |
|----|-----------------|
| command-injection-shipping | Command Injection - exec.Command with user input |
| hardcoded-secrets-payment | Hardcoded credentials in source code |
| path-traversal-catalog | Path traversal via unsanitized file path |
| ssrf-recommendation | SSRF via unvalidated URL fetch |
| xss-search-frontend | Reflected XSS via unescaped user input |

### Penetration Test Expected Findings

| Vulnerability | Severity | Location |
|--------------|----------|----------|
| Missing TLS | Medium | Frontend |
| Session fixation | Low | Cookie handling |
| Information disclosure | Low | Error messages |

---

## Remediation Workflow

1. **Review findings** in AWS Security Agent
2. **Prioritize** by severity and exploitability
3. **Create issues** in GitHub for tracking
4. **Implement fixes** following remediation guidance
5. **Re-run assessments** to verify fixes
6. **Mark resolved** in Security Agent

---

## Integration with CI/CD

### GitHub Actions Integration

AWS Security Agent automatically reviews PRs. Configure branch protection:

```yaml
# .github/workflows/security-gate.yml
name: Security Gate
on:
  pull_request:
    branches: [main]

jobs:
  wait-for-security-review:
    runs-on: ubuntu-latest
    steps:
      - name: Wait for AWS Security Agent
        run: |
          echo "Waiting for AWS Security Agent code review..."
          # PR checks will include Security Agent status
```

### Branch Protection Rules

1. Go to repository Settings > Branches
2. Add rule for `main` branch
3. Enable "Require status checks to pass"
4. Select AWS Security Agent check

---

## Useful Links

- [AWS Security Agent Documentation](https://docs.aws.amazon.com/security-agent/)
- [AWS Security Agent Console](https://console.aws.amazon.com/security-agent)
- [Online Boutique Risk Assessment](./online-boutique-risk-assessment.md)
- [Custom Security Requirements](./aws-security-agent-requirements.md)
