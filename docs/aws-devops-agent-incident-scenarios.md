# AWS DevOps Agent Incident Scenarios for Online Boutique

This guide provides detailed incident response scenarios specific to the Online Boutique microservices application, demonstrating how AWS DevOps Agent assists in investigation, root cause analysis, and mitigation.

---

## Introduction: Using AWS DevOps Agent for Incident Response

When an incident occurs, AWS DevOps Agent:

1. **Detects the incident** via observability platform alerts (CloudWatch, Datadog, etc.)
2. **Correlates data** across metrics, logs, traces, and recent deployments
3. **Analyzes root causes** using application topology and historical patterns
4. **Recommends mitigations** with specific commands and actions
5. **Coordinates response** via Slack channels and ticketing systems
6. **Documents the incident** for post-incident review

**Typical Investigation Flow:**
```
Alert Triggered → Agent Notifies Slack → Investigation Starts
    ↓
Metrics Analysis → Log Examination → Trace Analysis
    ↓
Deployment Correlation → Dependency Analysis → Root Cause Identified
    ↓
Mitigation Recommended → Responder Executes → Incident Resolved
    ↓
Prevention Analysis → Long-term Fixes Recommended
```

---

## Investigation Starting Points

The agent can start investigations from multiple entry points:

| Starting Point | When to Use | Example |
|----------------|-------------|---------|
| **Latest alarm** | CloudWatch/Datadog alarm triggered | "Investigate the latest critical alarm" |
| **High CPU** | Resource utilization spike | "Why is frontend CPU at 95%?" |
| **Error rate spike** | Sudden increase in errors | "Why are we seeing 500 errors in checkout?" |
| **Specific service** | Known problematic service | "Investigate cartservice issues" |
| **Time range** | Incident during specific period | "What happened between 14:00-14:30 UTC?" |

---

## Scenario 1: Frontend Service High CPU Usage

### Symptoms

- Frontend pods showing 90%+ CPU utilization
- Response time increased from 200ms to 3000ms
- CloudWatch alarm: `Frontend-HighCPU` triggered
- Users reporting slow page loads

### AWS DevOps Agent Investigation

**Agent automatically:**
1. Detects CloudWatch alarm for frontend service
2. Posts to Slack: `#incidents-online-boutique`
   ```
   🚨 INCIDENT DETECTED - High CPU
   Service: frontend
   CPU: 95% (threshold: 80%)
   Duration: 5 minutes
   [View Investigation] [Acknowledge]
   ```

3. Creates ServiceNow incident: `INC0012345`

**Investigation workflow:**

```
Agent: "I've detected high CPU usage in the frontend service. Let me investigate."

Agent: Checking metrics...
- frontend CPU: 95% (up from 15% baseline)
- frontend memory: 450MB / 512MB (87%)
- Request rate: 250 req/s (normal: 50 req/s)
- Error rate: 0.5% (within normal range)

Agent: Examining recent deployments...
- No deployments in the last 6 hours
- Last deployment: frontend@abc123f (6h ago, successful)

Agent: Checking load generator...
- loadgenerator replicas: 5 (up from 1)
- Unusual spike in user sessions

Agent: Analyzing logs...
- Frequent template rendering operations
- High allocation rate in session management
- Connection pool showing saturation warnings
```

### Common Root Causes

| Root Cause | Indicators | Likelihood |
|------------|------------|------------|
| **Load generator spike** | loadgenerator replicas increased | High |
| **Template rendering inefficiency** | High CPU during page render | Medium |
| **Connection pool exhaustion** | gRPC connection warnings | Medium |
| **Memory pressure causing GC** | High GC activity in Go runtime | Low |

### Immediate Mitigation

**Agent recommends:**

```bash
# Option 1: Scale frontend horizontally
kubectl scale deployment/frontend --replicas=5

# Option 2: Reduce load generator
kubectl scale deployment/loadgenerator --replicas=1

# Option 3: Increase CPU limits (temporary)
kubectl set resources deployment/frontend --limits=cpu=500m,memory=512Mi
```

