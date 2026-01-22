# AWS DevOps Agent Setup Guide for Online Boutique

This guide walks through setting up AWS DevOps Agent to provide autonomous incident response, root cause analysis, and operational excellence for the Online Boutique microservices application.

---

## Prerequisites

- AWS Account with access to US East (N. Virginia) region
- Deployed instance of Online Boutique (on EKS, GKE, or local Kubernetes)
- GitHub repository with Online Boutique code
- Observability platform configured (CloudWatch, Datadog, New Relic, etc.)
- (Optional) Slack workspace for incident communication
- (Optional) ServiceNow or PagerDuty for ticketing

---

## What is AWS DevOps Agent?

AWS DevOps Agent (announced December 2025) is a frontier AI agent that acts as an autonomous on-call engineer. It:

- **Automatically responds** to production incidents
- **Correlates data** across metrics, logs, traces, and deployments
- **Identifies root causes** and recommends mitigations
- **Manages incident coordination** via Slack and ticketing systems
- **Analyzes past incidents** to prevent future issues
- **Builds intelligent topology maps** of your application

**Key Differentiator:** While AWS Security Agent focuses on pre-deployment security assessment, AWS DevOps Agent monitors production operations and responds to runtime incidents.

---

## Step 1: Create Agent Space

### Choose Agent Space Strategy

| Strategy | Use Case | Example |
|----------|----------|---------|
| Per-Application | Dedicated agent for one app | `online-boutique-devops` |
| Per-Team | Shared agent for multiple apps | `platform-team-devops` |
| Centralized | Organization-wide agent | `engineering-devops` |

**Recommendation:** Start with per-application for Online Boutique.

### Create the Agent Space

1. Navigate to the [AWS DevOps Agent console](https://console.aws.amazon.com/devops-agent)
2. Click **Set up AWS DevOps Agent**
3. Configure Agent Space:
   - **Agent Space name:** `online-boutique-devops`
   - **Description:** `Incident response for Online Boutique microservices demo`
4. Choose authentication method:
   - **SSO with IAM Identity Center** (recommended for teams)
   - **IAM-only access** (for quick setup)
5. Configure permissions - create default IAM role or select existing
6. Click **Set up AWS DevOps Agent**

### Configure IAM Role Permissions

The agent requires read access to:
- **CloudWatch Logs:** Read log groups for all services
- **CloudWatch Metrics:** Query metrics and alarms
- **EKS/ECS:** Describe clusters and services (if using AWS)
- **X-Ray:** Read traces (if using X-Ray)
- **CloudFormation/CDK:** Read stack information

Example IAM policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:FilterLogEvents",
        "logs:GetLogEvents",
        "cloudwatch:DescribeAlarms",
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "eks:DescribeCluster",
        "xray:GetTraceSummaries",
        "xray:BatchGetTraces"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Step 2: Configure Observability Integrations

### Option A: AWS CloudWatch (Native Integration)

**Best for:** EKS deployments on AWS

1. **Enable Container Insights** on your EKS cluster:
   ```bash
   aws eks update-cluster-config \
     --name online-boutique-cluster \
     --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

   # Deploy CloudWatch agent
   kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml
   ```

2. **Configure log groups** in AWS DevOps Agent:
   - Go to **Observability integrations** in your agent space
   - Click **Add integration** > **Amazon CloudWatch**
   - Select log groups:
     - `/aws/containerinsights/online-boutique-cluster/application`
     - `/aws/eks/online-boutique-cluster/cluster`

3. **Add CloudWatch alarms:**
   - Select alarms for high CPU, memory, error rates
   - Agent will correlate alarms with incidents

### Option B: Third-Party Observability Platforms

**Datadog Integration:**
1. Go to **Observability integrations** > **Add integration** > **Datadog**
2. Enter Datadog API key and application key
3. Select services to monitor: `frontend`, `cartservice`, `checkoutservice`, etc.
4. Configure alert routing

**New Relic Integration:**
1. Go to **Observability integrations** > **Add integration** > **New Relic**
2. Enter New Relic license key
3. Select application: `Online Boutique`
4. Map services to entities

**Splunk Integration:**
1. Go to **Observability integrations** > **Add integration** > **Splunk**
2. Enter Splunk HEC endpoint and token
3. Configure search queries for common patterns

**Dynatrace Integration:**
1. Go to **Observability integrations** > **Add integration** > **Dynatrace**
2. Enter Dynatrace environment URL and API token
3. Select monitored entities

### Option C: Custom MCP Servers (Prometheus, Grafana)

For open-source observability stacks:

1. **Create MCP server** for Prometheus:
   ```javascript
   // mcp-server-prometheus.js
   const { MCPServer } = require('@aws/mcp-server-sdk');

   const server = new MCPServer({
     name: 'prometheus-online-boutique',
     version: '1.0.0'
   });

   server.addTool({
     name: 'query_metrics',
     description: 'Query Prometheus metrics',
     parameters: {
       query: { type: 'string', description: 'PromQL query' }
     },
     handler: async ({ query }) => {
       const response = await fetch(
         `http://prometheus:9090/api/v1/query?query=${encodeURIComponent(query)}`
       );
       return response.json();
     }
   });

   server.start();
   ```

2. **Register MCP server** in agent space:
   - Go to **Custom integrations** > **Add MCP server**
   - Enter endpoint URL: `https://your-mcp-server.example.com`
   - Test connection

