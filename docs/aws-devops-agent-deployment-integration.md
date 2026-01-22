# AWS DevOps Agent Deployment Integration for Online Boutique

This guide provides deep integration patterns for enhancing observability and deployment tracking to maximize AWS DevOps Agent effectiveness with Online Boutique.

---

## Introduction: Enhanced Observability

AWS DevOps Agent's effectiveness depends on the quality and completeness of observability data:

- **Rich logs** → Faster root cause identification
- **Comprehensive metrics** → Accurate anomaly detection
- **Distributed traces** → Service dependency analysis
- **Deployment tracking** → Change correlation
- **Service annotations** → Intelligent topology mapping

This guide shows how to instrument Online Boutique for optimal agent performance.

---

## CloudWatch Integration (AWS Native Path)

### Step 1: Enable Container Insights on EKS

Container Insights provides comprehensive metrics and logs for EKS clusters.

```bash
# Create IAM policy for CloudWatch
aws iam create-policy \
  --policy-name CloudWatchAgentServerPolicy \
  --policy-document file://cloudwatch-policy.json

# Attach policy to node IAM role
aws iam attach-role-policy \
  --role-name eksNodeRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# Deploy CloudWatch agent and Fluent Bit
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml

# Verify deployment
kubectl get pods -n amazon-cloudwatch
```

### Step 2: Configure CloudWatch Logs

Create log groups for each service:

```bash
# Create log groups
aws logs create-log-group --log-group-name /online-boutique/frontend
aws logs create-log-group --log-group-name /online-boutique/cartservice
aws logs create-log-group --log-group-name /online-boutique/checkoutservice
aws logs create-log-group --log-group-name /online-boutique/paymentservice
aws logs create-log-group --log-group-name /online-boutique/shippingservice
aws logs create-log-group --log-group-name /online-boutique/emailservice
aws logs create-log-group --log-group-name /online-boutique/currencyservice
aws logs create-log-group --log-group-name /online-boutique/productcatalogservice
aws logs create-log-group --log-group-name /online-boutique/recommendationservice
aws logs create-log-group --log-group-name /online-boutique/adservice

# Set retention policy (30 days recommended)
aws logs put-retention-policy \
  --log-group-name /online-boutique/frontend \
  --retention-in-days 30
```

### Step 3: Create CloudWatch Alarms

Create alarms for each critical service:

```bash
# Frontend high CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name frontend-high-cpu \
  --alarm-description "Frontend CPU utilization exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=ServiceName,Value=frontend

# CartService error rate alarm
aws cloudwatch put-metric-alarm \
  --alarm-name cartservice-error-rate \
  --alarm-description "CartService error rate exceeds 5%" \
  --metric-name ErrorRate \
  --namespace OnlineBoutique \
  --statistic Average \
  --period 60 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2

# CheckoutService high latency alarm
aws cloudwatch put-metric-alarm \
  --alarm-name checkoutservice-high-latency \
  --alarm-description "CheckoutService P99 latency exceeds 3 seconds" \
  --metric-name P99Latency \
  --namespace OnlineBoutique \
  --statistic Maximum \
  --period 300 \
  --threshold 3000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1
```

### Step 4: CloudWatch Logs Insights Queries

Pre-configured queries for common investigations:

**Error Rate by Service:**
```
fields @timestamp, kubernetes.pod_name, log
| filter log like /error|ERROR|Error/
| stats count() as error_count by kubernetes.pod_name
| sort error_count desc
```

**Frontend Response Time Analysis:**
```
fields @timestamp, http.status_code, http.duration_ms
| filter kubernetes.pod_name like /frontend/
| stats avg(http.duration_ms) as avg_latency, max(http.duration_ms) as max_latency, count() as request_count by http.status_code
```

**CartService Redis Connection Errors:**
```
fields @timestamp, log
| filter kubernetes.pod_name like /cartservice/
| filter log like /redis|connection refused|timeout/
| sort @timestamp desc
```

**Payment Failures:**
```
fields @timestamp, transaction_id, error_message
| filter kubernetes.pod_name like /paymentservice/
| filter log like /payment failed|declined|error/
| stats count() as failures by error_message
```

---

## GitHub Actions Integration