**Responder actions:**
1. Acknowledge incident in Slack: `/devops-agent ack INC0012345`
2. Execute mitigation: Scale frontend to 5 replicas
3. Monitor CPU metrics: Should drop below 50% within 2 minutes
4. Update ServiceNow: Mark as "In Progress" → "Resolved"

### Long-term Prevention

**Agent prevention tab recommendations:**

1. **Implement horizontal pod autoscaling:**
   ```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: frontend-hpa
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: frontend
     minReplicas: 2
     maxReplicas: 10
     metrics:
     - type: Resource
       resource:
         name: cpu
         target:
           type: Utilization
           averageUtilization: 70
   ```

2. **Optimize template rendering:** Cache compiled templates
3. **Implement connection pooling:** Configure gRPC keepalive settings
4. **Add load shedding:** Reject requests when overloaded

---

## Scenario 2: CartService Redis Connection Errors

### Symptoms

- Cart operations failing with "connection refused"
- Error logs: `dial tcp 10.0.0.15:6379: connect: connection refused`
- CloudWatch alarm: `CartService-ErrorRate` (20% error rate)
- Users cannot add items to cart

### AWS DevOps Agent Investigation

**Agent automatically:**
1. Detects error rate spike in cartservice
2. Posts to Slack with severity HIGH
3. Creates PagerDuty incident, pages on-call engineer

**Investigation workflow:**

```
Agent: "Detected high error rate in cartservice. Investigating Redis connectivity."

Agent: Checking cartservice logs...
- Error: "rpc error: code = Unavailable desc = connection error"
- Redis connection pool: 0/10 active connections
- Connection timeout after 5s

Agent: Checking Redis pod status...
- redis-cart pod: Running, 1/1 ready
- Redis service: Exists, ClusterIP 10.0.0.15
- Redis health check: FAILED (connection refused on port 6379)

Agent: Checking Redis logs...
- Redis pod restarted 3 minutes ago (OOMKilled)
- Now in CrashLoopBackOff state
- Error: "Failed loading the RDB file, can't recover"

Agent: Checking network policies...
- NetworkPolicy allows cartservice → redis-cart on port 6379
- No recent network policy changes
```

### Common Root Causes

| Root Cause | Indicators | Likelihood |
|------------|------------|------------|
| **Redis pod crashed/restarting** | OOMKilled, CrashLoopBackOff | High |
| **Network policy blocking traffic** | Connection refused, policy changes | Medium |
| **Redis disk full** | RDB save failure | Medium |
| **Connection pool exhausted** | Max connections reached | Low |

### Immediate Mitigation

**Agent recommends:**

```bash
# Check Redis status
kubectl get pod -l app=redis-cart
kubectl describe pod redis-cart-xxxxx

# Option 1: Restart Redis pod (if corrupted state)
kubectl delete pod -l app=redis-cart

# Option 2: Increase Redis memory limit
kubectl set resources deployment/redis-cart --limits=memory=512Mi

# Option 3: Clear Redis data (dev/test only!)
kubectl exec -it redis-cart-xxxxx -- redis-cli FLUSHALL
```

**Responder actions:**
1. Verify Redis pod status: `kubectl logs redis-cart-xxxxx`
2. If OOMKilled, increase memory limits
3. Restart Redis pod to clear corrupted state
4. Monitor cartservice error rate: Should drop to 0% within 1 minute
5. Update PagerDuty incident with root cause

### Long-term Prevention

**Agent prevention tab recommendations:**

1. **Implement Redis High Availability:**
   - Use Redis Sentinel for automatic failover
   - Consider managed Redis (AWS ElastiCache, GCP Memorystore)

