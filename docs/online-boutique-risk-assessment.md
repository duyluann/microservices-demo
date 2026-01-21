# Online Boutique Security Risk Assessment

## Document Information
- **Application Name:** Online Boutique
- **Version:** 1.0
- **Date:** January 2026
- **Classification:** Internal

---

## 1. Executive Summary

Online Boutique is a cloud-native e-commerce microservices demonstration application. This document describes the security architecture, data flows, and security controls implemented across the 11 microservices that comprise the application.

---

## 2. System Architecture Overview

### 2.1 High-Level Architecture

```
                                    ┌─────────────────┐
                                    │   Load Balancer │
                                    │   (External IP) │
                                    └────────┬────────┘
                                             │
                                             ▼
┌────────────────────────────────────────────────────────────────────────┐
│                           Kubernetes Cluster                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        Frontend Service                          │   │
│  │                     (Go - HTTP Server)                           │   │
│  │              Handles user sessions, serves web UI                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│            ┌───────────┬───────────┼───────────┬───────────┐           │
│            ▼           ▼           ▼           ▼           ▼           │
│     ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│     │  Cart    │ │ Product  │ │ Currency │ │ Checkout │ │   Ad     │  │
│     │ Service  │ │ Catalog  │ │ Service  │ │ Service  │ │ Service  │  │
│     │  (C#)    │ │  (Go)    │ │ (Node.js)│ │  (Go)    │ │ (Java)   │  │
│     └────┬─────┘ └──────────┘ └──────────┘ └────┬─────┘ └──────────┘  │
│          │                                      │                      │
│          ▼                                      ▼                      │
│     ┌──────────┐                    ┌──────────────────────┐          │
│     │  Redis   │                    │   Payment Service    │          │
│     │  Cache   │                    │      (Node.js)       │          │
│     └──────────┘                    ├──────────────────────┤          │
│                                     │   Shipping Service   │          │
│                                     │        (Go)          │          │
│                                     ├──────────────────────┤          │
│                                     │    Email Service     │          │
│                                     │      (Python)        │          │
│                                     ├──────────────────────┤          │
│                                     │ Recommendation Svc   │          │
│                                     │      (Python)        │          │
│                                     └──────────────────────┘          │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Service Inventory

| Service | Language | Port | Protocol | Data Sensitivity |
|---------|----------|------|----------|------------------|
| frontend | Go | 8080 | HTTP | Medium (sessions) |
| cartservice | C# | 7070 | gRPC | Low |
| productcatalogservice | Go | 3550 | gRPC | Low |
| currencyservice | Node.js | 7000 | gRPC | Low |
| paymentservice | Node.js | 50051 | gRPC | **High (payment data)** |
| shippingservice | Go | 50051 | gRPC | Medium (addresses) |
| emailservice | Python | 8080 | gRPC | Medium (PII) |
| checkoutservice | Go | 5050 | gRPC | **High (orchestrates payment)** |
| recommendationservice | Python | 8080 | gRPC | Low |
| adservice | Java | 9555 | gRPC | Low |
| redis-cart | Redis | 6379 | Redis | Medium (cart data) |

---

## 3. Data Flow Analysis

### 3.1 User Purchase Flow

1. **User browses products** (Frontend → ProductCatalog)
   - No authentication required
   - Session ID generated automatically

2. **User adds items to cart** (Frontend → CartService → Redis)
   - Cart stored with session ID as key
   - No encryption at rest in default Redis configuration

3. **User initiates checkout** (Frontend → CheckoutService)
   - Checkout service orchestrates downstream calls
   - Payment data transmitted to PaymentService

4. **Payment processing** (CheckoutService → PaymentService)
   - Credit card data processed (mock implementation)
   - Transaction ID returned

5. **Order confirmation** (CheckoutService → EmailService)
   - Email sent with order details (mock)

### 3.2 Data Classification

| Data Type | Classification | Services Handling |
|-----------|---------------|-------------------|
| Credit Card Numbers | **PCI-DSS Sensitive** | paymentservice, checkoutservice |
| Email Addresses | PII | emailservice, checkoutservice |
| Shipping Addresses | PII | shippingservice, checkoutservice |
| Session IDs | Internal | frontend, cartservice |
| Product Data | Public | productcatalogservice |
| Cart Contents | Internal | cartservice, redis |

---

## 4. Authentication and Authorization

### 4.1 Current Implementation

**User Authentication:** None required
- Application generates anonymous session IDs
- No user accounts or login functionality
- Sessions are ephemeral and tied to browser cookies

**Service-to-Service Authentication:** None
- All internal gRPC calls are unauthenticated
- Services trust all incoming requests from within the cluster

### 4.2 Security Implications

- No authorization controls between services
- Any compromised service can call any other service
- No audit trail for service-to-service calls

### 4.3 Recommended Improvements

1. Implement mutual TLS (mTLS) between services
2. Deploy Istio service mesh for identity and access management
3. Add JWT-based authentication for service-to-service calls

---

## 5. Network Security

### 5.1 Network Boundaries

```
┌─────────────────────────────────────────────────────────────┐
│                     EXTERNAL ZONE                            │
│                    (Internet Traffic)                        │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS (recommended) / HTTP
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      DMZ ZONE                                │
│                 (Load Balancer, Ingress)                     │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   APPLICATION ZONE                           │
│              (Frontend Service - Port 8080)                  │
└──────────────────────────┬──────────────────────────────────┘
                           │ gRPC
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    INTERNAL ZONE                             │
│     (All backend microservices - gRPC communication)         │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      DATA ZONE                               │
│                 (Redis Cache - Port 6379)                    │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Network Policies

