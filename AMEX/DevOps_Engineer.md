# AMEX

**YOE---> 3 yrs**

- whats the difference between docker and kubernetes
- how to reduce the downtime with deployments
- what agents you have deployed
- suppose there are 100 applications how do you log analysis
- difference between sre and devops
- did you do any migrations from onpremises to cloud if so what challenges you have faced
- how the end point authentication works in kubernetes
- how did you troubleshoot the pod crashback loop
- what is sla and slo
- provide the agent which you worked for customer or your company
- tell me the cloud architecture which you have worked
- explain the production issue which you have faced



------

---

## 1. Difference between Docker and Kubernetes

**Docker:**
- Container runtime/platform for building, packaging, and running individual containers
- Handles container lifecycle on a single host
- Provides image format, Dockerfile, container registry
- Manages container isolation, networking, and storage on one machine
- Example: `docker run nginx`

**Kubernetes:**
- Container orchestration platform that manages multiple containers across multiple hosts
- Handles scheduling, scaling, self-healing, service discovery, load balancing
- Manages rolling updates, rollbacks, and declarative configuration
- Provides abstractions: Pods, Deployments, Services, Ingress, ConfigMaps, Secrets
- Uses container runtimes (including Docker via containerd) to run containers
- Example: `kubectl scale deployment nginx --replicas=10`

**Analogy:** Docker is like a shipping container; Kubernetes is the port management system that decides where containers go, how many are needed, and what to do if one fails.

---

## 2. How to reduce downtime with deployments

**Strategy 1: Rolling Updates**
- Deploy new version incrementally, one pod at a time
- Configure `maxSurge` and `maxUnavailable` in Kubernetes
- Zero downtime if configured correctly

**Strategy 2: Blue-Green Deployment**
- Run old (blue) and new (green) versions simultaneously
- Switch traffic to green after health checks pass
- Instant rollback by switching back to blue

**Strategy 3: Canary Deployment**
- Release new version to small percentage of users first
- Monitor metrics, gradually increase traffic
- Rollback if issues detected

**Strategy 4: Infrastructure-level practices**
- Use Readiness Probes and Liveness Probes
- Implement Health Check endpoints
- Use Pod Disruption Budgets
- Pre-pull images to reduce scheduling delay
- Use Graceful Shutdown (SIGTERM handling, `terminationGracePeriodSeconds`)
- Database migrations using expand-contract pattern
- Implement proper CI/CD with automated testing before production

**Example Kubernetes config:**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

---

## 3. What agents you have deployed?

Common agents I have deployed in production environments:

**Monitoring & Observability:**
- **Prometheus Node Exporter** – collects host metrics
- **Prometheus** – metrics collection and storage
- **Grafana Agent** – lightweight metrics/logs/traces forwarding
- **Datadog Agent** – comprehensive monitoring
- **New Relic Agent** – application performance monitoring
- **Elastic Beats (Filebeat, Metricbeat)** – log and metric shipping
- **Fluentd/Fluent Bit** – log collection and forwarding
- **OpenTelemetry Collector** – unified telemetry collection

**CI/CD & Automation:**
- **Jenkins Agent/Slave** – build execution
- **GitLab Runner** – CI/CD job execution
- **GitHub Actions Runner** – self-hosted runner
- **ArgoCD Agent** – GitOps deployment

**Security & Compliance:**
- **Falco** – runtime security monitoring
- **Vault Agent** – secrets injection into pods
- **AWS Systems Manager Agent** – EC2 management

**Networking:**
- **Istio sidecar (Envoy proxy)** – service mesh
- **NGINX Ingress Controller** – traffic routing

---

## 4. Log analysis for 100 applications

**Architecture Approach:**

**Centralized Logging Stack:**
- **Collection:** Fluentd/Fluent Bit deployed as DaemonSet on every node
- **Aggregation:** Kafka/Amazon Kinesis for buffering (handles high throughput)
- **Storage:** Elasticsearch/OpenSearch cluster with index per application
- **Visualization:** Kibana/OpenSearch Dashboards/Grafana
- **Correlation:** Add trace ID and request ID to logs