2. **Add connection retry logic in cartservice:**
   ```go
   // cartservice/main.go
   func connectToRedis(ctx context.Context) (*redis.Client, error) {
       client := redis.NewClient(&redis.Options{
           Addr:         redisAddr,
           MaxRetries:   3,
           DialTimeout:  10 * time.Second,
           ReadTimeout:  30 * time.Second,
           WriteTimeout: 30 * time.Second,
           PoolSize:     10,
       })
       return client, nil
   }
   ```

3. **Configure resource requests and limits:**
   ```yaml
   resources:
     requests:
       memory: "256Mi"
       cpu: "100m"
     limits:
       memory: "512Mi"
       cpu: "200m"
   ```

4. **Add Redis monitoring:** Memory usage, connection count, command latency

---

## Scenario 3: Payment Service Deployment Regression

### Symptoms

- Payment failures started 10 minutes ago
- Error rate: 0% → 45% immediately after deployment
- User complaints: "Payment not processing"
- CloudWatch alarm: `PaymentService-ErrorRate` triggered

### AWS DevOps Agent Investigation

**Agent automatically:**
1. Detects error spike correlated with deployment
2. Posts to Slack with GitHub commit link
3. Creates critical ServiceNow incident

**Investigation workflow:**

```
Agent: "High error rate detected in paymentservice. Correlating with recent deployment."

Agent: Checking recent deployments...
- Deployment: paymentservice@def456a (8 minutes ago)
- Commit: "Update payment processor API version"
- Author: developer@example.com
- GitHub PR: #142

Agent: Analyzing error logs...
- Error: "ValidationError: invalid card_cvv format"
- New code path introduced in payment validation
- Expected format changed from string to int

Agent: Comparing with previous version...
- Previous: paymentservice@abc123f (no errors, 2 days stable)
- Change: Updated stripe-api from v3.2 to v4.0
- Breaking change: CVV validation format

Agent: Checking GitHub PR #142...
- PR description mentions API upgrade
- Tests passed but CVV format change not caught
- No staging deployment before production
```

### Common Root Causes

| Root Cause | Indicators | Likelihood |
|------------|------------|------------|
| **Introduced bug in new code** | Errors start immediately after deploy | High |
| **Breaking dependency change** | Library version upgrade | High |
| **Configuration mismatch** | Environment variable change | Medium |
| **Database migration issue** | Schema incompatibility | Low |

### Immediate Mitigation

**Agent recommends:**

```bash
# Option 1: Rollback to previous version (RECOMMENDED)
kubectl rollout undo deployment/paymentservice

# Option 2: Scale down new version, scale up old version
kubectl set image deployment/paymentservice server=gcr.io/google-samples/microservices-demo/paymentservice:abc123f

# Monitor rollback status
kubectl rollout status deployment/paymentservice
```

**Responder actions:**
1. Acknowledge incident and confirm rollback decision
2. Execute rollback: `kubectl rollout undo deployment/paymentservice`
3. Verify error rate drops to 0% within 1 minute
4. Comment on GitHub PR #142 with incident details
5. Update ServiceNow: Root cause + rollback completed
6. Schedule fix deployment with proper testing

### Long-term Prevention

**Agent prevention tab recommendations:**

1. **Implement canary deployments:**
   ```yaml
   # skaffold.yaml
   deploy:
     kubectl:
       manifests:
         - kubernetes-manifests/*.yaml
     helm:
       releases:
       - name: paymentservice
         chartPath: helm/paymentservice
         setValues:
           canary:
             enabled: true
             steps:
             - setWeight: 10
             - pause: {duration: 5m}
             - setWeight: 50
             - pause: {duration: 10m}
             - setWeight: 100
   ```

2. **Add integration tests for payment flows:**
   ```javascript
   // tests/integration/payment.test.js
   describe('Payment Processing', () => {
     it('should accept valid credit card with CVV', async () => {
       const result = await paymentClient.charge({
         amount: 1000,
         card_number: '4242424242424242',
         card_cvv: '123',  // Test both string and int formats
         expiry: '12/25'
       });
       expect(result.status).toBe('succeeded');
     });
   });
   ```