**Default Configuration:** No network policies enforced
- All pods can communicate with all other pods
- No egress restrictions

**Recommended Configuration:** Enable network policies component
- Restrict pod-to-pod communication to required paths only
- Block unnecessary egress traffic

### 5.3 Exposed Ports

| Port | Service | Exposure | Risk Level |
|------|---------|----------|------------|
| 80/443 | frontend-external | Public | Medium |
| 8080 | frontend | Cluster-internal | Low |
| 6379 | redis-cart | Cluster-internal | **High if exposed** |
| 50051 | paymentservice | Cluster-internal | **High** |

---

## 6. Data Protection

### 6.1 Encryption at Rest

| Component | Encryption Status | Key Management |
|-----------|------------------|----------------|
| Redis (default) | **Not encrypted** | N/A |
| Redis (Memorystore) | Encrypted | Google-managed |
| Spanner (optional) | Encrypted | Google-managed or CMK |
| Product catalog JSON | Not encrypted | N/A |

### 6.2 Encryption in Transit

| Communication Path | Encryption | Protocol |
|-------------------|------------|----------|
| User → Frontend | **Recommended: TLS** | HTTPS |
| Frontend → Backend Services | **None (default)** | gRPC |
| Backend → Backend | **None (default)** | gRPC |
| Backend → Redis | **None (default)** | Redis protocol |

### 6.3 Recommended Encryption Configuration

1. Enable TLS termination at ingress/load balancer
2. Deploy Istio service mesh for automatic mTLS
3. Use Google Cloud Memorystore with in-transit encryption
4. Enable encryption at rest for all persistent storage

---

## 7. Secrets Management

### 7.1 Current Implementation

**Secrets Storage:**
- Environment variables in Kubernetes manifests
- Kubernetes Secrets for sensitive configuration

**Known Sensitive Configurations:**
- Redis connection strings
- External API keys (if configured)
- Cloud provider credentials

### 7.2 Security Concerns

1. Secrets may be visible in container environment
2. No rotation mechanism for secrets
3. No centralized secrets management

### 7.3 Recommended Improvements

1. Use Google Secret Manager or HashiCorp Vault
2. Implement automatic secret rotation
3. Use Workload Identity for cloud credentials
4. Never store secrets in source code

---

## 8. Input Validation and Injection Prevention

### 8.1 Attack Surface Analysis

