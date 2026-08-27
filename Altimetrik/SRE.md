# Altimetrik

- Question : How do you import a resource into Terraform that was created manually in AWS or GCP? What command would you use?

- Question : Can you describe your exposure to different environments like Dev, QA, and Prod?
- Question : What is your understanding of software architecture components (Load balancers, web servers, application servers, databases, and integrations)?
- Question : What is your experience with alerts, logging, and incident/problem resolution?
- Question : What is your knowledge of production system sizing, provisioning, setup, maintenance, and closure?
- Question : Describe your experience with infrastructure administration tasks like licensing, billing, cost reduction, and security.

- Question : Walk me through what is there in helm charts and how it is integrated.explain me that what is written on your helm charts?
- what is defined on your value dot yaml file?
- Question : Considering you have different environments and you have one application or which is microservice which is finally going to get deployed into one of the pod in GKE,
- right? So how it is basically getting deployed in cluster? I mean the deployment is basically failing just on the pod is currently in the error state. It is getting terminated. So how are you going to troubleshoot those such kind of Kubernetes issues?
- What is your approach of doing a troubleshooting?

- Question : Can you, talked about the dashboards of observability? Which are the observability framework tools That you're currently using?

- Question : can you tell me are you familiar with setting up dashboards and alerts yourself like creating dashboards and alerts?

- Question : So can you walk me through what all dashboards that you have created, what type of alerts that you have created?

- Question : Have you used it in your day-to-day basis of dashboarding and alerting like based on the principles of SRE golden signals to set up alerts and dashboards?

- Question : Did all those things and have you kind of done anything like SLO based learning?

- Question : can you explain me what is error budget?

- Question : In Observability there is a concept of SLO based alerting. So have you configured that in your project?

- Question : What is what is an ideal burn rate?


- Question : So how you are using terraform to deploy the cluster nodes?

- Question : Set up the nodes and everything. what do you write inside a terraform code basically now what will be inside your provider file provided ATF in your main dotf?

- Question : I'll give you an example like let's talk about AWS. You have you're going to create a VPC and subnet And your provider is AWS and I'm asking you to write in using terraform.
- Like you have to deploy, you have to basically create VPC and map you have to create a subnet and you have to attach let's be the public or a private submit. It's of your choice to attach it to your VPC.


- Question : You have optimized Kubernetes deployment configs. So can you explain me what have what was the role and what what have you done there?


- Question : You mentioned about like the are you have architected a blameless postmortem framework, right? So for continuous learning, so how I mean what have you done to improve occurrence of critical incidents that you have mentioned in your resume.
- You also mentioned about basically to reduce the MTTR so can you explain me what automation that you have done?



---
Here are direct, interview-ready answers with accurate technical detail.

---

## 1. How to Import Resource into Terraform

### Import Command:
```bash
terraform import <resource_type>.<resource_name> <resource_id>
```

### Step-by-Step Process:

**Step 1: Write Resource Configuration**
```hcl
# main.tf
resource "aws_instance" "example" {
  # Configuration will be filled after import
}
```

**Step 2: Run Import Command**
```bash
# AWS EC2 Instance
terraform import aws_instance.example i-1234567890abcdef0

# AWS VPC
terraform import aws_vpc.main vpc-12345678

# AWS S3 Bucket
terraform import aws_s3_bucket.example my-bucket-name

# GCP Compute Instance
terraform import google_compute_instance.example projects/my-project/zones/us-central1-a/instances/my-instance

# GCP VPC
terraform import google_compute_network.main projects/my-project/global/networks/my-vpc
```

**Step 3: Get Current State**
```bash
# Show imported resource state
terraform state show aws_instance.example

# List all resources in state
terraform state list
```

**Step 4: Update Configuration**
```bash
# Generate configuration from state (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf

# Or manually write configuration based on state
terraform show
```

**Step 5: Validate Import**
```bash
# Plan should show no changes
terraform plan
```

### Complete Example:
```bash
# 1. Create empty resource block
cat > main.tf << EOF
resource "aws_instance" "web" {
  # To be filled after import
}
EOF

# 2. Initialize Terraform
terraform init

# 3. Import the resource
terraform import aws_instance.web i-0abcdef1234567890

# 4. Show imported state
terraform state show aws_instance.web

# 5. Write actual configuration
cat > main.tf << EOF
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  
  tags = {
    Name = "WebServer"
  }
}
EOF

# 6. Verify no changes
terraform plan
```