3. **Require staging deployment before production**
4. **Add deployment gates:** Manual approval for payment service changes
5. **Implement feature flags:** Gradual rollout of payment processor changes

---

## Scenario 4: Currency Service High Latency

### Symptoms

- Checkout process taking 15+ seconds (normal: 2 seconds)
- Users timing out during payment
- CloudWatch alarm: `CurrencyService-HighLatency` (p99: 12s)
- External API dependency failing

### AWS DevOps Agent Investigation

**Agent automatically:**
1. Detects latency spike in currencyservice
2. Correlates with downstream services (checkoutservice also slow)
3. Posts to Slack with dependency chain visualization

**Investigation workflow:**

```
Agent: "High latency detected in currencyservice. Investigating external dependencies."

Agent: Checking service metrics...
- currencyservice latency: p50=8s, p99=15s (baseline: p50=50ms, p99=200ms)
- Request rate: Normal (150 req/s)
- Error rate: 5% (timeout errors)

Agent: Checking external API calls...
- External call to ECB API: https://www.ecb.europa.eu/stats/eurofxref/eurofxref-daily.xml
- API response time: 12s (normal: 100ms)
- HTTP status: 200 (no errors, just slow)

Agent: Checking traces (X-Ray)...
- 90% of time spent in "fetchExchangeRates" function
- External API call blocking event loop
- No caching in place, every request hits external API

Agent: Checking recent changes...
- No deployments in last 24 hours
- External ECB API status: Degraded (check status page)
```

### Common Root Causes

| Root Cause | Indicators | Likelihood |
|------------|------------|------------|
| **External API degraded** | High latency to ECB API | High |
| **Network issues** | Timeouts, packet loss | Medium |
| **No caching strategy** | Every request hits external API | High |
| **Synchronous blocking calls** | Node.js event loop blocked | Medium |

### Immediate Mitigation

**Agent recommends:**

```bash
# Option 1: Increase timeout and add fallback rates
kubectl set env deployment/currencyservice \
  EXTERNAL_API_TIMEOUT=30s \
  FALLBACK_RATES_ENABLED=true

# Option 2: Scale currencyservice to handle load
kubectl scale deployment/currencyservice --replicas=5

# Option 3: Restart service to clear any stuck connections
kubectl rollout restart deployment/currencyservice
```

**Responder actions:**
1. Check ECB API status page: https://www.ecb.europa.eu/status
2. Enable fallback exchange rates if available
3. Increase timeout temporarily: 30 seconds
4. Monitor checkoutservice latency: Should improve immediately
5. Plan caching implementation for permanent fix

### Long-term Prevention

**Agent prevention tab recommendations:**

1. **Implement caching with TTL:**
   ```javascript
   // currencyservice/server.js
   const NodeCache = require('node-cache');
   const cache = new NodeCache({ stdTTL: 3600 }); // 1 hour TTL

   async function getExchangeRates() {
     const cached = cache.get('exchange_rates');
     if (cached) {
       return cached;
     }

     try {
       const rates = await fetchFromECB({ timeout: 5000 });
       cache.set('exchange_rates', rates);
       return rates;
     } catch (error) {
       // Return last known good rates
       return cache.get('exchange_rates_fallback');
     }
   }
   ```

2. **Implement circuit breaker:**
   ```javascript
   const CircuitBreaker = require('opossum');

   const breaker = new CircuitBreaker(fetchFromECB, {
     timeout: 5000,
     errorThresholdPercentage: 50,
     resetTimeout: 30000
   });

   breaker.fallback(() => getFallbackRates());
   ```

3. **Add health checks:** Probe external API availability
4. **Monitor external dependencies:** Track ECB API response times
5. **Consider backup currency data sources**

---

## Scenario 5: Recommendation Service Memory Leak

### Symptoms

- Recommendationservice pods restarting frequently (OOMKilled)
- Memory usage growing steadily from 100MB → 512MB over 2 hours
- Restarts every 30 minutes
- Recommendation feature intermittently unavailable