**Strategy:**
1. **Standardize log format** – JSON with fields: timestamp, service name, log level, trace ID, message
2. **Namespace-based separation** – each application in its own namespace
3. **Index lifecycle management** – hot/warm/cold phases
4. **Retention policies** – 7 days hot, 30 days warm, 90 days cold
5. **Alerting** – trigger alerts on error patterns
6. **Dashboards** – per-application and cross-application views
7. **Log sampling** for high-volume applications

**Tools:**
- ELK Stack (Elasticsearch, Logstash, Kibana)
- EFK Stack (Elasticsearch, Fluentd, Kibana)
- Loki + Grafana (lightweight alternative)
- AWS CloudWatch Logs + Log Insights
- Splunk (enterprise)

**Example Fluentd config:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      tag kubernetes.*
    </source>
    <match kubernetes.**>
      @type elasticsearch
      host elasticsearch-master
      port 9200
      logstash_format true
    </match>
```

---

## 5. Difference between SRE and DevOps

| Aspect | DevOps | SRE |
|--------|--------|-----|
| **Philosophy** | Cultural movement breaking silos between Dev and Ops | Implementation of DevOps using engineering practices |
| **Focus** | Collaboration, automation, delivery speed | Reliability, availability, performance |
| **Metrics** | Deployment frequency, lead time | SLOs, SLIs, error budgets |
| **Origin** | Started as cultural movement | Started at Google as job role |
| **Core Activities** | CI/CD, infrastructure as code, automation | Monitoring, incident response, capacity planning |
| **Approach** | Reduce friction between teams | Use engineering to solve operations problems |
| **Automation** | Automate everything possible | Automate repetitive tasks but leave creative work |

**Key Insight:** SRE is a specific implementation of DevOps principles with a focus on reliability. DevOps is the "what" and "why"; SRE is the "how."

---

## 6. On-premises to cloud migration challenges

**Challenges I have faced:**

**1. Network Connectivity**
- Setting up VPN/Direct Connect for hybrid architecture
- Managing data transfer bandwidth limitations
- Latency between on-prem and cloud resources

**Solution:** AWS Direct Connect, staged data transfer, Snowball for large datasets

**2. Application Refactoring**
- Monolithic apps not designed for cloud
- Hardcoded IPs, local file storage assumptions
- License constraints

**Solution:** Lift-and-shift first, then gradually refactor to microservices/containers

**3. Database Migration**
- Oracle/SQL Server to RDS/Aurora
- Zero-downtime migration using DMS (Database Migration Service)
- Data consistency and validation

**Solution:** AWS DMS with change data capture, dual-write period, cutover planning

**4. Security & Compliance**
- IAM policy design from scratch
- Encryption requirements
- Compliance certifications (HIPAA, PCI)

**Solution:** Landing zone with AWS Control Tower, security baseline

**5. Cultural Resistance**
- Teams used to traditional infrastructure
- Skill gap in cloud technologies

**Solution:** Training programs, proof of concepts, phased migration

**6. Cost Management**
- Initial over-provisioning
- Understanding cloud pricing models

**Solution:** Right-sizing, reserved instances, cost alerts

---

## 7. How endpoint authentication works in Kubernetes

**Authentication Flow:**

1. **Client → API Server**
   - Client (kubectl, pod, service account) sends request to API server
   - Authentication methods:
     - **Client Certificates (X.509)**
     - **Bearer Tokens (JWT)**
     - **Service Account Tokens**
     - **OIDC Tokens**
     - **Webhook Token Authentication**

2. **Authentication Phase**
   - API server validates credentials
   - Determines the identity (user or service account)
   - If authentication fails → 401 Unauthorized

3. **Authorization Phase**
   - RBAC checks if authenticated user has permission for the action
   - If authorization fails → 403 Forbidden

**Service Account Authentication (Pod → API Server):**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: default
---
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  serviceAccountName: my-app
```