---

## 2. Exposure to Different Environments (Dev, QA, Prod)

### Environment Setup:

**Development (DEV):**
- **Purpose:** Developer testing and feature development
- **Infrastructure:** Smaller instances, fewer replicas
- **Data:** Dummy/test data
- **Access:** All developers
- **Deployment:** Auto-deploy on every commit
- **Example:** 1 replica, t3.small instances

**Quality Assurance (QA/Staging):**
- **Purpose:** Testing, validation, UAT
- **Infrastructure:** Similar to production but smaller
- **Data:** Sanitized production data or synthetic data
- **Access:** QA team, developers
- **Deployment:** Auto-deploy on release branches
- **Example:** 2 replicas, t3.medium instances

**Production (PROD):**
- **Purpose:** Live application for end users
- **Infrastructure:** Full-scale, highly available
- **Data:** Real production data
- **Access:** Limited (only ops team)
- **Deployment:** Manual approval, scheduled releases
- **Example:** 3+ replicas, t3.large instances, multi-AZ

### Environment Management:

**Terraform Workspaces:**
```hcl
# Using workspaces for environments
terraform workspace new dev
terraform workspace new qa
terraform workspace new prod

# Use workspace in configuration
locals {
  environment = terraform.workspace
  
  instance_size = {
    dev  = "t3.small"
    qa   = "t3.medium"
    prod = "t3.large"
  }
  
  replica_count = {
    dev  = 1
    qa   = 2
    prod = 3
  }
}

resource "aws_instance" "app" {
  instance_type = local.instance_size[local.environment]
  count         = local.replica_count[local.environment]
}
```

**Kubernetes Namespaces:**
```yaml
# Separate namespaces per environment
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: qa
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

**Configuration Management:**
```yaml
# values-dev.yaml
replicaCount: 1
resources:
  limits:
    cpu: 500m
    memory: 512Mi

# values-prod.yaml
replicaCount: 3
resources:
  limits:
    cpu: 2000m
    memory: 2Gi
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
```

---

## 3. Software Architecture Components

### Complete Architecture Understanding:

**1. Load Balancers:**
- **Purpose:** Distribute traffic across multiple servers
- **Types:**
  - L4 (Network): TCP/UDP, IP-based routing
  - L7 (Application): HTTP/HTTPS, content-based routing
- **Examples:** AWS ALB/NLB, NGINX, HAProxy, F5
- **Key Features:** Health checks, SSL termination, session persistence

**2. Web Servers:**
- **Purpose:** Serve static content, handle HTTP requests
- **Examples:** NGINX, Apache, IIS
- **Responsibilities:** Static files, reverse proxy, caching
- **Configuration:** Virtual hosts, SSL certificates, compression

**3. Application Servers:**
- **Purpose:** Run business logic, process dynamic requests
- **Examples:** Tomcat, Node.js, Python/Django, Java/Spring
- **Responsibilities:** Business logic, data processing, API endpoints
- **Communication:** REST APIs, message queues, RPC

**4. Databases:**
- **Types:**
  - Relational (SQL): PostgreSQL, MySQL, Oracle
  - NoSQL: MongoDB, Cassandra, DynamoDB
  - Cache: Redis, Memcached
- **Considerations:** Replication, sharding, backup, failover

**5. Integrations:**
- **Types:**
  - Message Queues: RabbitMQ, Kafka, SQS
  - API Gateways: Kong, AWS API Gateway
  - Service Mesh: Istio, Linkerd
- **Patterns:** Synchronous vs Asynchronous, Pub/Sub

### Architecture Flow:
```
User → DNS → CDN → Load Balancer → Web Server → Application Server → Database
                    ↓                    ↓              ↓
                 Cache              Message Queue   External APIs
