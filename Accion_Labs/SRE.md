# Accion Labs

**Exp---17yrs SRE**

1)  How do you ensure high availability in Kubernetes?
2) What are SLO, SLI, and SLA, and why are they important?
3) Can you describe a recent major incident you handled and how you resolved it?
4) What is the deployment setup in your organization?
5) What is the maximum time taken for a node to start after a failure or restart?
6) How have you resolved high-performance issues or critical incidents?
7) What is the difference between observability and monitoring?
8) What is the Linux command used for mounting a file system?
9) How do you detect the root cause when an application goes down in the cloud?
10) How do you respond when:
- a) the master node goes down?
- b) a worker/slave node goes down?
11) How do you configure a VPC for high availability?
12) Have you worked on scripting? if so which tool ? explain what you have implemented?
13) Have you written Terraform code for deployments? If yes, can you explain the implementation?
14) Can you explain your hands-on experience with Docker?
15) Why do we use workspaces in Terraform?
16) Tell me about Ansible. Have you worked on it before? If yes, in what context?







---
Here are direct, interview-ready answers with accurate technical detail.

---

## 1. How do you ensure high availability in Kubernetes?

**Control Plane HA:**
- Deploy multiple master nodes (3 or 5) across availability zones
- Use external etcd cluster or stacked etcd with proper quorum
- Configure API server behind a load balancer
- Set up `kube-controller-manager` and `kube-scheduler` with leader election

**Worker Node HA:**
- Spread worker nodes across multiple availability zones
- Use auto-scaling groups to replace failed nodes
- Configure Pod Disruption Budgets (PDBs)
- Set up node auto-repair (cloud provider integration)

**Application HA:**
- Deploy applications with multiple replicas (min 2-3)
- Use anti-affinity rules to spread pods across nodes/AZs
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: my-app
      topologyKey: topology.kubernetes.io/zone
```

- Configure Rolling Updates strategy with zero downtime
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

**Infrastructure HA:**
- Use multiple AZs for cluster
- Configure etcd backups regularly
- Use external storage (EBS multi-AZ, EFS, RDS Multi-AZ)
- Implement Service Mesh (Istio/Linkerd) for resilient communication
- Configure Ingress with multiple replicas
- Use Horizontal Pod Autoscaler (HPA) and Cluster Autoscaler

**Networking HA:**
- Multiple ingress controllers behind cloud load balancer
- Configure DNS failover with health checks
- Use Network Policies for isolation

---

## 2. What are SLO, SLI, and SLA, and why are they important?

**SLI (Service Level Indicator):**
- The actual measured metric of service performance
- Examples:
  - Availability: 99.95% (measured)
  - Latency: 200ms at 95th percentile
  - Error rate: 0.1% of requests
  - Throughput: 5000 requests/second

**SLO (Service Level Objective):**
- Internal target for an SLI
- Usually more strict than SLA
- Examples:
  - Availability target: 99.9%
  - Latency target: <300ms for 99% of requests
  - Error rate: <0.5%

**SLA (Service Level Agreement):**
- Legal contract with customers
- Defines promised service level
- Includes consequences if violated (refunds, credits)
- Example: 99.5% availability or 20% credit

**Why Important:**
1. **Error Budget:** `100% - SLO = Error Budget` (acceptable downtime)
2. **Decision Making:** Can we deploy this risky change? If error budget is exhausted, no.
3. **Customer Trust:** SLA sets expectations clearly
4. **Priority Setting:** If SLO is met, focus on features; if not, focus on reliability
5. **Incident Response:** Triggers alerts when SLOs are being breached

**Example:**
- SLI: 99.95% availability (actual measurement)
- SLO: 99.9% availability (internal target) = 43.2 minutes downtime/month
- SLA: 99.5% availability (customer promise) = 216 minutes downtime/month

---

## 3. Can you describe a recent major incident you handled and how you resolved it?

**Incident: Production Database Outage Due to Connection Pool Exhaustion**

**Timeline:**
- **14:00** - Alert triggered: "RDS CPU utilization above 90%"
- **14:05** - On-call engineer acknowledged alert
- **14:10** - Multiple services start throwing "too many connections" errors
- **14:15** - Incident declared as SEV-1 (Critical)
- **14:20** - I joined the incident bridge

**Initial Investigation:**
```bash
# Check RDS connections
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
# Result: 1000+ connections (max limit reached)