- Kubernetes automatically mounts service account token at `/var/run/secrets/kubernetes.io/serviceaccount/token`
- Pod uses this token to authenticate to API server
- Token is validated by API server against service account

**User Authentication Example (OIDC):**
```yaml
apiVersion: v1
kind: Config
users:
- name: user
  user:
    auth-provider:
      name: oidc
      config:
        client-id: kubernetes
        id-token: eyJhbGciOiJSUzI1Ni...
        idp-issuer-url: https://accounts.google.com
        refresh-token: ...
```

---

## 8. Troubleshooting Pod CrashLoopBackOff

**Systematic Approach:**

```bash
# 1. Get pod details
kubectl describe pod <pod-name> -n <namespace>

# 2. Check previous container logs
kubectl logs <pod-name> -n <namespace> --previous

# 3. Check current logs
kubectl logs <pod-name> -n <namespace>

# 4. Check events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

**Common Causes and Solutions:**

**1. Application Error**
```bash
# Check logs for exception/error
kubectl logs <pod> --previous
# Fix application code or configuration
```

**2. Missing ConfigMap/Secret**
```bash
# Error: "configmap not found"
kubectl get configmap
# Create missing resource or fix reference
```

**3. Insufficient Resources**
```bash
# Error: "Insufficient cpu/memory"
# Check limits
kubectl describe pod <pod> | grep -A 5 Limits
# Increase limits or reduce requests
```

**4. Wrong Image or Tag**
```bash
# Error: "ImagePullBackOff" followed by "CrashLoopBackOff"
# Verify image exists
docker pull <image>:<tag>
# Fix image name/tag or push image
```

**5. Health Check Failing**
```bash
# Readiness probe failing immediately
# Check probe configuration
kubectl describe pod <pod> | grep -A 10 Readiness
# Adjust initialDelaySeconds or probe path
```

**6. Command/Args Wrong**
```bash
# Container exits immediately
# Check command
kubectl describe pod <pod> | grep -A 5 Command
# Fix command or entrypoint
```

**7. Permission Issues**
```bash
# Error: "permission denied"
# Check security context
kubectl describe pod <pod> | grep -A 5 Security
# Adjust fsGroup or runAsUser
```

---

## 9. What is SLA and SLO

**SLA (Service Level Agreement)**
- Legal contract between service provider and customer
- Defines guaranteed service levels
- Includes consequences if not met (credits, penalties)
- Example: "99.9% uptime per month, else 10% credit"

**SLO (Service Level Objective)**
- Internal goal/target for service reliability
- More aggressive than SLA
- Helps maintain buffer before breaching SLA
- Example: "99.95% availability target internally"

**SLI (Service Level Indicator)**
- Actual measured metric
- Examples: uptime %, latency, error rate

**Example:**
- SLI: 99.95% availability (measured)
- SLO: 99.9% availability (internal target)
- SLA: 99.5% availability (customer contract)

**Error Budget:**
- The acceptable amount of downtime: `100% - SLO = error budget`
- If SLO is 99.9%, error budget is 0.1% (~43.2 minutes/month)
- Teams can use error budget for risky deployments

---

## 10. Agent which you worked for customer or company

**Example answer for interview:**

"I have worked with the following agents in production:

**Monitoring Agents (company infrastructure):**
- **Prometheus Node Exporter** deployed via DaemonSet on all Kubernetes nodes for host metrics (CPU, memory, disk)
- **Fluent Bit** as a DaemonSet for log collection from all pods, forwarding to Elasticsearch
- **Datadog Agent** for APM traces and custom metrics in our payment services

**CI/CD Agents (customer projects):**
- **Jenkins Agents** – I configured ephemeral Jenkins agents on EKS for building Java microservices, with auto-scaling based on queue length
- **GitLab Runners** – deployed Kubernetes executors for running CI/CD pipelines in a customer's on-prem environment

**Security Agents (enterprise customer):**
- **Falco** – deployed for runtime security monitoring in a financial services customer's Kubernetes clusters
- **Vault Agent Injector** – used HashiCorp Vault sidecar injection to provide database credentials to pods without storing secrets in Kubernetes

Each agent deployment followed the same pattern: DaemonSet or sidecar, centralized configuration via ConfigMap, metrics/logs shipped to central platform, and health checks via Prometheus."

---

## 11. Cloud architecture you have worked on

**Example answer:**

"I've worked on a highly available, multi-tier architecture on AWS:

**Architecture Overview:**

**Presentation Tier:**
- Amazon CloudFront for CDN and static content delivery
- Route 53 for DNS management
- AWS WAF for security filtering

**Application Tier:**
- Amazon EKS running microservices (Node.js, Java Spring Boot)
- Amazon ECS Fargate for batch jobs
- Application Load Balancer (ALB) routing traffic
- Auto Scaling Groups for worker nodes

**Data Tier:**
- Amazon RDS (PostgreSQL) Multi-AZ for relational data
- Amazon ElastiCache (Redis) for caching
- Amazon DynamoDB for high-throughput NoSQL
- Amazon S3 for object storage and backups

**Networking:**
- Custom VPC with public/private subnets across 3 AZs
- NAT Gateways for private subnet internet access
- VPC Endpoints for S3 and ECR
- Transit Gateway for connecting to on-prem

**Security:**
- IAM Roles for Service Accounts (IRSA) for pod-level permissions
- Security Groups as primary firewall
- AWS KMS for encryption at rest
- ACM for TLS certificates

**Observability:**
- Prometheus for metrics (self-managed on EKS)
- Grafana for dashboards
- ELK Stack for log aggregation
- AWS CloudWatch for AWS service metrics
- Datadog for APM tracing

**CI/CD:**
- GitHub → Jenkins → ECR → ArgoCD → EKS
- Terraform for infrastructure as code
- Helm for application packaging

**Disaster Recovery:**
- Multi-AZ deployment for HA
- S3 cross-region replication for DR
- RDS automated backups with point-in-time recovery
- Pilot Light strategy for DR region

This architecture supported 200+ microservices, handling ~50,000 requests per second with 99.95% availability."

---

## 12. Production issue you have faced

**Example 1: Memory Leak in Production**

**Issue:** Applications in EKS started OOMKilled after 3-4 hours of uptime. Pods would restart, causing intermittent 502 errors.

**Troubleshooting:**
```bash
# 1. Check pod status
kubectl get pods -n production
# Multiple restarts observed