```

---

## 4. Experience with Alerts, Logging, Incident Resolution

### Alerting Setup:

**1. Alert Types:**
```yaml
# Prometheus Alert Rules
groups:
- name: application
  rules:
  - alert: HighErrorRate
    expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High error rate detected"
      
  - alert: HighLatency
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High latency detected"
```

**2. Logging Pipeline:**
```
Application → Filebeat/Fluentd → Kafka → Logstash → Elasticsearch → Kibana
```

**3. Incident Response:**
```bash
# SEV Levels
SEV1: Complete outage, immediate response (15 min)
SEV2: Major feature broken, response within 1 hour
SEV3: Minor issue, response within 4 hours
SEV4: Cosmetic issue, next business day
```

**4. Resolution Process:**
1. Alert triggered
2. On-call engineer acknowledges
3. Investigate using dashboards/logs
4. Mitigate (not necessarily fix)
5. Root cause analysis
6. Permanent fix
7. Post-mortem

---

## 5. Production System Sizing, Provisioning, Setup

### Sizing Considerations:

**1. Capacity Planning:**
```
- Expected users: 100,000
- Peak concurrent users: 10,000
- Average requests/second: 1,000
- Peak requests/second: 5,000
- Data growth: 10GB/month
```

**2. Resource Calculation:**
```python
# CPU Requirements
requests_per_second = 1000
cpu_per_request_ms = 50  # milliseconds
total_cpu_ms = requests_per_second * cpu_per_request_ms
total_cores_needed = total_cpu_ms / 1000
# Add 30% buffer
final_cores = total_cores_needed * 1.3

# Memory Requirements
per_instance_memory = 512  # MB
instances = 10
total_memory = per_instance_memory * instances
# Add 50% buffer
final_memory = total_memory * 1.5
```

**3. Provisioning:**
```hcl
# Terraform for production
resource "aws_instance" "app" {
  count         = 3
  instance_type = "t3.large"  # 2 vCPU, 8GB RAM
  
  root_block_device {
    volume_size = 100  # GB
    volume_type = "gp3"
  }
}
```

**4. Maintenance:**
- Regular patching (monthly)
- Security updates (immediate)
- Performance monitoring
- Capacity reviews
- Cost optimization

---

## 6. Infrastructure Administration (Licensing, Billing, Cost Reduction, Security)

### Cost Optimization:

**1. Right-Sizing:**
```bash
# Analyze usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-31T23:59:59Z \
  --period 3600 \
  --statistics Average
```

**2. Reserved Instances:**
- 1-year term: ~40% savings
- 3-year term: ~60% savings

**3. Auto-Scaling:**
```yaml
# Scale down during off-hours
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  minReplicas: 1  # Reduced from 3
  maxReplicas: 10
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

**4. Security:**
- IAM least privilege
- Security groups
- Encryption at rest and in transit
- Regular security audits

---

## 7. Helm Charts and Integration

### Helm Chart Structure:
```
myapp-chart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration
├── charts/             # Sub-charts
├── templates/          # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── _helpers.tpl    # Template helpers
└── .helmignore
```

### Chart.yaml:
```yaml
apiVersion: v2
name: myapp
description: My application chart
type: application
version: 0.1.0
appVersion: "1.16.0"

dependencies:
  - name: postgresql
    version: "12.1.6"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

### values.yaml:
```yaml
# Default values
replicaCount: 3

image:
  repository: myregistry/myapp
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

configMap:
  APP_ENV: production
  LOG_LEVEL: info

secrets:
  DB_PASSWORD: ""  # Override at deployment

readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 20
```

### Environment-Specific values:
```yaml
# values-dev.yaml
replicaCount: 1
resources:
  limits:
    memory: "256Mi"
    cpu: "250m"
configMap:
  APP_ENV: development

---
# values-prod.yaml
replicaCount: 5
resources:
  limits:
    memory: "2Gi"
    cpu: "2000m"
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
configMap:
  APP_ENV: production
```

### Deployment:
```bash
# Install chart
helm install myapp ./myapp-chart -f values-prod.yaml

# Upgrade
helm upgrade myapp ./myapp-chart -f values-prod.yaml

# Rollback
helm rollback myapp 1
```

---

## 8. Troubleshooting Kubernetes Pod Errors

### Systematic Approach:

**Step 1: Check Pod Status**
```bash
kubectl get pods -n <namespace>
# Output: NAME READY STATUS RESTARTS AGE
# pod-name 0/1 CrashLoopBackOff 5 10m
```

**Step 2: Describe Pod**
```bash
kubectl describe pod <pod-name> -n <namespace>
# Look for:
# - Events section
# - Container status
# - Last state
# - Exit codes
```

**Step 3: Check Logs**
```bash
# Current logs
kubectl logs <pod-name> -n <namespace>

# Previous container logs
kubectl logs <pod-name> --previous -n <namespace>