# Check application logs
kubectl logs <pod-name> --tail=100
# Error: "FATAL: sorry, too many clients already"

# Check recent deployments
kubectl get deployments -n production --sort-by=.metadata.creationTimestamp
# Found: New version deployed 1 hour ago
```

**Root Cause:**
- A recent deployment changed connection pool settings
- HikariCP `maximumPoolSize` was accidentally increased from 10 to 100 per pod
- With 30 pods running, that's 3000 potential connections
- RDS max connections was 1000

**Immediate Mitigation (14:30):**
1. Rolled back deployment to previous version
```bash
kubectl rollout undo deployment/payment-service -n production
```
2. Restarted RDS to clear connection queue
3. Verified services recovering

**Permanent Fix:**
- Added validation in CI/CD pipeline for connection pool settings
- Implemented connection pool monitoring in Grafana
- Added alert when connection usage exceeds 70%
- Performed load testing with production-like traffic

**Impact:**
- Total downtime: 25 minutes
- Affected: 5 microservices (payment, order, notification)
- No data loss

**Lessons Learned:**
1. Connection pool changes need review and load testing
2. Alert thresholds should be more aggressive
3. Rollback should be faster (automated rollback on error rate)
4. Added chaos engineering tests for DB connection exhaustion

---

## 4. What is the deployment setup in your organization?

**Current Setup (My Organization):**

**CI/CD Pipeline:**
```
GitHub/GitLab → Jenkins/GitHub Actions → Build → Test → Package (Docker) → Push to ECR/Artifactory → Deploy to Kubernetes via ArgoCD/Helm
```

**Environment Setup:**
- **Development:** Auto-deploy on every commit to dev branch
- **Staging:** Auto-deploy on merge to staging branch
- **Production:** Manual approval after successful staging deployment

**Deployment Strategy:**
- **Rolling Updates** for most services (zero downtime)
- **Blue-Green** for critical services (payment, authentication)
- **Canary** for high-traffic services (search, recommendations)

**Tools:**
- **CI:** Jenkins (self-hosted on EKS) or GitHub Actions
- **Container Registry:** AWS ECR
- **CD:** ArgoCD (GitOps) or Helm
- **Config Management:** Helm charts + Kustomize
- **Secrets:** AWS Secrets Manager + External Secrets Operator
- **Infrastructure:** Terraform

**Example ArgoCD Application:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/payment-service
    targetRevision: main
    path: helm/payment-service
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**Release Process:**
1. Developer creates PR
2. CI runs unit tests, integration tests, security scans
3. PR merged → CI builds image → pushes to ECR
4. ArgoCD detects image change → syncs to staging
5. QA validates on staging
6. Manual approval → ArgoCD syncs to production
7. Automated smoke tests on production
8. Metrics monitored for 30 minutes post-deployment

---

## 5. What is the maximum time taken for a node to start after a failure or restart?

**Based on my experience:**

**EC2 Node (AWS EKS):**
- **EC2 instance launch:** 2-5 minutes
- **Node bootstrapping (kubelet, containerd):** 1-2 minutes
- **Node joins cluster (Ready state):** 1-2 minutes
- **Total: 4-9 minutes** typically

**Fargate/Serverless:**
- **Pod scheduling on Fargate:** 30-60 seconds
- **Total: 1-2 minutes**

**Factors Affecting Start Time:**
1. **AMI size:** Custom AMIs with pre-installed packages start faster
2. **Instance type:** Larger instances may take longer
3. **User data scripts:** Additional bootstrap scripts add time
4. **Container image pull time:** Large images (>1GB) take longer
5. **Storage initialization:** EBS volumes need time to attach
6. **Cluster autoscaler:** Detection + provisioning time (2-5 min additional)

**Optimization Techniques:**
- **Pre-baked AMIs** with all necessary components
- **Image caching** (e.g., using local registry or pre-pulled images)
- **Warm pool** in auto-scaling group
- **Overprovisioning** (running spare capacity)
- **Using smaller, optimized container images** (distroless, alpine)

**Real-world example:**
```yaml
# Cluster Autoscaler configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler
data:
  scale-down-unneeded-time: 10m
  scale-down-delay-after-add: 10m
  max-node-provision-time: 15m