| Service | Input Sources | Validation Status |
|---------|--------------|-------------------|
| frontend | HTTP requests, cookies | Partial |
| cartservice | gRPC messages | Protocol-enforced |
| productcatalogservice | gRPC product IDs | Minimal |
| paymentservice | Credit card data | Basic validation |
| shippingservice | Address data | Minimal |
| emailservice | Email addresses | Basic format check |

### 8.2 Injection Risks

1. **SQL Injection:** Low risk (no SQL databases in default config)
2. **Command Injection:** Risk in services executing shell commands
3. **Path Traversal:** Risk in services handling file paths
4. **XSS:** Risk in frontend HTML rendering
5. **SSRF:** Risk in services making outbound HTTP requests

### 8.3 Required Mitigations

1. Validate and sanitize all user inputs
2. Use parameterized queries for any database operations
3. Implement output encoding for HTML responses
4. Validate URLs before making outbound requests
5. Restrict file system access to designated directories

---

## 9. Logging and Monitoring

### 9.1 Current Logging

- Standard output logging from all services
- Structured JSON logging in most services
- No centralized log aggregation (default)

### 9.2 Security Events to Monitor

1. Failed authentication attempts (N/A - no auth)
2. Unusual traffic patterns
3. Payment failures
4. Service errors and exceptions
5. Outbound connection attempts

### 9.3 Recommended Configuration

1. Enable Google Cloud Operations (Logging, Monitoring, Trace)
2. Configure audit logging for sensitive operations
3. Set up alerts for security-relevant events
4. Retain logs for compliance requirements

---

## 10. Compliance Considerations

### 10.1 PCI-DSS (Payment Card Industry)

**Current Status:** Not compliant (demo application)

**Gaps:**
- Credit card handling in mock payment service
- No encryption of cardholder data
- No access controls
- No audit logging

**Note:** This is a demonstration application with mock payment processing. Production deployments handling real payment data must implement full PCI-DSS controls.

### 10.2 GDPR (General Data Protection Regulation)

**Data Subject Rights:**
- No mechanism for data access requests
- No data deletion capability
- No consent management

**Required for Production:**
- Implement data subject request handling
- Add consent tracking
- Enable data retention policies

---

## 11. Threat Model Summary

### 11.1 STRIDE Analysis

| Threat | Risk Level | Mitigations |
|--------|------------|-------------|
| **S**poofing | High | Implement service identity (mTLS) |
| **T**ampering | Medium | Enable message signing |
| **R**epudiation | Medium | Implement audit logging |
| **I**nformation Disclosure | High | Encrypt data in transit/rest |
| **D**enial of Service | Medium | Implement rate limiting |
| **E**levation of Privilege | High | Implement RBAC, least privilege |

### 11.2 Top Security Risks

1. **No service-to-service authentication** - Any compromised service can access all others
2. **Unencrypted internal traffic** - Data visible to network attackers
3. **No input validation** - Vulnerable to injection attacks
4. **Hardcoded/exposed secrets** - Credential theft risk
5. **No network segmentation** - Lateral movement possible

---

## 12. Security Requirements Checklist

| Requirement | Status | Priority |
|-------------|--------|----------|
| TLS for external traffic | Recommended | High |
| mTLS for internal traffic | Not implemented | High |
| Network policies | Optional component | Medium |
| Secrets management | Basic (K8s secrets) | High |
| Input validation | Partial | High |
| Audit logging | Not implemented | Medium |
| Rate limiting | Not implemented | Medium |
| Container security scanning | CI/CD recommended | High |
| Vulnerability management | Manual | Medium |

---

## 13. Appendix

### 13.1 Protocol Buffers Definition Location
`/protos/demo.proto`

### 13.2 Related Documentation
- [Development Guide](/docs/development-guide.md)
- [Kubernetes Manifests](/kubernetes-manifests/)
- [Kustomize Components](/kustomize/components/)
- [Network Policies Component](/kustomize/components/network-policies/)

### 13.3 Security Contact
For security vulnerabilities, please see [SECURITY.md](/.github/SECURITY.md)