# Follow logs
kubectl logs -f <pod-name> -n <namespace>
```

**Step 4: Check Resources**
```bash
# Resource usage
kubectl top pods -n <namespace>
kubectl top nodes

# Resource limits
kubectl get pods <pod-name> -n <namespace> -o yaml | grep -A 10 resources
```

**Step 5: Check Events**
```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
```

**Step 6: Common Issues and Fixes:**

| Issue | Symptom | Fix |
|-------|---------|-----|
| ImagePullBackOff | Can't pull image | Check image name, registry auth |
| CrashLoopBackOff | Container exits | Check logs, fix application |
| OOMKilled | Exit 137 | Increase memory limits |
| Pending | No resources | Add nodes or reduce requests |
| FailedScheduling | No matching nodes | Check node selectors, resources |

---

## 9. Observability Tools and Dashboards

### Tools Used:

**1. Monitoring Stack:**
- **Prometheus** - Metrics collection
- **Grafana** - Dashboards
- **Alertmanager** - Alert routing
- **Node Exporter** - Host metrics
- **kube-state-metrics** - Kubernetes metrics

**2. Logging Stack:**
- **ELK** (Elasticsearch, Logstash, Kibana)
- **Loki** - Lightweight logging
- **Fluentd** - Log collection

**3. Tracing:**
- **Jaeger** - Distributed tracing
- **OpenTelemetry** - Unified instrumentation

### Dashboards Created:

**1. Application Dashboard:**
```yaml
# Grafana Dashboard
- Panel 1: Request Rate (requests/sec)
  Query: sum(rate(http_requests_total[5m]))
  
- Panel 2: Error Rate
  Query: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
  
- Panel 3: Latency (p95)
  Query: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
  
- Panel 4: Active Users
  Query: sum(active_users)
```

**2. Infrastructure Dashboard:**
```yaml
- Panel 1: CPU Usage per Node
  Query: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  
- Panel 2: Memory Usage
  Query: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
  
- Panel 3: Disk I/O
  Query: rate(node_disk_io_time_seconds_total[5m])
  
- Panel 4: Network Traffic
  Query: rate(node_network_receive_bytes_total[5m])
```

**3. Business Dashboard:**
```yaml
- Panel 1: Orders per minute
- Panel 2: Revenue per hour
- Panel 3: User signups
- Panel 4: Conversion rate
```

### Alerts Configured:

```yaml
# Prometheus Alert Rules
groups:
- name: SLO_alerts
  rules:
  - alert: HighErrorRate
    expr: |
      sum(rate(http_requests_total{status=~"5.."}[5m])) / 
      sum(rate(http_requests_total[5m])) > 0.01
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Error rate above 1%"
      
  - alert: HighLatency
    expr: |
      histogram_quantile(0.95, 
        sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 0.5
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "P95 latency above 500ms"
```

---

## 10. SRE Golden Signals and Error Budget

### Golden Signals:
1. **Latency** - Time to serve request
2. **Traffic** - Requests per second
3. **Errors** - Error rate
4. **Saturation** - Resource utilization

### Error Budget:
```
Error Budget = 100% - SLO

Example:
SLO = 99.9% availability
Error Budget = 0.1% = 43.2 minutes/month
```

### SLO-Based Alerting:

```yaml
# Multi-window, multi-burn-rate alerting
groups:
- name: slo-alerts
  rules:
  - alert: HighBurnRate
    expr: |
      (
        sum(rate(http_requests_total{status=~"5.."}[1h])) / 
        sum(rate(http_requests_total[1h]))
      ) > 14.4 * 0.001
    for: 1h
    labels:
      severity: page
    annotations:
      summary: "High burn rate detected"
```

### Ideal Burn Rate:
- **Fast burn:** 14.4x (alerts in 1 hour)
- **Slow burn:** 1x (alerts in 3 days)

---

## 11. Terraform for Cluster Nodes

### Provider Configuration:
```hcl
# provider.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "terraform-state"
    key            = "eks/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = "us-east-1"
  
  default_tags {
    tags = {
      Environment = "production"
      ManagedBy   = "Terraform"
    }
  }
}
```

### Main Configuration:
```hcl
# main.tf
# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  
  tags = {
    Name = "eks-vpc"
  }
}