```

---

## 6. How have you resolved high-performance issues or critical incidents?

**Case 1: High CPU Usage in Kubernetes Cluster**

**Issue:** Alert: Node CPU > 90% for 15 minutes

**Troubleshooting:**
```bash
# 1. Identify high CPU pods
kubectl top pods -n production --sort-by=cpu

# 2. Check node resources
kubectl top nodes

# 3. Check pod resource usage details
kubectl describe pod <pod-name> -n production | grep -A 5 Resources

# 4. Check application metrics
# Found: One service consuming 400% CPU (4 cores)
```

**Root Cause:** Inefficient database query causing high CPU in application

**Resolution:**
- Immediate: Scaled deployment from 3 to 6 replicas
- Immediate: Added CPU limits to prevent node exhaustion
- Permanent: Optimized database query (added index)
- Permanent: Implemented caching layer (Redis)
- Permanent: Added HPA based on CPU utilization

**Case 2: High Memory Usage and OOM Kills**

**Issue:** Pods restarting with OOMKilled status

**Resolution Steps:**
1. Identified memory leak in Java application
2. Short-term: Increased memory limits
3. Long-term: Fixed connection leak in code
4. Added JVM heap dump analysis in CI
5. Implemented memory-based HPA

**Case 3: Network Latency Issues**

**Issue:** API response time increased from 200ms to 5s

**Troubleshooting:**
```bash
# Network debugging
kubectl exec -it <pod> -- curl -v http://backend-service:8080/health

# Check service mesh metrics
kubectl logs <istio-ingressgateway> -n istio-system

# Found: DNS resolution delay
```

**Resolution:**
- Fixed CoreDNS configuration
- Increased CoreDNS replicas
- Implemented connection pooling
- Added network policies optimization

---

## 7. What is the difference between observability and monitoring?

**Monitoring:**
- **Focus:** Known unknowns - tracking predefined metrics
- **Approach:** Reactive - alert on known failure patterns
- **Data:** Metrics, logs (structured, predefined)
- **Questions:** "Is the system working?" "Is CPU above 90%?"
- **Tools:** Prometheus, Nagios, CloudWatch

**Observability:**
- **Focus:** Unknown unknowns - understanding system behavior
- **Approach:** Proactive - explore system state to find issues
- **Data:** Metrics, logs, traces (all correlated)
- **Questions:** "Why is the system slow?" "What's happening inside?"
- **Tools:** Jaeger, OpenTelemetry, Grafana Tempo

**Key Differences:**

| Aspect | Monitoring | Observability |
|--------|-----------|---------------|
| **Purpose** | Detect known issues | Understand system behavior |
| **Data Types** | Metrics, logs | Metrics, logs, traces, events |
| **Approach** | Alert on thresholds | Investigate correlations |
| **Flexibility** | Fixed dashboards | Ad-hoc queries |
| **Proactivity** | Reactive | Proactive |
| **Questions** | What happened? | Why did it happen? |
| **Example** | "CPU > 90% alert" | "Trace shows slow DB query from service A" |

**Modern Approach:**
- Implement both: Monitoring for alerts, Observability for debugging
- Use OpenTelemetry for unified telemetry collection
- Correlate metrics, logs, and traces
- Example: Alert fires → View trace → See DB query slow → Query logs → Find root cause

---

## 8. What is the Linux command used for mounting a file system?

**Mount Command:**
```bash
# Basic mount syntax
mount [options] <device> <mount_point>