### Deployment Tracking Workflow

Create `.github/workflows/deploy-with-tracking.yml`:

```yaml
name: Deploy with AWS DevOps Agent Tracking
on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  DEVOPS_AGENT_ENDPOINT: https://devops-agent.us-east-1.amazonaws.com/api/v1
  GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for change analysis

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v1
        with:
          project_id: ${{ secrets.GCP_PROJECT_ID }}

      - name: Configure Docker
        run: gcloud auth configure-docker

      - name: Notify deployment start
        run: |
          curl -X POST ${{ env.DEVOPS_AGENT_ENDPOINT }}/deployments \
            -H "Authorization: Bearer ${{ secrets.DEVOPS_AGENT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "event": "deployment_started",
              "repository": "${{ github.repository }}",
              "commit": "${{ github.sha }}",
              "branch": "${{ github.ref_name }}",
              "author": "${{ github.actor }}",
              "environment": "production",
              "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
            }'

      - name: Deploy with Skaffold
        id: deploy
        run: |
          skaffold run --default-repo=gcr.io/${{ secrets.GCP_PROJECT_ID }}/microservices-demo
        continue-on-error: true

      - name: Get deployment metadata
        if: always()
        id: metadata
        run: |
          # Get list of deployed images
          IMAGES=$(kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u | jq -R . | jq -s .)
          echo "images=$IMAGES" >> $GITHUB_OUTPUT

          # Get changed services
          CHANGED_SERVICES=$(git diff --name-only ${{ github.event.before }} ${{ github.sha }} | grep '^src/' | cut -d'/' -f2 | sort -u | jq -R . | jq -s .)
          echo "changed_services=$CHANGED_SERVICES" >> $GITHUB_OUTPUT

      - name: Notify deployment completion
        if: always()
        run: |
          curl -X POST ${{ env.DEVOPS_AGENT_ENDPOINT }}/deployments \
            -H "Authorization: Bearer ${{ secrets.DEVOPS_AGENT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "event": "deployment_completed",
              "repository": "${{ github.repository }}",
              "commit": "${{ github.sha }}",
              "branch": "${{ github.ref_name }}",
              "author": "${{ github.actor }}",
              "environment": "production",
              "status": "${{ steps.deploy.outcome }}",
              "changed_services": ${{ steps.metadata.outputs.changed_services }},
              "deployed_images": ${{ steps.metadata.outputs.images }},
              "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
              "workflow_run_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }'

      - name: Fail if deployment failed
        if: steps.deploy.outcome == 'failure'
        run: exit 1
```

### Per-Service Deployment Tracking

For microservice-specific deployments:

```yaml
name: Deploy Single Service
on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to deploy'
        required: true
        type: choice
        options:
          - frontend
          - cartservice
          - checkoutservice
          - paymentservice

jobs:
  deploy-service:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy ${{ inputs.service }}
        run: |
          # Build and deploy specific service
          skaffold run -m ${{ inputs.service }}

      - name: Track deployment
        run: |
          curl -X POST https://devops-agent.us-east-1.amazonaws.com/api/v1/deployments \
            -H "Authorization: Bearer ${{ secrets.DEVOPS_AGENT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "event": "deployment_completed",
              "services": ["${{ inputs.service }}"],
              "commit": "${{ github.sha }}",
              "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
            }'
```

---

## Service Annotations for Topology

Enhance Kubernetes manifests with metadata for AWS DevOps Agent:

### Frontend Service

```yaml
# kubernetes-manifests/frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
    tier: frontend
    devops-agent.aws/service-name: frontend
    devops-agent.aws/criticality: high
    devops-agent.aws/team: platform-team
    devops-agent.aws/language: go
    devops-agent.aws/communication: grpc
spec:
  template:
    metadata:
      annotations:
        devops-agent.aws/dependencies: "cartservice,productcatalogservice,currencyservice,recommendationservice,shippingservice,checkoutservice,adservice"
        devops-agent.aws/external-dependencies: "none"
        devops-agent.aws/database: "none"
        devops-agent.aws/cache: "none"
        devops-agent.aws/observability: "cloudwatch,xray"
        devops-agent.aws/owner: "platform-team@example.com"
        devops-agent.aws/runbook: "https://wiki.example.com/runbooks/frontend"
        devops-agent.aws/sla: "99.9"
```

