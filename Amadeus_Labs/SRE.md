# Amadeus Labs

**Exp-5yr SRE**

- How to fix pod-level autoscaling not happening, what will be your approach .
- suppose we have application configured with hpa where it was running fine but suddenly it is not running what will be your approach?
- How to access application if Ingress is configured but not accessible to end users, how you will resolve ?
- How Kubernetes handles service discovery can you explain .
- How to ensure all five application teams don't use more than a particular amount of space?
- Leap second in Linux
- what type of autmation you have done in k8
- Tell what type of aumatation you have done that saved time in your project .
- 11 . Can pod be scheduled if have below security constent .
- securityContext:
- runAsNonRoot: true
- runAsUser: 0



--
Here are direct, interview-ready answers with accurate technical detail.

---

## 1. How to Fix Pod-Level Autoscaling Not Happening

### Troubleshooting Approach:

**Step 1: Verify HPA Configuration**
```bash
# Check HPA status
kubectl get hpa -n <namespace>

# Output shows:
# NAME    REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS
# myapp   Deployment/myapp   <unknown>/70%   1         10        1

# If TARGETS shows <unknown>, metrics not available
```

**Step 2: Check Metrics Server**
```bash
# Verify metrics-server is running
kubectl get pods -n kube-system | grep metrics-server

# Check metrics-server logs
kubectl logs -n kube-system <metrics-server-pod>

# Test metrics API
kubectl top pods -n <namespace>
# Error: "metrics not available yet" means metrics-server issue
```

**Step 3: Verify HPA Details**
```bash
# Describe HPA
kubectl describe hpa myapp -n <namespace>

# Look for:
# - Events section
# - Conditions section
# - Warning messages
```

**Step 4: Check Resource Requests**
```bash
# HPA requires resource requests on containers
kubectl get deployment myapp -n <namespace> -o yaml | grep -A 5 resources

# If missing requests, add them:
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
```

**Step 5: Verify Metrics Availability**
```bash
# Check if metrics are being scraped
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods

# Check Prometheus adapter (if using custom metrics)
kubectl get pods -n monitoring | grep prometheus-adapter
```

**Common Fixes:**

**Fix 1: Install Metrics Server**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Or with Helm
helm install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set args[0]=--kubelet-insecure-tls
```

**Fix 2: Add Resource Requests**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
```

**Fix 3: Fix HPA Configuration**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## 2. HPA Was Working Fine but Suddenly Stopped

### Systematic Troubleshooting:

**Step 1: Check Recent Changes**
```bash
# Check HPA events
kubectl describe hpa myapp -n <namespace>

# Check recent deployments
kubectl rollout history deployment/myapp -n <namespace>

# Check configmap changes
kubectl get configmap -n <namespace> --sort-by=.metadata.creationTimestamp
```

**Step 2: Verify Metrics Flow**
```bash
# 1. Check metrics-server
kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system <metrics-server-pod> --tail=50

# 2. Test metrics API
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods

# 3. Check pod metrics
kubectl top pods -n <namespace>
```

**Step 3: Check HPA Status**
```bash
# Get HPA YAML
kubectl get hpa myapp -n <namespace> -o yaml

# Check conditions
# Look for:
# - AbleToScale: True/False
# - ScalingActive: True/False
# - ScalingLimited: True/False

# Check events
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | grep -i hpa
```

**Step 4: Common Causes and Fixes:**

**Cause 1: Metrics Server Down**
```bash
# Check status
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Restart if needed
kubectl rollout restart deployment metrics-server -n kube-system
```

**Cause 2: Resource Limits Reached**
```bash
# Check if max replicas reached
kubectl get hpa myapp -n <namespace>
# REPLICAS shows max (e.g., 10/10)

# Check cluster resources
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources"
```

**Cause 3: Metrics Values Changed**
```bash
# Check if metric threshold is too high
kubectl get hpa myapp -n <namespace> -o yaml | grep -A 5 metrics

# Check actual metrics
kubectl top pods -n <namespace>
```

**Cause 4: Label Selector Mismatch**
```bash
# Verify HPA selector matches deployment
kubectl get hpa myapp -n <namespace> -o yaml | grep -A 3 scaleTargetRef
kubectl get deployment myapp -n <namespace> -o yaml | grep -A 3 selector
```

---

## 3. Ingress Configured but Not Accessible

### Troubleshooting Ingress Issues:

**Step 1: Check Ingress Resource**
```bash
# Get ingress status
kubectl get ingress -n <namespace>

# Check if ADDRESS is populated
# NAME    CLASS   HOSTS              ADDRESS        PORTS
# myapp   nginx   app.example.com   <pending>      80

# If ADDRESS is <pending>, ingress controller issue
```

**Step 2: Describe Ingress**
```bash
kubectl describe ingress myapp -n <namespace>

# Check Events section for errors
# Check if backend service is found
```

**Step 3: Verify Ingress Controller**
```bash
# Check ingress controller pods
kubectl get pods -n ingress-nginx

# Check controller logs
kubectl logs -n ingress-nginx <ingress-controller-pod> --tail=100

# Check controller service
kubectl get svc -n ingress-nginx
```