# Examples:
# Mount a partition
mount /dev/sdb1 /mnt/data

# Mount with specific filesystem type
mount -t ext4 /dev/sdb1 /mnt/data

# Mount NFS share
mount -t nfs 192.168.1.100:/shared /mnt/nfs

# Mount with options
mount -o rw,noatime /dev/sdb1 /mnt/data

# Mount from /etc/fstab
mount -a

# Mount ISO file
mount -o loop /path/to/file.iso /mnt/iso
```

**Related Commands:**
```bash
# View mounted filesystems
df -h
mount
cat /proc/mounts

# Unmount
umount /mnt/data
umount -l /mnt/data  # Lazy unmount

# Find filesystem type
blkid /dev/sdb1
lsblk -f

# Auto-mount on boot (edit /etc/fstab)
/dev/sdb1  /mnt/data  ext4  defaults  0  2
```

**Troubleshooting Mount Issues:**
```bash
# Check if device exists
lsblk

# Check filesystem
fsck /dev/sdb1

# Force unmount busy filesystem
fuser -km /mnt/data
umount /mnt/data

# Remount read-only filesystem as read-write
mount -o remount,rw /
```

---

## 9. How do you detect the root cause when an application goes down in the cloud?

**Systematic Troubleshooting Approach:**

**Step 1: Initial Assessment**
```bash
# Check application status
kubectl get pods -n <namespace>
# Check services
kubectl get svc -n <namespace>
# Check ingress
kubectl get ingress -n <namespace>
```

**Step 2: Check Application Logs**
```bash
# Get pod logs
kubectl logs <pod-name> -n <namespace> --tail=100
# Check previous container logs (if restarted)
kubectl logs <pod-name> -n <namespace> --previous
# Stream logs
kubectl logs -f <pod-name> -n <namespace>
```

**Step 3: Check Events**
```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
kubectl describe pod <pod-name> -n <namespace>
```

**Step 4: Check Resource Usage**
```bash
# CPU/Memory usage
kubectl top pods -n <namespace>
kubectl top nodes

# Check resource limits
kubectl describe pod <pod-name> | grep -A 10 Limits
```

**Step 5: Check Dependencies**
```bash
# Check if database is accessible
kubectl exec -it <pod> -- nc -zv <db-host> 5432
# Check DNS resolution
kubectl exec -it <pod> -- nslookup <service-name>
# Check external services
kubectl exec -it <pod> -- curl -I https://external-api.com/health
```

**Step 6: Check Network**
```bash
# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>
# Check network policies
kubectl get networkpolicies -n <namespace>
# Test connectivity
kubectl run test-pod --rm -it --image=busybox -- /bin/sh
wget -O- http://<service-name>:<port>/health
```

**Step 7: Check Cloud Resources**
```bash
# AWS-specific checks
aws rds describe-db-instances --query 'DBInstances[*].[DBInstanceStatus]'
aws elbv2 describe-target-health --target-group-arn <arn>
aws ec2 describe-instance-status --instance-ids <id>
```

**Step 8: Use Observability Tools**
- **Metrics:** Check Prometheus/Grafana for anomalies
- **Traces:** Use Jaeger/Datadog APM to trace requests
- **Logs:** Use ELK/Loki to search for error patterns

**Step 9: Time Correlation**
- What changed recently?
```bash
# Check recent deployments
kubectl rollout history deployment/<name> -n <namespace>
# Check config changes
kubectl get configmap -n <namespace> --sort-by=.metadata.creationTimestamp
```

---

## 10. How do you respond when:

### a) The master node goes down?

**Response Plan:**

**1. Immediate Assessment (2 minutes):**
- Identify which master node is down
- Check if cluster is still operational (if multiple masters)
- Alert team and stakeholders

**2. If Single Master:**
```bash
# Check if API server is accessible
kubectl get nodes
# If API server down, check kubelet
systemctl status kubelet
# Check container runtime
systemctl status docker/containerd
# Check kubelet logs
journalctl -u kubelet -n 100
```

**3. Recovery Steps:**
- **If VM/instance down:** Start the instance in cloud console
- **If kubelet failed:** Restart kubelet
```bash
systemctl restart kubelet
```
- **If etcd corrupted:** Restore from etcd backup
```bash
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=master-1 \
  --initial-cluster=master-1=https://10.0.0.1:2380