### CartService with Redis

```yaml
# kubernetes-manifests/cartservice.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cartservice
  labels:
    app: cartservice
    tier: backend
    devops-agent.aws/service-name: cartservice
    devops-agent.aws/criticality: high
    devops-agent.aws/team: shopping-team
    devops-agent.aws/language: csharp
    devops-agent.aws/communication: grpc
spec:
  template:
    metadata:
      annotations:
        devops-agent.aws/dependencies: "none"
        devops-agent.aws/external-dependencies: "redis-cart"
        devops-agent.aws/database: "redis"
        devops-agent.aws/cache: "redis"
        devops-agent.aws/observability: "cloudwatch"
        devops-agent.aws/owner: "shopping-team@example.com"
        devops-agent.aws/runbook: "https://wiki.example.com/runbooks/cartservice"
        devops-agent.aws/sla: "99.95"
```

### CheckoutService (Critical Path)

```yaml
# kubernetes-manifests/checkoutservice.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkoutservice
  labels:
    app: checkoutservice
    tier: backend
    devops-agent.aws/service-name: checkoutservice
    devops-agent.aws/criticality: critical
    devops-agent.aws/team: checkout-team
    devops-agent.aws/language: go
    devops-agent.aws/communication: grpc
spec:
  template:
    metadata:
      annotations:
        devops-agent.aws/dependencies: "cartservice,productcatalogservice,currencyservice,shippingservice,emailservice,paymentservice"
        devops-agent.aws/external-dependencies: "none"
        devops-agent.aws/database: "none"
        devops-agent.aws/cache: "none"
        devops-agent.aws/observability: "cloudwatch,xray"
        devops-agent.aws/owner: "checkout-team@example.com"
        devops-agent.aws/runbook: "https://wiki.example.com/runbooks/checkoutservice"
        devops-agent.aws/sla: "99.99"
        devops-agent.aws/business-impact: "Revenue generation blocked if down"
```

### Annotation Reference Table

| Annotation | Purpose | Values |
|------------|---------|--------|
| `devops-agent.aws/service-name` | Service identifier | Service name |
| `devops-agent.aws/criticality` | Business criticality | `low`, `medium`, `high`, `critical` |
| `devops-agent.aws/team` | Owning team | Team name |
| `devops-agent.aws/language` | Programming language | `go`, `python`, `node`, `csharp`, `java` |
| `devops-agent.aws/communication` | Protocol | `grpc`, `http`, `rest` |
| `devops-agent.aws/dependencies` | Service dependencies | Comma-separated service names |
| `devops-agent.aws/external-dependencies` | External dependencies | DB, cache, APIs |
| `devops-agent.aws/owner` | Contact email | Email address |
| `devops-agent.aws/runbook` | Runbook URL | Documentation link |
| `devops-agent.aws/sla` | Target availability | Percentage (e.g., `99.9`) |

---

## Structured Logging Best Practices

### Go Services (frontend, checkoutservice, shippingservice, productcatalogservice)

```go
// Use structured logging with fields
import (
    "go.uber.org/zap"
    "context"
)

func main() {
    logger, _ := zap.NewProduction()
    defer logger.Sync()

    logger.Info("service started",
        zap.String("service", "frontend"),
        zap.String("version", "1.2.3"),
        zap.Int("port", 8080),
    )
}

func handleRequest(ctx context.Context, logger *zap.Logger, req *Request) {
    logger.Info("handling request",
        zap.String("request_id", req.ID),
        zap.String("trace_id", getTraceID(ctx)),
        zap.String("user_id", req.UserID),
        zap.String("session_id", req.SessionID),
        zap.Duration("duration", time.Since(start)),
    )
}

func handleError(logger *zap.Logger, err error) {
    logger.Error("operation failed",
        zap.Error(err),
        zap.String("operation", "get_cart"),
        zap.String("service", "cartservice"),
        zap.Stack("stacktrace"),
    )
}
```

### Node.js Services (currencyservice, paymentservice)