# Subnets
resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = element(["us-east-1a", "us-east-1b", "us-east-1c"], count.index)
  
  tags = {
    Name = "private-subnet-${count.index + 1}"
    "kubernetes.io/role/internal-elb" = "1"
  }
}

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = element(["us-east-1a", "us-east-1b", "us-east-1c"], count.index)
  map_public_ip_on_launch = true
  
  tags = {
    Name = "public-subnet-${count.index + 1}"
    "kubernetes.io/role/elb" = "1"
  }
}

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.0.0"
  
  cluster_name    = "production-eks"
  cluster_version = "1.28"
  
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.private[*].id
  
  node_groups = {
    general = {
      desired_capacity = 3
      min_capacity     = 1
      max_capacity     = 10
      
      instance_types = ["t3.large"]
      
      labels = {
        role = "general"
      }
    }
  }
}
```

---

## 12. Optimized Kubernetes Deployment

### Before Optimization:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080
```

### After Optimization:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: topology.kubernetes.io/zone
      containers:
      - name: myapp
        image: myapp:1.2.3  # Pinned version
        ports:
        - name: http
          containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

---

## 13. Blameless Postmortem and MTTR Reduction

### Blameless Postmortem Framework:

**1. Document Structure:**
```
- Incident Summary
- Timeline
- Root Cause
- Impact
- What Went Well
- What Went Wrong
- Action Items
- Lessons Learned
```

**2. Automation for MTTR Reduction:**

**Automated Runbooks:**
```python
#!/usr/bin/env python3
# Auto-remediation for common issues

import subprocess
import json
from datetime import datetime

def check_pod_health():
    """Check pod health and auto-remediate"""
    cmd = "kubectl get pods --all-namespaces -o json"
    pods = json.loads(subprocess.check_output(cmd.split()))
    
    for pod in pods['items']:
        # Check for CrashLoopBackOff
        if pod['status'].get('containerStatuses'):
            for container in pod['status']['containerStatuses']:
                if container.get('restartCount', 0) > 5:
                    namespace = pod['metadata']['namespace']
                    pod_name = pod['metadata']['name']
                    print(f"Auto-restarting {namespace}/{pod_name}")
                    
                    # Get logs before restart
                    subprocess.run(
                        f"kubectl logs {pod_name} -n {namespace} --previous > /tmp/{pod_name}.log",
                        shell=True
                    )
                    
                    # Restart pod
                    subprocess.run(
                        f"kubectl delete pod {pod_name} -n {namespace}",
                        shell=True
                    )

def auto_scale_database():
    """Auto-scale database if connections high"""
    cmd = "kubectl get pods -n database -o json"
    pods = json.loads(subprocess.check_output(cmd.split()))
    
    # Check database connections
    for pod in pods['items']:
        if 'postgres' in pod['metadata']['name']:
            # Query connections
            conn_cmd = f"kubectl exec {pod['metadata']['name']} -n database -- psql -c 'SELECT count(*) FROM pg_stat_activity'"
            result = subprocess.run(conn_cmd, shell=True, capture_output=True, text=True)
            
            # If connections > 80%, scale up
            if int(result.stdout.split('\n')[2].strip()) > 80:
                subprocess.run("kubectl scale deployment postgres -n database --replicas=3", shell=True)

if __name__ == "__main__":
    check_pod_health()
    auto_scale_database()
```

**Automated Alert Triage:**
```yaml
# Alertmanager configuration
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
  - match:
      severity: critical
    receiver: 'on-call'
    continue: true
  - match:
      alertname: HighErrorRate
    receiver: 'auto-remediation'
    
receivers:
- name: 'on-call'
  pagerduty_configs:
  - service_key: '<key>'
  
- name: 'auto-remediation'
  webhook_configs:
  - url: 'http://remediation-service/trigger'
```

**CI/CD for Infrastructure:**
```yaml
# GitHub Actions for auto-remediation
name: Auto-Remediation
on:
  workflow_dispatch:
  schedule:
    - cron: '*/5 * * * *'

jobs:
  remediate:
    runs-on: ubuntu-latest
    steps:
      - name: Check for issues
        run: python3 check_health.py
      
      - name: Auto-fix
        if: failure()
        run: |
          kubectl rollout restart deployment/myapp
          kubectl scale deployment/myapp --replicas=3
```

These answers demonstrate practical experience and best practices for DevOps/SRE roles.