---

## Step 3: Configure CI/CD Integration

### GitHub Actions Integration

1. Go to **CI/CD integrations** in your agent space
2. Click **Add integration** > **GitHub Actions**
3. Authorize the AWS DevOps Agent GitHub App
4. Select repository: `your-org/microservices-demo`

**Annotate deployments** with metadata:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy with Skaffold
        run: |
          skaffold run --default-repo=us-docker.pkg.dev/$PROJECT_ID/microservices-demo

      - name: Notify AWS DevOps Agent
        if: always()
        run: |
          curl -X POST https://devops-agent.us-east-1.amazonaws.com/api/v1/deployments \
            -H "Authorization: Bearer ${{ secrets.DEVOPS_AGENT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "repository": "${{ github.repository }}",
              "commit": "${{ github.sha }}",
              "branch": "${{ github.ref_name }}",
              "status": "${{ job.status }}",
              "environment": "production",
              "services": ["frontend", "cartservice", "checkoutservice", "paymentservice", "shippingservice", "emailservice", "currencyservice", "productcatalogservice", "recommendationservice", "adservice"]
            }'
```

### GitLab CI/CD Integration

For GitLab users:

```yaml
# .gitlab-ci.yml
deploy:
  stage: deploy
  script:
    - skaffold run --default-repo=registry.gitlab.com/$CI_PROJECT_PATH
  after_script:
    - |
      curl -X POST https://devops-agent.us-east-1.amazonaws.com/api/v1/deployments \
        -H "Authorization: Bearer $DEVOPS_AGENT_TOKEN" \
        -H "Content-Type: application/json" \
        -d "{
          \"repository\": \"$CI_PROJECT_PATH\",
          \"commit\": \"$CI_COMMIT_SHA\",
          \"branch\": \"$CI_COMMIT_REF_NAME\",
          \"status\": \"$CI_JOB_STATUS\",
          \"environment\": \"production\",
          \"services\": [\"frontend\", \"cartservice\", \"checkoutservice\"]
        }"
```

---

## Step 4: Configure Communication Channels

### Slack Integration

1. Go to **Communication channels** in your agent space
2. Click **Add integration** > **Slack**
3. Click **Add to Slack** and authorize the workspace
4. Create incident channels:
   - `#incidents-online-boutique` (main incident channel)
   - `#alerts-online-boutique` (alert notifications)
5. Configure routing rules:

| Alert Severity | Channel | Notification Level |
|----------------|---------|-------------------|
| Critical | #incidents-online-boutique | @channel |
| High | #incidents-online-boutique | @here |
| Medium | #alerts-online-boutique | Normal |
| Low | #alerts-online-boutique | Quiet |

### Microsoft Teams Integration

For Teams users:
1. Go to **Communication channels** > **Add integration** > **Microsoft Teams**
2. Add incoming webhook URL
3. Configure channel: `Online Boutique Incidents`

---

## Step 5: Configure Ticketing Systems

### ServiceNow Integration

1. Go to **Ticketing integrations** in your agent space
2. Click **Add integration** > **ServiceNow**
3. Enter ServiceNow instance URL: `https://your-instance.service-now.com`
4. Create service account and generate OAuth token
5. Configure incident creation rules:
   - **Critical/High:** Auto-create P1/P2 incidents
   - **Medium/Low:** Create tickets for tracking

### PagerDuty Integration

1. Go to **Ticketing integrations** > **Add integration** > **PagerDuty**
2. Enter PagerDuty integration key
3. Create service: `Online Boutique Production`
4. Configure escalation policy
5. Set up routing:
   - **Critical:** Page on-call engineer immediately
   - **High:** Create incident, alert via Slack
   - **Medium/Low:** Create low-urgency incident

---

## Step 6: Build Application Topology

AWS DevOps Agent automatically builds a topology map of your application. Enhance it with annotations:

### Kubernetes Service Annotations

Add to each service's deployment manifest:

```yaml
# kubernetes-manifests/frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
    devops-agent/service-name: frontend
    devops-agent/criticality: high
    devops-agent/team: platform
    devops-agent/language: go
spec:
  template:
    metadata:
      annotations:
        devops-agent/dependencies: "cartservice,productcatalogservice,currencyservice,recommendationservice,shippingservice,checkoutservice,adservice"
        devops-agent/external-dependencies: "none"
```

### Service Dependency Map