### AWS DevOps Agent Investigation

**Agent automatically:**
1. Detects repeated OOMKilled events
2. Identifies memory growth pattern
3. Posts to Slack with memory usage graph

**Investigation workflow:**

```
Agent: "Recommendation service showing memory leak. Analyzing heap growth."

Agent: Checking pod events...
- Last 6 hours: 12 OOMKilled events
- Memory limit: 512Mi
- Pattern: Gradual growth to 512Mi, then killed

Agent: Analyzing memory metrics...
- Memory at pod start: 80MB
- Memory after 30 minutes: 400MB
- Memory growth rate: ~10MB/minute
- No corresponding increase in request volume

Agent: Checking recent deployments...
- Deployment 3 days ago: "Add product feature embeddings for ML model"
- New feature: Load ML model for better recommendations
- Model size: 200MB (not released after inference)

Agent: Examining Python memory profile...
- High retention of product catalog objects
- ML model loaded per request (not singleton)
- List comprehensions creating large temporary objects
```

### Common Root Causes

| Root Cause | Indicators | Likelihood |
|------------|------------|------------|
| **Python object retention** | Gradual memory growth | High |
| **ML model not released** | Large memory blocks retained | High |
| **Memory leak in dependency** | Steady growth rate | Medium |
| **Inefficient algorithms** | Large temporary objects | Medium |

### Immediate Mitigation

**Agent recommends:**

```bash
# Option 1: Increase memory limit (temporary)
kubectl set resources deployment/recommendationservice --limits=memory=1Gi

# Option 2: Restart service to clear memory
kubectl rollout restart deployment/recommendationservice

# Option 3: Reduce request load temporarily
# (if memory leak is request-volume dependent)
```

**Responder actions:**
1. Increase memory limits to 1Gi to stop restart loop
2. Collect heap dump for analysis: `kubectl exec recommendationservice-xxx -- python -m pdb`
3. Create GitHub issue with memory profile
4. Schedule deployment with fix
5. Monitor memory growth rate post-mitigation

### Long-term Prevention

**Agent prevention tab recommendations:**

1. **Fix ML model singleton pattern:**
   ```python
   # recommendationservice/recommendation_server.py
   class RecommendationService:
       _model = None  # Class-level singleton

       @classmethod
       def get_model(cls):
           if cls._model is None:
               cls._model = load_ml_model()
           return cls._model

       def get_recommendations(self, request):
           model = self.get_model()  # Reuse singleton
           # ... inference logic
   ```

2. **Implement proper object lifecycle management:**
   ```python
   def list_recommendations(self, request, context):
       products = get_product_catalog()
       try:
           # Use generator instead of list comprehension
           recommendations = (
               self._score_product(p) for p in products
           )
           return list(itertools.islice(recommendations, 5))
       finally:
           products.clear()  # Explicit cleanup
   ```

3. **Add memory monitoring and alerts:**
   ```yaml
   # Alert if memory grows >50MB in 10 minutes
   - alert: MemoryLeakDetected
     expr: rate(container_memory_usage_bytes[10m]) > 50000000
   ```

4. **Implement memory profiling in CI/CD:** Detect leaks before production
5. **Add memory limit with appropriate buffer:** 512Mi → 768Mi with monitoring

---

## Scenario 6: Complete Service Outage

### Symptoms

- All services returning 503 errors
- Frontend unreachable
- CloudWatch alarms: All services critical
- Multiple PagerDuty alerts

### AWS DevOps Agent Investigation

**Agent automatically:**
1. Detects system-wide outage
2. Creates P1 incident in ServiceNow
3. Pages entire on-call rotation
4. Starts infrastructure-level investigation

**Investigation workflow:**