```
- **If control plane components failed:** Check and restart containers
```bash
docker ps -a | grep kube-system
docker restart <container-id>
```

**4. High Availability Setup:**
- **Multiple masters (3 or 5) with etcd quorum**
- **API server behind load balancer**
- **etcd backups every hour**
- **Auto-recovery via systemd**

**Example etcd backup:**
```bash
#!/bin/bash
etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db
```

### b) A worker/slave node goes down?

**Response Plan:**

**1. Immediate Assessment:**
```bash
# Check node status
kubectl get nodes
# Node shows NotReady

# Describe node
kubectl describe node <node-name>
# Check node conditions
kubectl get node <node-name> -o yaml | grep -A 10 conditions
```

**2. Identify Impact:**
```bash
# Check pods on failed node
kubectl get pods --all-namespaces -o wide | grep <node-name>
# Check if critical services affected
kubectl get pods -n production -o wide | grep <node-name>
```

**3. Recovery Actions:**
- **If transient issue:** Node will auto-recover
- **If instance terminated:** Cloud provider auto-replaces it
- **Manual intervention:**
```bash
# Cordon node (prevent new pods)
kubectl cordon <node-name>
# Drain node (move pods to other nodes)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
# Uncordon after recovery
kubectl uncordon <node-name>
```

**4. Pod Rescheduling:**
- Kubernetes automatically reschedules pods to healthy nodes
- StatefulSets with persistent volumes need manual intervention
- Verify pods are running after rescheduling

**5. High Availability Setup:**
- Multiple worker nodes across AZs
- Pod anti-affinity rules
- PDBs to prevent complete outage
- Cluster autoscaler for automatic replacement

---

## 11. How do you configure a VPC for high availability?

**High Availability VPC Architecture:**

**1. Multi-AZ Setup:**
```
VPC: 10.0.0.0/16
├── us-east-1a
│   ├── Public Subnet: 10.0.1.0/24
│   ├── Private Subnet (App): 10.0.10.0/24
│   └── Private Subnet (DB): 10.0.20.0/24
├── us-east-1b
│   ├── Public Subnet: 10.0.2.0/24
│   ├── Private Subnet (App): 10.0.11.0/24
│   └── Private Subnet (DB): 10.0.21.0/24
└── us-east-1c
    ├── Public Subnet: 10.0.3.0/24
    ├── Private Subnet (App): 10.0.12.0/24
    └── Private Subnet (DB): 10.0.22.0/24
```

**2. Terraform Configuration:**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "production-vpc"
  }
}

# Public subnets
resource "aws_subnet" "public" {
  count                   = 3
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = element(["us-east-1a", "us-east-1b", "us-east-1c"], count.index)
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

# Private subnets
resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = element(["us-east-1a", "us-east-1b", "us-east-1c"], count.index)

  tags = {
    Name = "private-subnet-${count.index + 1}"
  }
}

# NAT Gateway for each AZ
resource "aws_nat_gateway" "nat" {
  count         = 3
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}

# Route tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource "aws_route_table" "private" {
  count  = 3
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat[count.index].id
  }
}

# Load balancer across AZs
resource "aws_lb" "main" {
  internal           = false
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id
}

# RDS Multi-AZ
resource "aws_db_instance" "main" {
  multi_az            = true
  db_subnet_group_name = aws_db_subnet_group.main.name
}
```