| Service | Dependencies | Criticality |
|---------|--------------|-------------|
| frontend | cartservice, productcatalogservice, currencyservice, recommendationservice, shippingservice, checkoutservice, adservice | High |
| cartservice | redis | High |
| checkoutservice | cartservice, productcatalogservice, currencyservice, shippingservice, emailservice, paymentservice | Critical |
| paymentservice | none | Critical |
| emailservice | none | Medium |
| shippingservice | none | Medium |
| currencyservice | ECB API (external) | High |
| productcatalogservice | none | High |
| recommendationservice | productcatalogservice | Low |
| adservice | none | Low |

### AWS Resource Discovery

For EKS deployments, the agent automatically discovers:
- EKS cluster configuration
- Node groups and autoscaling
- Load balancers and ingress
- RDS instances (if using managed Redis)
- VPC and security groups

---

## Step 7: Configure DevOps Web App Access

### Enable Web Application

1. Go to your agent space settings
2. Enable **Web application access**
3. Choose authentication:
   - **IAM Identity Center** (recommended for teams)
   - **IAM credentials** (for programmatic access)

### Configure Team Access

Using IAM Identity Center:
1. Go to **Access management** in agent space
2. Click **Add users or groups**
3. Assign roles:
   - **Admin:** Full configuration access
   - **Responder:** Incident investigation and response
   - **Viewer:** Read-only access to incidents

| Role | Permissions |
|------|-------------|
| Admin | Configure integrations, manage agent space |
| Responder | Investigate incidents, acknowledge alerts, run investigations |
| Viewer | View incidents, read investigation results |

---

## Testing the Setup

### Trigger Test Incident

Create a test incident to verify the end-to-end flow:

1. **Deploy load generator** to simulate high traffic:
   ```bash
   kubectl scale deployment/loadgenerator --replicas=5
   ```

2. **Create resource constraint** to trigger alerts:
   ```bash
   # Reduce frontend resources to trigger CPU alerts
   kubectl set resources deployment/frontend --limits=cpu=50m,memory=64Mi
   ```

3. **Monitor agent response:**
   - Check Slack channel for incident notification
   - Verify CloudWatch alarm correlation
   - Review agent investigation in web app
   - Confirm ticket creation in ServiceNow/PagerDuty

### Verify Data Collection

1. **Check logs** are being ingested:
   - Go to **Investigations** tab
   - Search for recent log entries from `frontend`, `cartservice`

2. **Check metrics** are available:
   - Query CPU and memory metrics
   - Verify custom metrics (request rate, error rate)

3. **Check deployment tracking:**
   - View recent deployments
   - Verify commit and author information

---

## Integration with Existing Workflows

### On-Call Runbooks

Update on-call runbooks to include AWS DevOps Agent:

```markdown
## Incident Response Process

1. **Alert received** via PagerDuty/Slack
2. **Acknowledge** in ticketing system
3. **Check AWS DevOps Agent** investigation:
   - Go to agent web app
   - Review automated analysis
   - Check suggested mitigations
4. **Implement mitigation** based on recommendations
5. **Verify resolution** with monitoring
6. **Document** incident and resolution
7. **Review prevention tab** for long-term fixes
```

### Post-Incident Reviews

Use agent analysis in PIRs:
1. Export incident timeline from agent
2. Review root cause analysis
3. Implement prevention recommendations
4. Track metrics: MTTD, MTTR improvements

---

## Multi-Platform Support

| Platform | Recommended Integration | Notes |
|----------|------------------------|-------|
| **AWS EKS** | CloudWatch Container Insights | Native integration, best observability |
| **GCP GKE** | Cloud Operations (formerly Stackdriver) | Via custom MCP server |
| **Azure AKS** | Azure Monitor | Via custom MCP server |
| **Minikube/Kind** | Prometheus + Grafana | Local development, custom MCP server |
| **Docker Desktop** | Docker logs + Prometheus | Limited functionality |

---

## Cost Considerations

**Preview Period (Current):**
- No charge for AWS DevOps Agent
- Limited to 100 task hours per month
- Standard AWS service charges apply (CloudWatch, X-Ray, etc.)

**General Availability (Future):**
- Pricing to be announced
- Estimated based on task hours and data volume

---

## Next Steps

1. **Explore incident scenarios:** See [AWS DevOps Agent Incident Scenarios](./aws-devops-agent-incident-scenarios.md)
2. **Configure deployment tracking:** See [AWS DevOps Agent Deployment Integration](./aws-devops-agent-deployment-integration.md)
3. **Implement prevention recommendations:** See [AWS DevOps Agent Prevention Guide](./aws-devops-agent-prevention.md)

---

## Useful Links

- [AWS DevOps Agent Documentation](https://docs.aws.amazon.com/devops-agent/)
- [AWS DevOps Agent Console](https://console.aws.amazon.com/devops-agent)
- [Model Context Protocol (MCP) Specification](https://github.com/aws/model-context-protocol)
- [CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)
- [Online Boutique Architecture](../README.md#architecture)