```
Agent: "System-wide outage detected. Investigating infrastructure."

Agent: Checking cluster health...
- EKS cluster: Control plane healthy
- Node status: 0/3 nodes ready (all NotReady)
- Pod status: Most pods in Pending state

Agent: Checking node events...
- Event: "Node node-1: kubelet stopped posting status"
- Event: "Node node-2: NetworkPlugin not ready"
- Event: "Node node-3: disk pressure"

Agent: Checking recent infrastructure changes...
- 15 minutes ago: AWS maintenance event in us-east-1a
- VPC network ACL modified 10 minutes ago
- New network policy applied via kubectl

Agent: Analyzing network policy...
- NetworkPolicy "deny-all-ingress" applied to default namespace
- Blocks all ingress traffic (including kubelet health checks)
- Deployed via automation script
```

### Common Root Causes

| Root Cause | Indicators | Likelihood |
|------------|------------|------------|
| **Network policy blocking critical traffic** | All pods affected simultaneously | High |
| **Infrastructure issue** | AWS maintenance, node failures | High |
| **Resource exhaustion** | Cluster out of capacity | Medium |
| **Recent Kubernetes configuration change** | Applied just before outage | High |

### Immediate Mitigation

**Agent recommends (CRITICAL):**

```bash
# Check cluster nodes
kubectl get nodes

# Check recent network policies
kubectl get networkpolicies --all-namespaces

# Option 1: Remove problematic network policy
kubectl delete networkpolicy deny-all-ingress -n default

# Option 2: Restart kubelet on nodes (if SSH access)
ssh node-1
sudo systemctl restart kubelet

# Option 3: Drain and replace unhealthy nodes
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
# AWS auto-scaling will replace node
```

**Responder actions:**
1. Immediately check recent changes: `kubectl get events --sort-by='.lastTimestamp'`
2. Identify and remove blocking network policy
3. Verify pods start recovering within 2 minutes
4. Check service endpoints: All services should become healthy
5. Update P1 incident with root cause
6. Communicate resolution to stakeholders

### Long-term Prevention

**Agent prevention tab recommendations:**

1. **Implement policy validation in CI/CD:**
   ```bash
   # Pre-apply validation
   kubectl apply --dry-run=server -f network-policy.yaml
   kubectl auth can-i create networkpolicies --as=system:serviceaccount:default:deployer
   ```

2. **Use staging environment for network policy testing**
3. **Implement change management:** Require approval for network policies
4. **Add monitoring for cluster health:** Node readiness, pod scheduling failures
5. **Multi-region deployment:** Failover to secondary region
6. **Infrastructure as Code review:** Peer review for network changes

---

## Scenario 7: Load Generator Overwhelming System

### Symptoms

- All services showing high latency (5-10x normal)
- Resource utilization at 90%+ across all pods
- CloudWatch alarms: Multiple services high CPU/memory
- No recent deployments

### AWS DevOps Agent Investigation

**Agent automatically:**
1. Detects widespread performance degradation
2. Identifies load pattern anomaly
3. Posts to Slack with load graph

**Investigation workflow:**

```
Agent: "System-wide performance degradation. Checking load patterns."

Agent: Analyzing request rates...
- Frontend requests: 2000 req/s (baseline: 200 req/s)
- Load started 10 minutes ago
- User-Agent: "Locust/2.14.2" (load generator traffic)

Agent: Checking loadgenerator configuration...
- Replicas: 1 (normal)
- Environment variable changed: USERS=10000 (was: 100)
- ConfigMap updated 12 minutes ago
- Change author: automation-bot

Agent: Checking resource utilization...
- frontend: CPU 95%, memory 90%
- cartservice: CPU 85%, memory 75%
- checkoutservice: CPU 90%, memory 80%
- All services under heavy load, but functioning

Agent: No rate limiting configured in frontend
- Every request reaches backend services
- No circuit breakers in place
```

### Common Root Causes

| Root Cause | Indicators | Likelihood |
|------------|------------|------------|
| **Load generator misconfigured** | Unrealistic traffic volume | High |
| **No rate limiting** | All requests accepted | High |
| **Autoscaling not configured** | Fixed replica counts | High |
| **Missing capacity planning** | Insufficient resources | Medium |