**Step 4: Check Backend Service**
```bash
# Verify service exists
kubectl get svc -n <namespace>

# Check service endpoints
kubectl get endpoints -n <namespace>

# If endpoints are empty, check pod labels
kubectl get pods -n <namespace> --show-labels
```

**Step 5: Verify Ingress Configuration**
```yaml
# Check ingress YAML
kubectl get ingress myapp -n <namespace> -o yaml

# Verify:
# 1. Host name correct
# 2. Path correct
# 3. Service name correct
# 4. Service port correct
```

**Step 6: Test Connectivity**
```bash
# Test from inside cluster
kubectl run test-pod --rm -it --image=busybox -- /bin/sh
wget -O- http://myapp-service:8080

# Test ingress controller directly
kubectl exec -it <ingress-controller-pod> -n ingress-nginx -- curl -v http://myapp-service:8080

# Test from outside
curl -v http://app.example.com
```

**Common Fixes:**

**Fix 1: Missing Ingress Class**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    kubernetes.io/ingress.class: nginx  # Add this
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 8080
```

**Fix 2: DNS Not Configured**
```bash
# Add DNS record pointing to ingress controller IP
# Check ingress controller service IP
kubectl get svc -n ingress-nginx

# Configure DNS
app.example.com → <ingress-controller-IP>
```

**Fix 3: TLS/SSL Issues**
```bash
# Check certificate
kubectl get certificate -n <namespace>
kubectl describe certificate myapp-tls -n <namespace>

# Check secret exists
kubectl get secret myapp-tls -n <namespace>
```

---

## 4. How Kubernetes Handles Service Discovery

### Service Discovery Mechanisms:

**1. Environment Variables:**
```bash
# When pod starts, Kubernetes injects environment variables
# Example for service "myapp" in namespace "default":
MYAPP_SERVICE_HOST=10.0.0.1
MYAPP_SERVICE_PORT=8080
MYAPP_PORT=tcp://10.0.0.1:8080
MYAPP_PORT_8080_TCP=tcp://10.0.0.1:8080
MYAPP_PORT_8080_TCP_PROTO=tcp
MYAPP_PORT_8080_TCP_PORT=8080
MYAPP_PORT_8080_TCP_ADDR=10.0.0.1
```

**2. DNS (CoreDNS):**
```bash
# Service DNS format:
<service-name>.<namespace>.svc.cluster.local

# Examples:
# Same namespace: myapp
# Different namespace: myapp.default.svc.cluster.local
# Headless service: myapp.default.svc.cluster.local returns pod IPs

# From inside pod:
nslookup myapp
# Returns ClusterIP of service

nslookup myapp.default.svc.cluster.local
# Returns full DNS resolution
```

**3. ClusterIP:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP  # Default
```

**4. Headless Service (For StatefulSets):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
spec:
  clusterIP: None  # Headless
  selector:
    app: myapp
  ports:
  - port: 8080
```

**5. Service Mesh (Istio/Linkerd):**
```yaml
# Istio VirtualService for advanced routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90
    - destination:
        host: myapp
        subset: v2
      weight: 10
```

### How Service Discovery Works:

**Flow:**
1. Pod starts
2. Kubelet registers pod with API server
3. Service controller creates endpoints
4. CoreDNS updates DNS records
5. Other pods discover service via DNS or env vars
6. Kube-proxy/iptables routes traffic

**Example:**
```bash
# Create deployment
kubectl create deployment nginx --image=nginx --replicas=3

# Create service
kubectl expose deployment nginx --port=80

# Service automatically gets ClusterIP
kubectl get svc nginx
# NAME    TYPE        CLUSTER-IP     PORT(S)
# nginx   ClusterIP   10.96.0.100   80/TCP

# Pods can access via:
# 1. DNS: nginx.default.svc.cluster.local
# 2. Env vars: NGINX_SERVICE_HOST=10.96.0.100
# 3. ClusterIP: 10.96.0.100
```

---

## 5. Ensure Teams Don't Use More Than Particular Space

### Resource Quota Implementation:

**1. Namespace-Level ResourceQuota:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    requests.storage: "500Gi"
    persistentvolumeclaims: "10"
    pods: "50"
    services: "20"
    configmaps: "50"
    secrets: "50"
```

**2. Per-Team Namespace:**
```bash
# Create namespace for each team
kubectl create namespace team-a
kubectl create namespace team-b
kubectl create namespace team-c

# Apply quota to each namespace
kubectl apply -f team-quota.yaml -n team-a
```

**3. LimitRange for Default Limits:**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-limits
  namespace: team-a
spec:
  limits:
  - max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    type: Container
  - max:
      storage: "100Gi"
    min:
      storage: "1Gi"
    type: PersistentVolumeClaim
```

**4. Storage Quotas:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: team-a
spec:
  hard:
    requests.storage: "500Gi"
    persistentvolumeclaims: "20"
```

**5. Network Policies (Isolation):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: team-isolation
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          team: team-a
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          team: team-a
```

**6. Monitoring and Enforcement:**
```bash
# Check quota usage
kubectl describe resourcequota team-quota -n team-a

