# AWS DevOps Agent Prevention Guide for Online Boutique

This guide demonstrates how to use AWS DevOps Agent's prevention recommendations to proactively improve reliability and prevent future incidents in the Online Boutique application.

---

## Introduction: From Reactive to Proactive

After responding to incidents, AWS DevOps Agent analyzes patterns and recommends long-term improvements to prevent recurrence. The **Prevention Tab** in the agent web application provides:

- **Root cause analysis** across multiple incidents
- **Proactive recommendations** for reliability improvements
- **Implementation specifications** with code examples
- **Priority ranking** based on incident frequency and impact
- **Progress tracking** for implemented improvements

**Workflow:**
```
Incident Resolved → Pattern Analysis → Prevention Recommendations
    ↓
Review Weekly → Prioritize → Create Specifications
    ↓
Implement → Test → Deploy → Validate → Track Metrics
```

---

## Prevention Tab Features

### How the Agent Analyzes Incidents

After each incident, the agent:

1. **Identifies patterns** across similar incidents
2. **Analyzes root causes** to find systemic issues
3. **Correlates with code** to locate problem areas
4. **Generates specifications** with implementation details
5. **Estimates impact** based on incident frequency and severity

### Recommendation Categories

| Category | Description | Example |
|----------|-------------|---------|
| **Observability Gaps** | Missing metrics, logs, or traces | Add Redis connection metrics |
| **Infrastructure Resilience** | Single points of failure, insufficient resources | Implement Redis HA |
| **Deployment Pipeline** | Testing gaps, rollback procedures | Add integration tests |
| **Code Quality** | Error handling, timeouts, retries | Implement circuit breaker |
| **Architecture** | Service dependencies, coupling | Add async communication |

---

## Common Improvement Categories

### 1. Observability Gaps

**Symptoms:** Incidents take long to detect or investigate due to missing data.

**Common Recommendations:**
- Add custom metrics for business operations
- Implement distributed tracing
- Enhance structured logging
- Add health check endpoints
- Configure alerting for anomalies

**Example:** After multiple incidents where high Redis latency caused cart failures, the agent recommends:
```
Add Redis connection pool metrics to cartservice:
- Active connections
- Idle connections
- Connection wait time
- Connection failures
```

### 2. Infrastructure Resilience

**Symptoms:** Service outages due to single points of failure or resource constraints.

**Common Recommendations:**
- Implement high availability for stateful services
- Configure resource limits and requests
- Add horizontal pod autoscaling
- Implement multi-region deployment
- Configure persistent storage

**Example:** After Redis OOMKilled events caused cart data loss:
```
Implement Redis High Availability:
1. Deploy Redis Sentinel (3 instances)
2. Configure automatic failover
3. Use managed Redis (ElastiCache/Memorystore)
4. Add persistent volume for data durability
```

### 3. Deployment Pipeline

**Symptoms:** Regressions introduced in production due to insufficient testing or unsafe deployment practices.

**Common Recommendations:**
- Add integration and end-to-end tests
- Implement canary deployments
- Add deployment gates and approvals
- Configure automatic rollback on errors
- Implement blue-green deployments

**Example:** After payment service regression:
```
Add integration tests for payment flows:
- Test all credit card validation formats
- Test payment processor API compatibility
- Require staging deployment before production
- Add manual approval gate for critical services
```

### 4. Code Quality

**Symptoms:** Incidents caused by poor error handling, missing timeouts, or lack of retry logic.

**Common Recommendations:**
- Implement circuit breakers for external APIs
- Add retry logic with exponential backoff
- Configure appropriate timeouts
- Implement graceful degradation
- Add request validation

**Example:** After currency service timeouts from ECB API:
```
Implement circuit breaker for external API:
- Open circuit after 50% error rate
- Half-open state after 30 seconds
- Fallback to cached exchange rates
- Alert when circuit opens
```

---

## Online Boutique Specific Recommendations

### Frontend Service

**Common Issues:**
- High CPU during traffic spikes
- Template rendering inefficiency
- Connection pool exhaustion

**Prevention Recommendations:**