**Key HA Components:**
1. **Multiple AZs:** 3 AZs for redundancy
2. **NAT Gateway per AZ:** Internet access for private subnets even if one AZ fails
3. **Load Balancers:** Spread across public subnets in all AZs
4. **Auto Scaling Groups:** Span multiple AZs
5. **RDS Multi-AZ:** Automated failover
6. **VPC Endpoints:** For S3, DynamoDB access without internet
7. **Transit Gateway:** For connecting to other VPCs/on-prem

---

## 12. Have you worked on scripting? If so which tool? Explain what you have implemented?

**Yes, extensively. Tools I've used:**

**1. Bash Scripting:**
```bash
#!/bin/bash
# AWS EKS cluster backup script
# Backs up all Kubernetes resources and etcd

set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/eks-${TIMESTAMP}"

echo "Starting EKS backup..."

# Backup all Kubernetes resources
kubectl get all --all-namespaces -o yaml > "${BACKUP_DIR}/all-resources.yaml"

# Backup ConfigMaps and Secrets
kubectl get configmaps --all-namespaces -o yaml > "${BACKUP_DIR}/configmaps.yaml"
kubectl get secrets --all-namespaces -o yaml > "${BACKUP_DIR}/secrets.yaml"

# Backup PVCs
kubectl get pvc --all-namespaces -o yaml > "${BACKUP_DIR}/pvcs.yaml"

# Upload to S3
aws s3 sync "${BACKUP_DIR}" "s3://my-backup-bucket/eks/${TIMESTAMP}/"

echo "Backup completed: ${BACKUP_DIR}"
```

**2. Python Scripts:**
```python
#!/usr/bin/env python3
# Auto-remediation script for unhealthy pods
import subprocess
import json
from datetime import datetime

def get_unhealthy_pods():
    """Get pods that are not running or in CrashLoopBackOff"""
    cmd = "kubectl get pods --all-namespaces -o json"
    output = subprocess.check_output(cmd.split())
    pods = json.loads(output)
    
    unhealthy = []
    for pod in pods['items']:
        if pod['status']['phase'] != 'Running':
            unhealthy.append(pod)
        elif pod['status']['containerStatuses']:
            for container in pod['status']['containerStatuses']:
                if container['restartCount'] > 5:
                    unhealthy.append(pod)
    return unhealthy

def restart_pod(namespace, pod_name):
    """Restart a pod by deleting it"""
    cmd = f"kubectl delete pod {pod_name} -n {namespace}"
    subprocess.run(cmd.split())
    print(f"Restarted {pod_name} in {namespace}")

def main():
    unhealthy_pods = get_unhealthy_pods()
    for pod in unhealthy_pods:
        namespace = pod['metadata']['namespace']
        pod_name = pod['metadata']['name']
        print(f"Found unhealthy pod: {namespace}/{pod_name}")
        restart_pod(namespace, pod_name)

if __name__ == "__main__":
    main()
```

**3. PowerShell (Windows environments):**
```powershell
# IIS log rotation script
$logPath = "C:\inetpub\logs\LogFiles"
$retentionDays = 30

Get-ChildItem -Path $logPath -Recurse -File |
    Where-Object {$_.LastWriteTime -lt (Get-Date).AddDays(-$retentionDays)} |
    Remove-Item -Verbose

Write-Host "Log rotation completed"
```

**Real-world implementations:**
- Automated backup scripts for databases
- Log rotation and cleanup scripts
- Auto-remediation scripts for common issues
- Deployment automation scripts
- Infrastructure validation scripts
- Cost optimization scripts (stopping dev environments at night)

---

## 13. Have you written Terraform code for deployments? If yes, can you explain the implementation?

**Yes, extensively. Here's a real implementation:**

**Complete EKS Cluster with Application Deployment:**