# 2. Check pod details
kubectl describe pod <pod-name> -n production
# OOMKilled

# 3. Check metrics
# Memory usage gradually increasing (memory leak pattern)

# 4. Check application logs
kubectl logs <pod-name> --previous
# Exception before termination
```

**Root Cause:** Java application had a connection pool leak. Database connections weren't being closed properly in error paths.

**Solution:**
- Short-term: Increased memory limit temporarily, added horizontal pod autoscaler
- Long-term: Fixed connection leak in code, added connection pool monitoring
- Preventive: Added Prometheus alerts for gradual memory increase pattern

**Example 2: Database Connection Exhaustion**

**Issue:** After a traffic spike, application couldn't connect to RDS. Error: "Too many connections"

**Troubleshooting:**
```sql
-- Check active connections
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

-- Check connection details
SELECT * FROM pg_stat_activity WHERE state = 'active';
```

**Root Cause:** Connection pool size in application wasn't properly configured. When traffic increased, each pod tried to create more connections than RDS could handle.

**Solution:**
- Reduced connection pool size per pod
- Implemented proper connection pooling (HikariCP)
- Scaled horizontally instead of increasing per-pod connections
- Added RDS connection monitoring dashboards
- Tested with load testing before peak traffic periods

**Impact:** Database was unavailable for 5 minutes, affecting 3 microservices. No data loss occurred.

**Lessons Learned:**
- Always configure connection pooling properly
- Monitor database connections proactively
- Test with production-like load
- Have rollback plan ready

---

These answers demonstrate practical knowledge and real-world experience. Customize the specific tools and numbers based on your actual experience.