1. **Implement Horizontal Pod Autoscaling:**
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
     - type: Pods
       pods:
         metric:
           name: http_requests_per_second
         target:
           type: AverageValue
           averageValue: "1000"
   ```

2. **Cache Compiled Templates:**
   ```go
   // frontend/main.go
   var templates = template.Must(template.ParseGlob("templates/*.html"))

   func renderTemplate(w http.ResponseWriter, name string, data interface{}) {
       err := templates.ExecuteTemplate(w, name, data)
       if err != nil {
           log.Printf("template rendering error: %v", err)
           http.Error(w, "Internal Server Error", 500)
       }
   }
   ```

3. **Implement gRPC Connection Pooling:**
   ```go
   // Reuse gRPC connections instead of creating per-request
   var grpcConnections = make(map[string]*grpc.ClientConn)
   var connMutex sync.RWMutex

   func getGRPCConnection(address string) (*grpc.ClientConn, error) {
       connMutex.RLock()
       conn, exists := grpcConnections[address]
       connMutex.RUnlock()

       if exists {
           return conn, nil
       }

       connMutex.Lock()
       defer connMutex.Unlock()

       conn, err := grpc.Dial(address,
           grpc.WithInsecure(),
           grpc.WithKeepaliveParams(keepalive.ClientParameters{
               Time:    30 * time.Second,
               Timeout: 10 * time.Second,
           }),
       )
       if err != nil {
           return nil, err
       }

       grpcConnections[address] = conn
       return conn, nil
   }
   ```

### CartService

**Common Issues:**
- Redis connection failures
- Data loss on Redis restarts
- Connection pool exhaustion

**Prevention Recommendations:**

1. **Implement Redis High Availability:**
   ```yaml
   # Use managed Redis or deploy Sentinel
   # Example: AWS ElastiCache with Multi-AZ
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: cartservice-config
   data:
     REDIS_ADDR: "master.online-boutique.abc123.use1.cache.amazonaws.com:6379"
     REDIS_READ_ENDPOINT: "replica.online-boutique.abc123.use1.cache.amazonaws.com:6379"
     REDIS_SENTINEL_ENABLED: "true"
   ```

2. **Add Connection Retry Logic:**
   ```csharp
   // cartservice/cartstore/RedisCartStore.cs
   private async Task<T> ExecuteWithRetry<T>(Func<Task<T>> operation, int maxRetries = 3)
   {
       int attempt = 0;
       while (true)
       {
           try
           {
               return await operation();
           }
           catch (RedisConnectionException ex)
           {
               attempt++;
               if (attempt >= maxRetries)
               {
                   _logger.LogError(ex, "Redis operation failed after {Attempts} attempts", attempt);
                   throw;
               }

               var delay = TimeSpan.FromMilliseconds(Math.Pow(2, attempt) * 100);
               _logger.LogWarning("Redis connection failed, retrying in {Delay}ms (attempt {Attempt}/{MaxRetries})",
                   delay.TotalMilliseconds, attempt, maxRetries);
               await Task.Delay(delay);
           }
       }
   }
   ```

3. **Configure Resource Limits:**
   ```yaml
   resources:
     requests:
       memory: "128Mi"
       cpu: "100m"
     limits:
       memory: "256Mi"
       cpu: "200m"
   ```

### CurrencyService

**Common Issues:**
- High latency from external ECB API
- Service degradation when API is slow
- No caching strategy

**Prevention Recommendations:**

1. **Implement Circuit Breaker with Caching:**
   ```javascript
   // currencyservice/server.js
   const CircuitBreaker = require('opossum');
   const NodeCache = require('node-cache');

   const cache = new NodeCache({ stdTTL: 3600 }); // 1 hour cache

   async function fetchExchangeRates() {
       const response = await fetch(ECB_API_URL, { timeout: 5000 });
       const data = await response.text();
       return parseXMLRates(data);
   }

   const breaker = new CircuitBreaker(fetchExchangeRates, {
       timeout: 5000,
       errorThresholdPercentage: 50,
       resetTimeout: 30000
   });

   breaker.fallback(() => {
       const cached = cache.get('exchange_rates_fallback');
       if (cached) return cached;
       throw new Error('No fallback data available');
   });

   breaker.on('open', () => {
       console.error('Circuit breaker opened - ECB API unavailable');
   });

   async function getExchangeRates() {
       const cached = cache.get('exchange_rates');
       if (cached) return cached;

       const rates = await breaker.fire();
       cache.set('exchange_rates', rates);
       cache.set('exchange_rates_fallback', rates); // Persistent fallback
       return rates;
   }
   ```

2. **Add Health Check with External Dependency:**
   ```javascript
   app.get('/health', async (req, res) => {
       const health = {
           status: 'healthy',
           timestamp: new Date().toISOString(),
           dependencies: {
               ecb_api: 'unknown'
           }
       };

       try {
           const start = Date.now();
           await fetch(ECB_API_URL, { timeout: 2000, method: 'HEAD' });
           health.dependencies.ecb_api = {
               status: 'healthy',
               latency_ms: Date.now() - start
           };
       } catch (err) {
           health.dependencies.ecb_api = {
               status: 'unhealthy',
               error: err.message
           };
       }

       const statusCode = health.dependencies.ecb_api.status === 'healthy' ? 200 : 503;
       res.status(statusCode).json(health);
   });
   ```

### CheckoutService

**Common Issues:**
- Long transaction timeouts
- No compensation for partial failures
- Downstream service failures cause checkout failures

**Prevention Recommendations:**

1. **Implement Saga Pattern with Compensation:**
   ```go
   // checkoutservice/main.go
   type CheckoutSaga struct {
       completedSteps []string
   }

   func (s *CheckoutSaga) Execute(ctx context.Context, order *pb.Order) error {
       // Step 1: Charge payment
       if err := s.chargePayment(ctx, order); err != nil {
           return s.compensate(ctx, err)
       }
       s.completedSteps = append(s.completedSteps, "payment")

       // Step 2: Reserve inventory
       if err := s.reserveInventory(ctx, order); err != nil {
           return s.compensate(ctx, err)
       }
       s.completedSteps = append(s.completedSteps, "inventory")

       // Step 3: Ship order
       if err := s.shipOrder(ctx, order); err != nil {
           return s.compensate(ctx, err)
       }
       s.completedSteps = append(s.completedSteps, "shipping")

       // Step 4: Send confirmation
       s.sendConfirmation(ctx, order) // Best effort, don't fail checkout
       return nil
   }

   func (s *CheckoutSaga) compensate(ctx context.Context, originalErr error) error {
       log.Printf("Checkout failed, compensating: %v", originalErr)

       for i := len(s.completedSteps) - 1; i >= 0; i-- {
           step := s.completedSteps[i]
           switch step {
           case "payment":
               s.refundPayment(ctx)
           case "inventory":
               s.releaseInventory(ctx)
           case "shipping":
               s.cancelShipment(ctx)
           }
       }
       return originalErr
   }
   ```

2. **Configure Service Timeouts:**
   ```go
   // Set appropriate timeout per service
   func getServiceTimeout(serviceName string) time.Duration {
       timeouts := map[string]time.Duration{
           "paymentservice":  5 * time.Second,
           "shippingservice": 3 * time.Second,
           "emailservice":    2 * time.Second,
           "currencyservice": 2 * time.Second,
       }
       return timeouts[serviceName]
   }

   func callServiceWithTimeout(ctx context.Context, serviceName string, call func(context.Context) error) error {
       timeout := getServiceTimeout(serviceName)
       ctx, cancel := context.WithTimeout(ctx, timeout)
       defer cancel()

       return call(ctx)
   }
   ```

### RecommendationService

**Common Issues:**
- Memory leaks from ML model
- OOMKilled events
- High memory growth rate

**Prevention Recommendations:**

1. **Implement Model Singleton and Memory Management:**
   ```python
   # recommendationservice/recommendation_server.py
   import gc
   import os

   class RecommendationService:
       _model = None
       _model_lock = threading.Lock()

       @classmethod
       def get_model(cls):
           if cls._model is None:
               with cls._model_lock:
                   if cls._model is None:  # Double-check locking
                       cls._model = cls._load_model()
           return cls._model

       @classmethod
       def _load_model(cls):
           logger.info("Loading ML model (singleton)")
           model = load_pretrained_model()
           return model

       def ListRecommendations(self, request, context):
           try:
               model = self.get_model()
               recommendations = self._get_recommendations(model, request.product_ids)
               return pb.ListRecommendationsResponse(product_ids=recommendations)
           finally:
               # Explicit garbage collection after large operations
               if len(request.product_ids) > 100:
                   gc.collect()

       def _get_recommendations(self, model, product_ids):
           # Use generator to avoid large temporary lists
           for product_id in product_ids:
               yield self._score_product(model, product_id)
   ```

2. **Add Memory Monitoring:**
   ```python
   import psutil
   import os

   def log_memory_usage():
       process = psutil.Process(os.getpid())
       mem_info = process.memory_info()
       logger.info("Memory usage",
           rss_mb=mem_info.rss / 1024 / 1024,
           vms_mb=mem_info.vms / 1024 / 1024,
           percent=process.memory_percent()
       )

   # Log memory usage periodically
   @app.before_request
   def before_request():
       if random.random() < 0.01:  # 1% sampling
           log_memory_usage()
   ```

---

## Implementation Workflow

### 1. Review Weekly Recommendations

Every week, review the Prevention Tab:
- Sort by **Impact Score** (frequency × severity)
- Filter by **Service** or **Category**
- Review agent-generated specifications

### 2. Prioritize Improvements

Use this prioritization matrix:

| Impact | Complexity | Priority | Action |
|--------|-----------|----------|--------|
| High | Low | P0 | Implement this week |
| High | High | P1 | Plan for next sprint |
| Medium | Low | P2 | Implement when capacity allows |
| Medium | High | P3 | Evaluate ROI |
| Low | Any | P4 | Backlog |

### 3. Create Implementation Specifications

For each prioritized improvement:
1. Review agent-generated specification
2. Estimate effort (hours/days)
3. Identify dependencies
4. Create GitHub issue with:
   - Problem statement
   - Proposed solution
   - Implementation plan
   - Testing approach
   - Rollout plan

### 4. Implement and Test

Follow standard development workflow:
```bash
# Create feature branch
git checkout -b feat/redis-ha-cartservice

