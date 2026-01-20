# Custom Security Requirements for AWS Security Agent

These custom security requirements are designed for use with AWS Security Agent's design review and code review capabilities for the Online Boutique microservices application.

---

## Custom Security Requirements

### 1. Service-to-Service Authentication Required

**Requirement Name:** `Service-to-Service mTLS Authentication`

**Description:** All internal service-to-service communication must use mutual TLS (mTLS) authentication. Services must not accept unauthenticated requests from other services within the cluster.

**Applicability:** All microservices performing inter-service communication

**Compliance Criteria:**
- mTLS certificates configured for all gRPC endpoints
- Service mesh (Istio) or equivalent mTLS solution deployed
- No plaintext gRPC communication between services

---

### 2. No Hardcoded Secrets in Source Code

**Requirement Name:** `Secret Protection Best Practices`

**Description:** Source code must not contain hardcoded credentials, API keys, passwords, or other sensitive secrets. All secrets must be retrieved from secure secret management systems at runtime.

**Applicability:** All services and configuration files

**Compliance Criteria:**
- No API keys, passwords, or tokens in source code
- No connection strings with embedded credentials
- Secrets loaded from Kubernetes Secrets, Secret Manager, or Vault
- No secrets in environment variable defaults in Dockerfiles

---

### 3. Input Validation for All External Data

**Requirement Name:** `Input Validation Best Practices`

**Description:** All user-supplied input must be validated and sanitized before processing. This includes HTTP request parameters, headers, cookies, and gRPC message fields that originate from external sources.

**Applicability:** Frontend service and any service receiving external input

**Compliance Criteria:**
- Input length limits enforced
- Input format validation (regex, type checking)
- Special characters escaped or rejected
- No direct use of user input in file paths, shell commands, or SQL queries

---

### 4. Output Encoding for HTML Responses

**Requirement Name:** `XSS Prevention Controls`

**Description:** All dynamic content rendered in HTML responses must be properly encoded to prevent Cross-Site Scripting (XSS) attacks. User-supplied data must never be directly embedded in HTML without encoding.

**Applicability:** Frontend service and any service generating HTML

**Compliance Criteria:**
- HTML entity encoding for user data in HTML context
- JavaScript encoding for user data in script context
- URL encoding for user data in URL context
- Use of secure templating engines with auto-escaping

---

### 5. URL Validation for Outbound Requests

**Requirement Name:** `SSRF Prevention Controls`

**Description:** Services making outbound HTTP requests must validate destination URLs to prevent Server-Side Request Forgery (SSRF) attacks. Requests to internal networks, metadata endpoints, and localhost must be blocked.

**Applicability:** All services making outbound HTTP/HTTPS requests

**Compliance Criteria:**
- Allowlist of permitted destination domains
- Block requests to private IP ranges (10.x, 172.16-31.x, 192.168.x)
- Block requests to cloud metadata endpoints (169.254.169.254)
- Block requests to localhost and loopback addresses

---

### 6. File Path Validation

**Requirement Name:** `Path Traversal Prevention`

**Description:** Services handling file operations must validate file paths to prevent path traversal attacks. User-supplied input must not be used to construct file paths without proper validation.

**Applicability:** All services performing file system operations

**Compliance Criteria:**
- Canonicalize file paths before access
- Validate paths stay within allowed directories
- Reject paths containing "../" sequences
- Use allowlist of permitted file extensions

---

### 7. Command Injection Prevention

**Requirement Name:** `Command Injection Prevention`

**Description:** Services must not execute shell commands with user-supplied input. If shell execution is required, input must be strictly validated and parameterized.

**Applicability:** All services

**Compliance Criteria:**
- Avoid shell execution with user input
- Use parameterized APIs instead of shell commands
- If shell required, use allowlist validation
- Never use string concatenation for command construction

---

### 8. Encryption in Transit Required

**Requirement Name:** `Data Encryption in Transit`

**Description:** All data transmission must be encrypted. External traffic must use TLS 1.2 or higher. Internal service communication should use mTLS.

**Applicability:** All services and network communication

**Compliance Criteria:**
- TLS 1.2+ for all external endpoints
- Valid certificates from trusted CA
- No HTTP endpoints for sensitive data
- mTLS for internal service communication

---

### 9. Sensitive Data Logging Prohibited

**Requirement Name:** `Log Protection Best Practices`

**Description:** Application logs must not contain sensitive data including passwords, credit card numbers, API keys, or personally identifiable information (PII).

**Applicability:** All services

**Compliance Criteria:**
- No passwords or secrets in logs
- Credit card numbers masked or excluded
- PII (email, addresses) redacted in logs
- Session tokens not logged in full

---

### 10. Rate Limiting for Public Endpoints

**Requirement Name:** `Rate Limiting Best Practices`

**Description:** Public-facing endpoints must implement rate limiting to prevent abuse and denial-of-service attacks.

**Applicability:** Frontend service and any public API endpoints

**Compliance Criteria:**
- Rate limits configured per IP/session
- Appropriate limits for different endpoint types
- Graceful degradation under load
- Rate limit headers in responses

---

### 11. Container Security Best Practices

**Requirement Name:** `Container Security Controls`

**Description:** Container images must follow security best practices including non-root users, minimal base images, and no unnecessary privileges.

**Applicability:** All service containers

**Compliance Criteria:**
- Containers run as non-root user
- Read-only root filesystem where possible
- No privileged containers
- Minimal base images (distroless preferred)
- No unnecessary capabilities

---

### 12. Network Segmentation Defined

**Requirement Name:** `Network Segmentation Strategy`

**Description:** The design must define clear network segmentation separating workload components into logical layers based on data sensitivity and function.

**Applicability:** Overall architecture design

**Compliance Criteria:**
- Network policies defined for pod-to-pod communication
- Egress restrictions for sensitive services
- Separation between public and internal services
- Database/cache access restricted to required services only

---

## How to Use with AWS Security Agent

### Design Review
1. Navigate to AWS Security Agent console
2. Go to Security Requirements > Custom Security Requirements
3. Create each requirement above with the name and description
4. Enable the requirements for your agent space
5. Upload `online-boutique-risk-assessment.md` for design review

### Code Review
1. Connect your GitHub repository to AWS Security Agent
2. Enable code review for the repository
3. The agent will check pull requests against these requirements
4. Review findings and remediation guidance in PR comments

### Penetration Testing
1. Deploy the application to a test environment
2. Configure target domains in AWS Security Agent
3. Verify domain ownership
4. Run penetration test with source code context from GitHub