### Immediate Mitigation

**Agent recommends:**

```bash
# Option 1: Reduce load generator traffic
kubectl set env deployment/loadgenerator USERS=100

# Option 2: Scale services to handle load
kubectl scale deployment/frontend --replicas=10
kubectl scale deployment/cartservice --replicas=5
kubectl scale deployment/checkoutservice --replicas=5

# Option 3: Temporarily disable load generator
kubectl scale deployment/loadgenerator --replicas=0
```

**Responder actions:**
1. Confirm this is load generator traffic (not real users)
2. Reduce load generator to normal levels
3. Monitor service recovery: Latency should normalize in 2-3 minutes
4. Review load generator configuration in version control
5. No customer impact (demo application)

### Long-term Prevention

**Agent prevention tab recommendations:**

1. **Implement rate limiting in frontend:**
   ```go
   // frontend/main.go
   import "golang.org/x/time/rate"

   var limiter = rate.NewLimiter(rate.Limit(100), 200) // 100 req/s, burst 200

   func rateLimitMiddleware(next http.Handler) http.Handler {
       return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
           if !limiter.Allow() {
               http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
               return
           }
           next.ServeHTTP(w, r)
       })
   }
   ```

2. **Configure horizontal pod autoscaling** (see Scenario 1)
3. **Separate load testing environment:** Don't run load tests against production
4. **Add resource quotas:**
   ```yaml
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: load-generator-quota
   spec:
     hard:
       pods: "1"
       requests.cpu: "100m"
       requests.memory: "128Mi"
   ```

5. **Implement circuit breakers:** Prevent cascade failures
6. **Add load shedding:** Reject requests when overloaded

---

## Scenario 8: gRPC Communication Failures

### Symptoms

- Services unable to communicate via gRPC
- Error logs: `rpc error: code = Unavailable desc = connection refused`
- Frontend showing "Service temporarily unavailable"
- CloudWatch alarm: `Frontend-ErrorRate` (30% errors)

### AWS DevOps Agent Investigation

**Agent automatically:**
1. Detects service mesh communication failures
2. Identifies gRPC connection errors
3. Posts to Slack with service dependency graph

**Investigation workflow:**

```
Agent: "gRPC communication failures detected. Investigating service mesh."

Agent: Checking frontend logs...
- Error calling productcatalogservice: "connection refused"
- Error calling currencyservice: "context deadline exceeded"
- Error calling recommendationservice: "transport: authentication handshake failed"

Agent: Checking service discovery...
- productcatalogservice.default.svc.cluster.local resolves to 10.0.1.20
- Service endpoint: 10.0.1.20:3550
- Endpoint status: Healthy (1 pod ready)

Agent: Checking network policies...
- NetworkPolicy allows frontend → productcatalogservice
- Port 3550 is open

Agent: Checking Istio/service mesh (if enabled)...
- mTLS policy changed to STRICT mode 20 minutes ago
- Frontend not enrolled in service mesh (no sidecar)
- Other services have Istio sidecars, enforcing mTLS
- Communication failure: frontend (no mTLS) → productcatalogservice (mTLS required)

Agent: Checking recent changes...
- Istio PeerAuthentication updated from PERMISSIVE to STRICT
- Change applied cluster-wide
```

### Common Root Causes

| Root Cause | Indicators | Likelihood |
|------------|------------|------------|
| **Service mesh mTLS mismatch** | Authentication handshake failures | High |
| **Service discovery issues** | DNS resolution failures | Medium |
| **Network policy blocking gRPC** | Connection refused errors | Medium |
| **Service mesh configuration change** | Recent Istio/Linkerd update | High |

### Immediate Mitigation

**Agent recommends:**