# Implement changes
# ...

# Run tests locally
make test

# Create PR
git push origin feat/redis-ha-cartservice
gh pr create --title "Add Redis HA for cartservice" \
  --body "Implements prevention recommendation #42"
```

### 5. Validate with Monitoring

After deployment:
- Monitor metrics for 1 week
- Verify incident frequency reduction
- Track MTTD and MTTR improvements
- Update prevention recommendation status

### 6. Track Metrics

**Key Reliability Metrics:**

| Metric | Target | Tracking |
|--------|--------|----------|
| Mean Time To Detect (MTTD) | < 2 minutes | CloudWatch alarms + agent detection |
| Mean Time To Resolve (MTTR) | < 15 minutes | Incident duration tracking |
| Incident Frequency | -20% quarter-over-quarter | Incident count per service |
| Deployment Success Rate | > 99% | Failed deployments / total deployments |
| Change Failure Rate | < 5% | Incidents caused by deployments / total deployments |

---

## Proactive Investigation Patterns

Use AWS DevOps Agent proactively (not just reactively):

### Weekly Capacity Review

```
Ask agent: "Analyze resource utilization trends for the past week.
Are any services approaching capacity limits?"
```

Agent provides:
- Services with growing CPU/memory trends
- Projected timeline to capacity exhaustion
- Recommended scaling adjustments

### Deployment Success Analysis

```
Ask agent: "Analyze deployment success rate for the past month.
Which services have high change failure rates?"
```

Agent identifies:
- Services with frequent rollbacks
- Common deployment failure patterns
- Testing gaps or CI/CD improvements needed

### Error Pattern Identification

```
Ask agent: "Identify recurring error patterns across all services
in the past 2 weeks."
```

Agent discovers:
- Frequent but non-critical errors (logging noise)
- Error patterns that may become incidents
- Services with degrading error rates

---

## Long-term Reliability Roadmap

Use prevention recommendations to build a reliability roadmap:

**Quarter 1: Foundation**
- Implement HPA for all stateless services
- Add comprehensive health checks
- Deploy observability stack (metrics, logs, traces)

**Quarter 2: Resilience**
- Implement Redis HA for cartservice
- Add circuit breakers for external APIs
- Deploy canary deployment pipeline

**Quarter 3: Quality**
- Expand integration test coverage to 80%
- Implement chaos engineering experiments
- Add SLO monitoring and alerting

**Quarter 4: Automation**
- Implement automated remediation for common incidents
- Deploy multi-region failover
- Build self-healing capabilities

---

## Example: Implementing Circuit Breaker in CurrencyService

### Step 1: Detection

Agent identifies pattern:
- 12 incidents in past month related to ECB API timeouts
- Average MTTR: 25 minutes
- Impact: Checkout delays, user frustration

### Step 2: Specification

Agent generates implementation spec:
```markdown
## Circuit Breaker for ECB API

