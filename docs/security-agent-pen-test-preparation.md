# AWS Security Agent Penetration Testing - Preparation Guide

This guide provides step-by-step instructions for preparing your Online Boutique application for penetration testing with AWS Security Agent.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Initial Setup](#initial-setup)
- [Domain Verification](#domain-verification)
- [Target Configuration](#target-configuration)
- [Available Endpoints for Penetration Testing](#available-endpoints-for-penetration-testing)
- [Context Gathering](#context-gathering)
- [Authentication Setup](#authentication-setup)
- [Network Configuration](#network-configuration)
- [Pre-Test Checklist](#pre-test-checklist)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Overview

AWS Security Agent provides on-demand penetration testing that discovers and validates vulnerabilities through multi-step attack scenarios. It tests against OWASP Top Ten vulnerabilities across 13 risk categories, including authentication, authorization, injection attacks, and more.

**Key Benefits:**
- Context-aware testing using your application's design, code, and specifications
- Dynamic attack adaptation based on application responses
- Comprehensive vulnerability discovery with detailed remediation guidance
- No scheduling delays - run tests on-demand

## Prerequisites

### AWS Account Requirements

- Active AWS account with appropriate permissions
- Access to US East (N. Virginia) region
- IAM permissions to create and manage:
  - AWS Security Agent resources
  - IAM roles and policies
  - CloudWatch log groups
  - VPC resources (if testing private endpoints)
  - AWS Secrets Manager (optional)
  - S3 buckets (optional for context storage)

### Application Requirements

- **Deployed Application**: Your Online Boutique instance must be running and accessible
- **Target URLs**: Public domain names or private endpoints within VPC
- **Domain Ownership**: Ability to verify domain ownership via DNS or other methods
- **Application Context** (recommended):
  - Source code repositories (GitHub)
  - API specifications (Protocol Buffers in `/protos/demo.proto`)
  - Architecture documentation
  - Design documents

### Team Requirements

- Security/AppSec team member to configure agent space
- Developer access to provide application context
- Access to authentication credentials for protected endpoints

## Initial Setup

### Step 1: Create Agent Space

1. Navigate to the [AWS Security Agent console](https://console.aws.amazon.com/security-agent/)
2. Choose **Set up AWS Security Agent**
3. Provide an **Agent Space Name**:
   - Use a descriptive name like `online-boutique-production` or `online-boutique-staging`
   - One agent space per application/environment is recommended
4. Add an optional **Description** for team context

### Step 2: Configure User Access

Choose one of two authentication methods:

#### Option A: SSO with IAM Identity Center (Recommended for Teams)
- Enables team-wide Single Sign-On access
- Centralized user management
- Best for organizations with multiple security team members

#### Option B: IAM-only Access
- Only AWS IAM principals can access the web application
- Simpler setup without SSO configuration
- Best for quick setup or single-user access

### Step 3: Configure Service Permissions

Create or select an IAM role for AWS Security Agent:

**Required Permissions:**
- VPC access (if testing private endpoints)
- CloudWatch Logs write access
- Secrets Manager read access (if using stored credentials)
- S3 read access (if using context from S3)

**Sample IAM Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:*:log-group:/aws/securityagent/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:*:secret:securityagent/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups"
      ],
      "Resource": "*"
    }
  ]
}
```

### Step 4: Complete Setup

Click **Set up AWS Security Agent** to create your agent space.

## Domain Verification

Before running penetration tests, you must verify ownership of all target domains.

### Verification Methods

#### DNS TXT Record (Recommended)
1. Navigate to your agent space configuration
2. Select **Penetration testing** > **Enable penetration test**
3. Add your target domain: `micro-demo.ops4life.com`
4. Choose **DNS txt record** as verification method
5. Copy the provided TXT record
6. Add the TXT record to your DNS configuration:
   ```
   # For micro-demo.ops4life.com
   _aws-security-agent-verification.micro-demo.ops4life.com TXT "verification-token"

   # Generic format
   _aws-security-agent-verification.yourdomain.com TXT "verification-token"
   ```
7. Wait for DNS propagation (can take up to 48 hours)
8. Verify DNS propagation:
   ```bash
   # Check if TXT record is visible
   dig TXT _aws-security-agent-verification.micro-demo.ops4life.com
   ```
9. Click **Verify** in the console

#### Other Verification Methods
- HTTP file upload
- Meta tag verification

### Online Boutique Deployment Scenarios

#### GKE with LoadBalancer
If using the default Kubernetes manifests with `frontend-external` service:
```bash
# Get your frontend external IP
kubectl get service frontend-external

# Expected output:
# NAME                TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
# frontend-external   LoadBalancer   10.x.x.x        <EXTERNAL-IP>   80:xxxxx/TCP   5m

# Verify domain points to this IP
dig +short micro-demo.ops4life.com

# Alternative: use nslookup
nslookup micro-demo.ops4life.com

# Test connectivity
curl -I https://micro-demo.ops4life.com
```

#### Minikube/Kind (Local Development)
For local testing, use port-forwarding and verify localhost or use a tunnel service like ngrok:
```bash
# Port forward frontend
kubectl port-forward deployment/frontend 8080:8080

# Or use ngrok for public access
ngrok http 8080
```

## Target Configuration

### Add and Verify Target Domains

1. Navigate to agent space > **Penetration testing**
2. Click **Enable penetration test**
3. Configure target domains:

**For Online Boutique:**
- Add your frontend domain
- Maximum 4 domains per agent space

### Target URL Examples

**Primary Frontend URL:**
```
https://micro-demo.ops4life.com
```

**Additional Target Examples:**

```
# Production deployment
https://boutique.example.com

# Staging environment
https://staging-boutique.example.com

# Specific service endpoints (if exposed)
https://api.boutique.example.com
```

**Recommended Target URLs for Penetration Testing:**

For comprehensive testing of the Online Boutique application, configure these target URLs:

1. **Frontend Application** (Primary Target):
   - `https://micro-demo.ops4life.com` - Main web interface
   - Tests: XSS, CSRF, session management, authentication bypass

2. **API Endpoints** (if separately exposed):
   - Add specific service endpoints if they're accessible externally
   - Example: `https://api.micro-demo.ops4life.com`

3. **Health Check Endpoints**:
   - Frontend: `https://micro-demo.ops4life.com/_healthz`
   - Useful for initial connectivity verification

**Testing Scope Recommendations:**

- **Start with**: `https://micro-demo.ops4life.com` (frontend only)
- **Expand to**: Individual microservice endpoints if exposed via Ingress/Gateway
- **Consider**: Different environment subdomains (dev, staging, prod)

## Available Endpoints for Penetration Testing

This section provides a comprehensive list of all HTTP endpoints available for security testing on your Online Boutique deployment.

### Frontend HTTP Endpoints

#### Main Application Routes

| URL | Method | Description | Attack Vectors to Test |
|-----|--------|-------------|----------------------|
| `/` | GET, HEAD | Home page - product listings | XSS, HTML injection |
| `/product/{id}` | GET, HEAD | Individual product details | Path traversal, IDOR, XSS |
| `/cart` | GET, HEAD | View shopping cart | Session hijacking, CSRF |
| `/cart` | POST | Add item to cart | CSRF, parameter tampering, business logic flaws |
| `/cart/empty` | POST | Empty shopping cart | CSRF, session manipulation |
| `/cart/checkout` | POST | Place order/checkout | Payment logic flaws, race conditions, CSRF |
| `/setCurrency` | POST | Change currency preference | CSRF, parameter tampering |
| `/logout` | GET | User logout | Session management flaws |
| `/assistant` | GET | Shopping assistant (AI feature) | Prompt injection, XSS |
| `/bot` | POST | Chatbot endpoint | Command injection, prompt injection, XSS |
| `/product-meta/{ids}` | GET | Product metadata API | IDOR, injection, information disclosure |

#### Static Resources

| URL | Method | Description | Attack Vectors to Test |
|-----|--------|-------------|----------------------|
| `/static/*` | GET | Static files (CSS, JS, images) | Path traversal, arbitrary file read |

#### System Endpoints

| URL | Method | Description | Attack Vectors to Test |
|-----|--------|-------------|----------------------|
| `/_healthz` | GET | Health check endpoint | Information disclosure |
| `/robots.txt` | GET | Robots exclusion file | Information gathering |

### Complete Target URLs for AWS Security Agent

#### Primary Targets (High Priority)

Configure these URLs as your primary targets in AWS Security Agent:

```
# Main application
https://micro-demo.ops4life.com/

# Product browsing
https://micro-demo.ops4life.com/product/OLJCESPC7Z
https://micro-demo.ops4life.com/product/66VCHSJNUP
https://micro-demo.ops4life.com/product/1YMWWN1N4O

# Shopping cart operations
https://micro-demo.ops4life.com/cart

# Checkout flow
https://micro-demo.ops4life.com/cart/checkout

# API endpoints
https://micro-demo.ops4life.com/product-meta/OLJCESPC7Z,66VCHSJNUP
```

**AI/Bot Features** (if enabled via environment variables):
```
https://micro-demo.ops4life.com/assistant
https://micro-demo.ops4life.com/bot
```

#### Secondary Targets (Medium Priority)

```
# Currency operations
https://micro-demo.ops4life.com/setCurrency

# Session management
https://micro-demo.ops4life.com/logout

# Cart management
https://micro-demo.ops4life.com/cart/empty
```

#### System Endpoints (Low Priority - Information Gathering)

```
# Health check
https://micro-demo.ops4life.com/_healthz

# Robots file
https://micro-demo.ops4life.com/robots.txt

# Static resources
https://micro-demo.ops4life.com/static/
```

### Backend gRPC Services

These services are internal and not directly accessible from the internet, but are called by the frontend:

| Service | Port | Description | Related Frontend Endpoints |
|---------|------|-------------|---------------------------|
| cartservice | 7070 | Shopping cart operations | `/cart`, `/cart/empty` |
| productcatalogservice | 3550 | Product catalog | `/`, `/product/{id}` |
| currencyservice | 7000 | Currency conversion | `/setCurrency`, all pages |
| recommendationservice | 8080 | Product recommendations | `/`, `/product/{id}` |
| shippingservice | 50051 | Shipping calculations | `/cart/checkout` |
| checkoutservice | 5050 | Order processing | `/cart/checkout` |
| adservice | 9555 | Advertisement serving | `/` |
| paymentservice | 50051 | Payment processing | `/cart/checkout` |
| emailservice | 5000 | Email notifications | `/cart/checkout` |

**Note**: These gRPC services cannot be directly tested via AWS Security Agent but vulnerabilities in frontend endpoints may expose underlying service issues.

### Recommended Penetration Test Scenarios

#### 1. Authentication & Session Management
**Test Endpoints:**
- `POST /cart`
- `POST /cart/checkout`
- `GET /logout`

**Scenarios:**
- Session fixation attacks
- Session hijacking via cookie theft
- Logout functionality bypass
- Session ID predictability

#### 2. Input Validation & Injection
**Test Endpoints:**
- `GET /product/{id}` - Test product ID parameter
- `POST /cart` - Test product_id and quantity parameters
- `POST /setCurrency` - Test currency_code parameter
- `GET /product-meta/{ids}` - Test multiple IDs

**Scenarios:**
- SQL injection in product IDs
- NoSQL injection in cart operations
- XSS in product names/descriptions
- Path traversal in static file access
- Command injection in bot/assistant features

#### 3. Business Logic Flaws
**Test Endpoints:**
- `POST /cart/checkout`
- `POST /cart`
- `POST /cart/empty`

**Scenarios:**
- Price manipulation during checkout
- Negative quantity orders
- Race conditions in order placement
- Cart manipulation after checkout initiation
- Discount code abuse

#### 4. CSRF Protection
**Test All POST Endpoints:**
- `POST /cart`
- `POST /cart/empty`
- `POST /cart/checkout`
- `POST /setCurrency`
- `POST /bot`

**Scenarios:**
- CSRF token validation bypass
- Cross-origin request testing
- SameSite cookie attribute testing

#### 5. Cross-Site Scripting (XSS)
**Test Endpoints:**
- `GET /` - Test search or filter parameters
- `GET /product/{id}` - Test reflected parameters
- `GET /cart` - Test stored cart data
- `GET /assistant` - Test AI assistant inputs/outputs

**Scenarios:**
- Reflected XSS in URL parameters
- Stored XSS in cart items
- DOM-based XSS in JavaScript
- XSS via product names or descriptions

#### 6. Information Disclosure
**Test Endpoints:**
- `GET /_healthz`
- `GET /robots.txt`
- Error pages (invalid product IDs, etc.)
- `GET /static/*`

**Scenarios:**
- Verbose error messages
- Stack traces in responses
- Version information disclosure
- Source code disclosure via static files
- Internal IP/service discovery

### Configuring AWS Security Agent

When setting up your penetration test in AWS Security Agent:

1. **Start with Primary Targets**:
   ```
   https://micro-demo.ops4life.com
   ```

2. **Provide Application Context**:
   - Upload Protocol Buffer definitions from `/protos/demo.proto`
   - Connect GitHub repository for source code context
   - Provide architecture documentation

3. **Configure Authentication**:
   - Session-based authentication (shop_session-id cookie)
   - No login required - sessions auto-generated

4. **Enable Test Categories**:
   - ✅ Injection attacks
   - ✅ Authentication/Authorization
   - ✅ Session management
   - ✅ Business logic flaws
   - ✅ XSS/CSRF
   - ✅ Information disclosure

5. **Set Rate Limiting**:
   - Start with 10 requests/second
   - Increase if application handles load well

### Sample Product IDs for Testing

Use these valid product IDs when testing product-related endpoints:

```
OLJCESPC7Z  # Sunglasses
66VCHSJNUP  # Tank Top
1YMWWN1N4O  # Watch
L9ECAV7KIM  # Loafers
2ZYFJ3GM2N  # Candle Holder
0PUK6V6EV0  # Hairdryer
LS4PSXUNUM  # Salt & Pepper Shakers
9SIQT8TOJO  # Bamboo Glass Jar
6E92ZMYYFZ  # Mug
```

These IDs can be used to test:
- `/product/{id}` endpoints
- `/product-meta/{ids}` API
- Cart add operations with various product combinations

## Context Gathering

Provide application context to improve testing accuracy and depth.

### Source Code (GitHub)

Connect your GitHub repository to provide AWS Security Agent with application context:

1. Navigate to **Code review** tab in agent configuration
2. Click **Connect GitHub**
3. Authorize AWS Security Agent
4. Select repositories:
   - Main application repository
   - Include all microservice source code

**For Online Boutique:**
```
Repository: GoogleCloudPlatform/microservices-demo
Branch: main (or your fork)
```

### API Specifications

AWS Security Agent can use Protocol Buffer definitions to understand service contracts:

**Option 1: GitHub Integration** (Automatic)
If connected to GitHub, protobuf files are automatically discovered.

**Option 2: S3 Upload**
1. Upload `/protos/demo.proto` to S3
2. Configure S3 resource in penetration test setup:
   ```
   s3://your-bucket/context/demo.proto
   ```

### Architecture Documentation

Upload design documents to provide business logic context:

**Recommended Documents:**
- Architecture diagrams (`/docs/architecture-diagram.png`)
- Service interaction flows
- Authentication/authorization models
- Data flow diagrams

**Upload via:**
- Design review feature (up to 5 files)
- S3 bucket configuration
- Direct file upload in web application

## Authentication Setup

Configure authentication credentials for testing protected endpoints.

### Session-Based Authentication (Online Boutique)

The Online Boutique frontend uses session IDs. Configure test credentials:

1. Navigate to **Penetration testing** > **Setup**
2. Expand **Secrets** section
3. Create a secret in AWS Secrets Manager:

```bash
# Create authentication secret
aws secretsmanager create-secret \
    --name securityagent/online-boutique/session \
    --description "Session credentials for Online Boutique testing" \
    --secret-string '{
      "session_id": "test-session-12345",
      "user_id": "test-user"
    }'
```

4. Select the secret in the penetration test configuration

### API Key Authentication

If you've added API authentication:

```bash
aws secretsmanager create-secret \
    --name securityagent/online-boutique/api-key \
    --secret-string '{
      "api_key": "your-api-key",
      "header_name": "X-API-Key"
    }'
```

### Basic Authentication

For services with basic auth:

```bash
aws secretsmanager create-secret \
    --name securityagent/online-boutique/basic-auth \
    --secret-string '{
      "username": "admin",
      "password": "secure-password"
    }'
```

## Network Configuration

### Public Endpoints (Default)

No additional configuration needed if your application is publicly accessible.

### Private Endpoints (VPC)

For applications deployed in private subnets:

1. Navigate to **Penetration testing** > **Setup**
2. Expand **VPC** section
3. Configure VPC settings:

**Required Information:**
- VPC ID
- Subnet IDs (at least 2 for high availability)
- Security Group ID

**Security Group Configuration:**
```bash
# Allow inbound from AWS Security Agent
# Add ingress rules for your application ports

aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxx \
    --protocol tcp \
    --port 80 \
    --cidr 10.0.0.0/16
```

### GKE Private Cluster

For GKE private clusters, use one of these approaches:

#### Option 1: Expose via LoadBalancer
```yaml
# frontend-external service already does this
apiVersion: v1
kind: Service
metadata:
  name: frontend-external
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
```

#### Option 2: VPC Peering
Set up VPC peering between AWS and GCP to allow AWS Security Agent to reach private GKE endpoints.

## CloudWatch Configuration

Configure logging to monitor penetration test execution:

1. Navigate to **Penetration testing** > **Setup**
2. Expand **CloudWatch configuration**
3. Select or create a log group:

```bash
# Create log group
aws logs create-log-group \
    --log-group-name /aws/securityagent/online-boutique
```

AWS Security Agent will automatically create one if not specified.

## Pre-Test Checklist

Before running your first penetration test, verify:

- [ ] Agent space created and configured
- [ ] Target domains added and verified
- [ ] Application is deployed and accessible
- [ ] GitHub repository connected (optional but recommended)
- [ ] Protocol Buffer specifications available
- [ ] Authentication credentials stored in Secrets Manager (if needed)
- [ ] VPC configuration completed (for private endpoints)
- [ ] CloudWatch log group created
- [ ] IAM service role has required permissions
- [ ] Team members have access to Security Agent Web Application

## Best Practices

### Test Scope Management

1. **Start Small**: Begin with a single target URL to understand behavior
2. **Isolate Environments**: Test staging before production
3. **Schedule Tests**: Run during low-traffic periods for production testing
4. **Incremental Context**: Add context gradually to see impact on findings

### Context Optimization

**Minimum Context:**
- Target URL
- Basic authentication credentials

**Recommended Context:**
- Source code repository
- API specifications (proto files)
- Authentication credentials

**Maximum Context:**
- Source code repository
- API specifications
- Architecture documentation
- Design documents
- Business logic documentation
- S3 buckets with additional resources

### Security Considerations

1. **Credential Management**:
   - Use AWS Secrets Manager, never hardcode credentials
   - Rotate test credentials regularly
   - Use dedicated test accounts, not production accounts

2. **Network Security**:
   - Use security groups to restrict access
   - Consider VPC endpoints for private testing
   - Enable VPC Flow Logs for audit trails

3. **Data Protection**:
   - Test with synthetic data when possible
   - Avoid testing with production PII
   - Configure CloudWatch log retention policies

### Cost Optimization

- AWS Security Agent is **free during preview**
- Monitor CloudWatch logs storage costs
- Clean up old test runs and logs
- Use IAM-only access for single users instead of IAM Identity Center

## Troubleshooting

### Domain Verification Failures

**Issue**: DNS TXT record not detected

**Solutions**:
```bash
# Verify DNS propagation for your domain
dig TXT _aws-security-agent-verification.micro-demo.ops4life.com

# Check with multiple DNS servers
dig @8.8.8.8 TXT _aws-security-agent-verification.micro-demo.ops4life.com
dig @1.1.1.1 TXT _aws-security-agent-verification.micro-demo.ops4life.com

# Check TTL settings
dig TXT _aws-security-agent-verification.micro-demo.ops4life.com +noall +answer
```
- Check TTL settings and wait for cache expiration
- Try alternate verification methods (HTTP file, meta tag)
- Ensure no DNSSEC validation issues

### Connection Failures

**Issue**: Cannot reach target application

**Solutions**:
```bash
# Verify application is running
kubectl get pods -l app=frontend

# Check service endpoints
kubectl get services frontend frontend-external

# Verify LoadBalancer has external IP
kubectl get service frontend-external -o wide

# Test DNS resolution
dig +short micro-demo.ops4life.com

# Test connectivity
curl -I https://micro-demo.ops4life.com

# Verbose connection test
curl -v https://micro-demo.ops4life.com

# Check if specific paths are accessible
curl https://micro-demo.ops4life.com/_healthz
```
- Review security group rules for VPC configurations
- Check CloudWatch logs for detailed error messages
- Verify SSL/TLS certificate is valid:
  ```bash
  openssl s_client -connect micro-demo.ops4life.com:443 -servername micro-demo.ops4life.com
  ```

### Authentication Issues

**Issue**: Test cannot authenticate to application

**Solutions**:
- Verify secret exists: `aws secretsmanager get-secret-value --secret-id securityagent/online-boutique/session`
- Check IAM role has Secrets Manager read permissions
- Validate credential format matches application expectations
- Test credentials manually before configuring agent

### Context Not Being Used

**Issue**: Tests don't seem to use provided context

**Solutions**:
- Verify GitHub repository is connected and accessible
- Check proto files are in correct path (`/protos/`)
- Ensure S3 bucket permissions allow read access
- Review CloudWatch logs for context loading errors
- Re-upload design documents if needed

### Performance Issues

**Issue**: Tests taking too long or timing out

**Solutions**:
- Reduce testing scope (fewer URLs, specific endpoints)
- Increase request timeout in configuration
- Check target application performance
- Review CloudWatch logs for bottlenecks
- Scale up target application resources

## Next Steps

After completing preparation:

1. **Launch Web Application**: Click **Admin access** in agent configuration
2. **Create Penetration Test**: Use the web interface to configure your first test
3. **Review Findings**: Analyze discovered vulnerabilities
4. **Remediate Issues**: Follow provided guidance to fix vulnerabilities
5. **Re-test**: Run tests again to verify fixes

## Additional Resources

- [AWS Security Agent Documentation](https://docs.aws.amazon.com/security-agent/)
- [AWS Security Agent Console](https://console.aws.amazon.com/security-agent/)
- [OWASP Top Ten](https://owasp.org/www-project-top-ten/)
- [Online Boutique Architecture](/docs/development-guide.md)
- [Security Agent Design Review](/aws_security_agent.pdf)

## Support

**Preview Period Support:**
- AWS Security Agent Forum
- AWS Support (for account/billing issues)

**Issues with this implementation:**
- GitHub Issues: [GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo/issues)

---

**Note**: AWS Security Agent is currently in preview and available only in the US East (N. Virginia) region. The service is free during the preview period.