```javascript
// Use winston for structured logging
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'currencyservice',
    version: process.env.VERSION || '1.0.0'
  },
  transports: [
    new winston.transports.Console()
  ]
});

app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    logger.info('request completed', {
      request_id: req.headers['x-request-id'],
      trace_id: req.headers['x-cloud-trace-context'],
      method: req.method,
      path: req.path,
      status_code: res.statusCode,
      duration_ms: Date.now() - start
    });
  });
  next();
});

// Error logging
logger.error('external API call failed', {
  error: err.message,
  stack: err.stack,
  api: 'ECB exchange rates',
  url: apiUrl,
  status_code: err.statusCode
});
```

### Python Services (recommendationservice, emailservice)

```python
# Use structlog for structured logging
import structlog
import logging

logging.basicConfig(
    format="%(message)s",
    level=logging.INFO,
)

structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

logger = structlog.get_logger()

# Service startup
logger.info("service started",
    service="recommendationservice",
    version="1.0.0",
    port=8080
)

# Request logging
logger.info("recommendation requested",
    request_id=request_id,
    trace_id=trace_id,
    user_id=user_id,
    product_ids=product_ids,
    duration_ms=duration
)

# Error logging
logger.error("ML model inference failed",
    error=str(e),
    stack_trace=traceback.format_exc(),
    model_version="v2.1",
    input_features=features
)
```

### C# Services (cartservice)

```csharp
// Use Serilog for structured logging
using Serilog;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.WithProperty("service", "cartservice")
    .Enrich.WithProperty("version", "1.0.0")
    .WriteTo.Console(new JsonFormatter())
    .CreateLogger();

// Request logging
Log.Information("Cart operation completed",
    new {
        RequestId = requestId,
        TraceId = traceId,
        UserId = userId,
        Operation = "AddItem",
        DurationMs = duration,
        ItemCount = cart.Items.Count
    }
);

// Error logging
Log.Error(ex, "Redis connection failed",
    new {
        RedisHost = redisHost,
        RedisPort = redisPort,
        Operation = "Get",
        RetryCount = retryCount
    }
);
```

### Java Services (adservice)

```java
// Use SLF4J with Logback for structured logging
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import net.logstash.logback.marker.Markers;

public class AdService {
    private static final Logger logger = LoggerFactory.getLogger(AdService.class);

    public void serveAd(String requestId, String[] contextKeys) {
        Map<String, Object> logData = Map.of(
            "request_id", requestId,
            "trace_id", getTraceId(),
            "context_keys", contextKeys,
            "ad_count", ads.size(),
            "duration_ms", duration
        );

        logger.info(Markers.appendEntries(logData), "Ad served");
    }

    public void handleError(Exception e) {
        Map<String, Object> logData = Map.of(
            "error", e.getMessage(),
            "error_type", e.getClass().getName(),
            "operation", "get_ads"
        );

        logger.error(Markers.appendEntries(logData), "Ad service error", e);
    }
}
```

---

## Tracing Integration

### AWS X-Ray Setup (EKS)

```bash
# Deploy X-Ray daemon as DaemonSet
kubectl apply -f https://github.com/aws/aws-xray-daemon/releases/latest/download/xray-daemon-daemonset.yaml

# Update service deployments to send traces
kubectl set env deployment/frontend AWS_XRAY_DAEMON_ADDRESS=xray-service.default:2000
kubectl set env deployment/checkoutservice AWS_XRAY_DAEMON_ADDRESS=xray-service.default:2000
```

### OpenTelemetry Setup (Platform-Agnostic)

```yaml
# Deploy OpenTelemetry Collector
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    processors:
      batch:
    exporters:
      awsxray:
        region: us-east-1
      logging:
        loglevel: debug
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [awsxray, logging]
```

---

## Next Steps

1. **Test observability setup:** Trigger test incidents and verify data collection
2. **Implement prevention recommendations:** See [AWS DevOps Agent Prevention Guide](./aws-devops-agent-prevention.md)
3. **Review incident scenarios:** See [AWS DevOps Agent Incident Scenarios](./aws-devops-agent-incident-scenarios.md)

---

## Useful Links

- [AWS Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)
- [AWS X-Ray Developer Guide](https://docs.aws.amazon.com/xray/latest/devguide/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Kubernetes Labels and Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