**Problem:** ECB API intermittently slow/unavailable, causing cascading failures

**Solution:** Implement circuit breaker with caching fallback

**Implementation:**
1. Add opossum library for circuit breaker
2. Implement 1-hour cache for exchange rates
3. Configure circuit: 50% error threshold, 30s reset timeout
4. Add fallback to cached data
5. Add monitoring for circuit state

**Testing:**
- Simulate ECB API failure
- Verify circuit opens after threshold
- Verify fallback to cached rates
- Verify circuit closes after recovery
```

### Step 3: Implementation

See [CurrencyService recommendations](#currencyservice) above for code.

### Step 4: Validation

After 1 month:
- ECB API incidents: 12 → 0
- Average latency: 150ms → 50ms (cached)
- MTTR for ECB outages: 25min → 0 (automatic fallback)

---

## Next Steps

1. **Set up AWS DevOps Agent:** See [AWS DevOps Agent Setup Guide](./aws-devops-agent-guide.md)
2. **Learn incident response patterns:** See [AWS DevOps Agent Incident Scenarios](./aws-devops-agent-incident-scenarios.md)
3. **Configure deployment tracking:** See [AWS DevOps Agent Deployment Integration](./aws-devops-agent-deployment-integration.md)

---

## Useful Links

- [AWS DevOps Agent Documentation](https://docs.aws.amazon.com/devops-agent/)
- [Site Reliability Engineering Book](https://sre.google/books/)
- [Chaos Engineering Principles](https://principlesofchaos.org/)
- [DORA Metrics](https://www.devops-research.com/research.html)