```bash
# Check service mesh configuration
kubectl get peerauthentication --all-namespaces

# Option 1: Revert mTLS to PERMISSIVE mode
kubectl patch peerauthentication default -n default \
  --type merge \
  -p '{"spec":{"mtls":{"mode":"PERMISSIVE"}}}'

# Option 2: Inject Istio sidecar into frontend
kubectl label namespace default istio-injection=enabled
kubectl rollout restart deployment/frontend

# Option 3: Temporarily disable service mesh for debugging
kubectl delete peerauthentication default -n default
```

**Responder actions:**
1. Verify service mesh configuration: `kubectl get peerauthentication`
2. Change mTLS mode to PERMISSIVE to allow mixed traffic
3. Plan gradual rollout: Inject sidecars into all services
4. Verify gRPC communication restored within 1 minute
5. Update runbook with service mesh enrollment procedure

### Long-term Prevention

**Agent prevention tab recommendations:**

1. **Enroll all services in service mesh:**
   ```bash
   # Label namespace for automatic sidecar injection
   kubectl label namespace default istio-injection=enabled

   # Restart all deployments to inject sidecars
   kubectl rollout restart deployment --all -n default
   ```

2. **Use PERMISSIVE mode during migration:**
   ```yaml
   apiVersion: security.istio.io/v1beta1
   kind: PeerAuthentication
   metadata:
     name: default
     namespace: default
   spec:
     mtls:
       mode: PERMISSIVE  # Allow both mTLS and plaintext during migration
   ```

3. **Gradual rollout to STRICT mode:**
   - Week 1: All services with sidecars, PERMISSIVE mode
   - Week 2: Verify all traffic is mTLS
   - Week 3: Enable STRICT mode

4. **Add gRPC health checks:**
   ```yaml
   livenessProbe:
     exec:
       command: ["/bin/grpc_health_probe", "-addr=:3550"]
   readinessProbe:
     exec:
       command: ["/bin/grpc_health_probe", "-addr=:3550"]
   ```

5. **Monitor service mesh metrics:** mTLS success rate, connection failures
6. **Test service mesh changes in staging first**

---

## Integration with Workflows

### Slack Commands

Once AWS DevOps Agent is integrated, use these commands in Slack:

```
/devops-agent investigate <service-name>
/devops-agent ack <incident-id>
/devops-agent mitigate <incident-id>
/devops-agent status
/devops-agent prevention <service-name>
```

### ServiceNow Ticket Updates

Agent automatically updates tickets with:
- Root cause analysis
- Mitigation steps executed
- Prevention recommendations
- Incident timeline

### GitHub Integration

Agent comments on recent PRs if correlation found:
```
🤖 AWS DevOps Agent

This deployment may be related to incident INC0012345:
- Error rate increased from 0% to 45% after deployment
- Root cause: Breaking change in payment validation
- Recommendation: Rollback and add integration tests

[View Full Investigation]
```

---

## Summary

These 8 scenarios demonstrate how AWS DevOps Agent accelerates incident response for Online Boutique:

| Scenario | MTTD Improvement | MTTR Improvement | Key Benefit |
|----------|------------------|------------------|-------------|
| 1. High CPU | 2min → 30sec | 15min → 5min | Automatic load pattern detection |
| 2. Redis Connection | 5min → 1min | 20min → 3min | Dependency health correlation |
| 3. Deployment Regression | 10min → 1min | 30min → 2min | GitHub commit correlation |
| 4. External API Latency | 15min → 2min | 45min → 10min | Trace analysis + caching recommendation |
| 5. Memory Leak | 60min → 10min | 2hr → 30min | Heap growth pattern detection |
| 6. Complete Outage | 5min → 1min | 60min → 10min | Infrastructure change correlation |
| 7. Load Spike | 10min → 2min | 20min → 5min | Load pattern analysis |
| 8. gRPC Failures | 20min → 3min | 45min → 10min | Service mesh configuration correlation |

**Next Steps:**
- Configure deployment tracking: [AWS DevOps Agent Deployment Integration](./aws-devops-agent-deployment-integration.md)
- Implement prevention recommendations: [AWS DevOps Agent Prevention Guide](./aws-devops-agent-prevention.md)