# Check current usage
kubectl get resourcequota team-quota -n team-a -o yaml
```

---

## 6. Leap Second in Linux

### What is Leap Second?
- Added to UTC to keep atomic time aligned with Earth's rotation
- Earth's rotation slows, so leap seconds added periodically
- Can cause system issues (CPU spikes, application errors)

### Impact on Linux Systems:
```bash
# Check if system affected
date --date='@1230768000'

# NTP handles leap seconds
# Modern kernels use leap second smearing

# Check kernel support
cat /proc/sys/kernel/time_status
```

### Managing Leap Seconds:

**1. Check Current Time:**
```bash
# Show current time with leap seconds
date
timedatectl status

# Check NTP status
chronyc tracking
ntpq -p
```

**2. Leap Second Handling:**
```bash
# Option 1: Leap second smearing (Google's approach)
# Spread leap second over 24 hours
# Configure in NTP server

# Option 2: Step insertion
# Add second instantly at midnight UTC

# Check for leap second announcements
cat /etc/leap-seconds.list
```

**3. Known Issues:**
```bash
# Java applications may hang
# Fix: -XX:+UseParallelGC -XX:-UsePerfData

# Linux kernel issues (pre-4.0)
# Fix: kernel upgrade
```

**4. Monitoring:**
```bash
# Watch for issues during leap second
dmesg | grep -i leap
journalctl -k | grep -i leap
```

---

## 7. Automation Done in Kubernetes

### Automation Examples:

**1. Auto-Scaling:**
```yaml
# HPA for application scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**2. Auto-Healing:**
```python
#!/usr/bin/env python3
# Auto-restart unhealthy pods
import subprocess
import json

def check_pod_health():
    cmd = "kubectl get pods --all-namespaces -o json"
    pods = json.loads(subprocess.check_output(cmd.split()))
    
    for pod in pods['items']:
        # Check for CrashLoopBackOff
        if pod['status'].get('containerStatuses'):
            for container in pod['status']['containerStatuses']:
                if container.get('restartCount', 0) > 10:
                    namespace = pod['metadata']['namespace']
                    name = pod['metadata']['name']
                    print(f"Restarting {namespace}/{name}")
                    subprocess.run(
                        f"kubectl delete pod {name} -n {namespace}",
                        shell=True
                    )
```

**3. Automated Backups:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-database
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h db-service > /backup/backup-$(date +%Y%m%d).sql
              aws s3 cp /backup/ s3://my-backup-bucket/ --recursive
          restartPolicy: OnFailure
```

**4. GitOps with ArgoCD:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

## 8. Automation That Saved Time

### High-Impact Automations:

**1. CI/CD Pipeline Automation:**
```groovy
// Jenkins Pipeline - Saved 4 hours per deployment
pipeline {
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps { sh 'mvn test' }
                }
                stage('Integration Tests') {
                    steps { sh 'mvn integration-test' }
                }
                stage('Code Quality') {
                    steps { sh 'mvn sonar:sonar' }
                }
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t myapp:latest .'
            }
        }
        stage('Deploy') {
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }
}
```

**2. Infrastructure Provisioning:**
```hcl
# Terraform - Saved 2 days per environment setup
module "environment" {
  source = "./modules/environment"
  
  name = "production"
  size = "large"
  region = "us-east-1"
}
```

**3. Automated Testing:**
```python
# Automated smoke tests - Saved 1 hour per release
def run_smoke_tests():
    tests = [
        test_health_endpoint(),
        test_database_connection(),
        test_redis_connection(),
        test_external_api()
    ]
    return all(tests)
```

**4. Monitoring and Alerting Automation:**
```yaml
# Prometheus Alert Rules - Reduced MTTR by 70%
groups:
- name: application
  rules:
  - alert: HighErrorRate
    expr: sum(rate(http_requests_total{status=~"5.."}[5m])) > 0.01
    annotations:
      summary: "Error rate above 1%"
```

---

## 9. Pod Scheduling with Security Context

### Given Security Context:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 0
```

### Analysis:
**This pod CANNOT be scheduled/running successfully**

**Reason:**
- `runAsNonRoot: true` requires container to run as non-root user
- `runAsUser: 0` means run as root user (UID 0)
- These are contradictory requirements

### What Happens:
```bash
# Pod will be rejected with error:
# Error: container has runAsNonRoot and image will run as root

# Or at runtime:
# Error: container has runAsNonRoot and image has non-numeric user
```

### Correct Configuration:

**Option 1: Run as Non-Root (Recommended):**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000  # Non-root user
  runAsGroup: 3000
  fsGroup: 2000
```

**Option 2: Run as Root (Not recommended):**
```yaml
securityContext:
  runAsNonRoot: false
  runAsUser: 0  # Root user
```

**Option 3: Use Container-Specific Security Context:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
```

### Validation Commands:
```bash
# Check pod security context
kubectl get pod <pod-name> -o yaml | grep -A 10 securityContext

# Check if pod is running
kubectl get pod <pod-name>
# If Pending or Error, check events
kubectl describe pod <pod-name>
```

These answers cover the key concepts and troubleshooting approaches for Kubernetes and DevOps interviews.