```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.0"
    }
  }
  
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/eks/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# VPC Module
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "eks-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = false
  enable_vpn_gateway = false

  tags = {
    Environment = "production"
    Terraform   = "true"
  }
}

# EKS Cluster
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  version = "19.0.0"

  cluster_name    = "production-eks"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access = true
  cluster_endpoint_private_access = true

  node_groups = {
    general = {
      desired_capacity = 3
      min_capacity     = 1
      max_capacity     = 10

      instance_types = ["t3.large"]
      capacity_type  = "ON_DEMAND"

      k8s_labels = {
        Environment = "production"
        NodeGroup   = "general"
      }
    }
    
    spot = {
      desired_capacity = 2
      min_capacity     = 0
      max_capacity     = 5

      instance_types = ["t3.medium"]
      capacity_type  = "SPOT"

      k8s_labels = {
        Environment = "production"
        NodeGroup   = "spot"
      }
      
      taints = [{
        key    = "spot"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }
  }
}

# Application Deployment with Helm
resource "helm_release" "nginx_ingress" {
  name       = "nginx-ingress"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"
  namespace  = "ingress-nginx"
  create_namespace = true

  set {
    name  = "controller.replicaCount"
    value = "3"
  }
  
  set {
    name  = "controller.service.type"
    value = "LoadBalancer"
  }
}

# Application Deployment with Kubernetes Provider
resource "kubernetes_deployment" "my_app" {
  metadata {
    name      = "my-app"
    namespace = "production"
    labels = {
      app = "my-app"
    }
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "my-app"
      }
    }

    template {
      metadata {
        labels = {
          app = "my-app"
        }
      }

      spec {
        container {
          image = "my-app:latest"
          name  = "my-app"

          port {
            container_port = 8080
          }

          resources {
            limits = {
              cpu    = "500m"
              memory = "512Mi"
            }
            requests = {
              cpu    = "250m"
              memory = "256Mi"
            }
          }

          readiness_probe {
            http_get {
              path = "/health"
              port = 8080
            }
            initial_delay_seconds = 10
            period_seconds       = 5
          }

          liveness_probe {
            http_get {
              path = "/health"
              port = 8080
            }
            initial_delay_seconds = 30
            period_seconds       = 10
          }
        }
      }
    }
  }
}

# RDS Database
resource "aws_db_instance" "main" {
  identifier     = "production-db"
  engine         = "postgres"
  engine_version = "15.3"
  instance_class = "db.t3.medium"
  
  allocated_storage     = 100
  max_allocated_storage = 1000
  storage_type          = "gp3"

  multi_az               = true
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period = 30
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"

  auto_minor_version_upgrade = true
  deletion_protection        = true
  skip_final_snapshot       = false
  final_snapshot_identifier = "production-db-final"

  performance_insights_enabled = true
  monitoring_interval          = 60
}
```

**Deployment Workflow:**
1. **Initialize:** `terraform init` (downloads providers)
2. **Plan:** `terraform plan` (preview changes)
3. **Apply:** `terraform apply` (deploy infrastructure)
4. **State Management:** State stored in S3 with DynamoDB locking
5. **Workspaces:** Separate environments (dev/staging/prod)

---

## 14. Can you explain your hands-on experience with Docker?

**Docker Experience:**

**1. Image Creation:**
```dockerfile
# Multi-stage build for Node.js app
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**2. Docker Compose for Local Development:**
```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
      - REDIS_HOST=redis
    depends_on:
      - db
      - redis
    volumes:
      - ./src:/app/src

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=secret
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

**3. Image Optimization:**
```bash
# Check image size
docker images

# Optimize by:
# 1. Using multi-stage builds
# 2. Using alpine-based images
# 3. Combining RUN commands
# 4. Using .dockerignore file
# 5. Cleaning up package managers

# Example optimized Dockerfile
FROM python:3.11-alpine
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    rm -rf /var/cache/apk/*
COPY . .
```

**4. Container Management:**
```bash
# Run container
docker run -d --name my-app -p 8080:8080 my-app:latest

# View logs
docker
