# Amazon

**Exp--7 years ( DevOps Consultant)**

- If you’re migrating a monolithic application from on-prem to Cloud and the system has its local file system, which file system you will use in AWS.,
- How will you store all the configurations related to your monolithic app in Cloud,
- What are the observibility needed for app —> Monitoring, Alerting, Logging, Remediation, PD,
- If you’re not allowed to install Filebeat in your worker nodes for logging, then what will be possible option,
- What are the security protocols will be taken into consideration while designing three tier architecture.,
- DB migration, how will you sync the data,
- If DB POD is down, will it affect the data it gets stored,
- How about the sticky session data if POD gets down → Yes the session data will be lost,
- What service to use sticky session as an alternative option → redis,
- During a sticky session problem what will be the cause for LB.,
- Do you create clusters in multi regions, is it possible? If yes, then how will you manage them,
- Onboarded trading app into AWS, how will you make sure availability, scalability, security,
- How will you take a back of your entire cluster regulary ?,
- Write a python script to list the EC2 instances running in your cloud which has the tag of PROD,
- You have been tasked to create 20 EC2 per account and you been provided with 10 AWS accounts, so totally you need to create 200 EC2 machines, how will connect all these machines. Which service will be used?,
- You need to connect your DB running in private subnet, not using NAT gateway or NAT instance or bastion host, what are the other options,
- Explain about OSI model,
- Diff between directory & mount,
- Diff between local and variable in Terraform,
- You created couple of resources using Terraform, how will you make sure that resources are not modified through UI, how will you automate this check





--
Here are direct, interview-ready answers with accurate technical detail.

---

## 1. File System for Monolithic App Migration to AWS

### Recommended AWS File Systems:

**1. Amazon EFS (Elastic File System) - Most Common:**
- NFS-based, POSIX-compliant
- Shared across multiple EC2 instances
- Auto-scaling storage
- Pay per use

```hcl
resource "aws_efs_file_system" "app" {
  creation_token = "monolithic-app"
  
  tags = {
    Name = "app-efs"
  }
}

resource "aws_efs_mount_target" "app" {
  count           = 3
  file_system_id  = aws_efs_file_system.app.id
  subnet_id       = aws_subnet.private[count.index].id
  security_groups = [aws_security_group.efs.id]
}
```

**2. Amazon FSx:**
- FSx for Windows File Server (Windows apps)
- FSx for Lustre (High performance computing)
- FSx for NetApp ONTAP (Enterprise apps)

**3. Amazon EBS (Single Instance):**
- Block storage for single EC2
- High performance
- Not shared

**Mount Example:**
```bash
# EFS Mount
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 fs-12345678.efs.us-east-1.amazonaws.com:/ /mnt/efs
```

**Choice Based on Requirements:**
- **Shared across instances:** EFS
- **Single instance, high IOPS:** EBS gp3/io2
- **Windows compatibility:** FSx for Windows
- **High-performance computing:** FSx for Lustre

---

## 2. Storing Monolithic App Configurations in Cloud

### Options:

**1. AWS Systems Manager Parameter Store:**
```bash
# Store configuration
aws ssm put-parameter \
  --name "/app/database/host" \
  --value "db.example.com" \
  --type "String"

aws ssm put-parameter \
  --name "/app/database/password" \
  --value "secret123" \
  --type "SecureString"

# Retrieve in app
aws ssm get-parameter \
  --name "/app/database/host" \
  --with-decryption
```

**2. AWS Secrets Manager (For Secrets):**
```bash
# Store secret
aws secretsmanager create-secret \
  --name "app/db-credentials" \
  --secret-string '{"username":"admin","password":"secret123"}'

# Retrieve secret
aws secretsmanager get-secret-value \
  --secret-id "app/db-credentials"
```

**3. AWS AppConfig:**
```hcl
resource "aws_appconfig_application" "app" {
  name = "monolithic-app"
}

resource "aws_appconfig_configuration_profile" "app" {
  application_id = aws_appconfig_application.app.id
  name           = "production"
  location_uri   = "hosted"
}
```

**4. Environment Variables in EC2:**
```hcl
resource "aws_instance" "app" {
  # ...
  user_data = <<-EOF
    #!/bin/bash
    echo "export DB_HOST=$(aws ssm get-parameter --name /app/database/host --query Parameter.Value --output text)" >> /etc/environment
    echo "export DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id app/db-credentials --query SecretString --output text)" >> /etc/environment
  EOF
}
```

**Best Practice:**
- Use Parameter Store for configuration
- Use Secrets Manager for credentials
- Use AppConfig for dynamic configuration
- Combine with IAM roles for access control

---

## 3. Observability for Application

### Complete Observability Stack:

**1. Monitoring:**
```yaml
# Prometheus Configuration
global:
  scrape_interval: 15s

scrape_configs:
- job_name: 'application'
  static_configs:
  - targets: ['app:8080']
  metrics_path: '/metrics'
```

**2. Alerting:**
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
      summary: "Error rate above 5%"
      
  - alert: HighLatency
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
    for: 10m
    labels:
      severity: warning
```

**3. Logging:**
```yaml
# Fluentd Configuration
<source>
  @type tail
  path /var/log/app/*.log
  tag application.*
</source>

<match application.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
</match>
```

**4. Remediation:**
```python
#!/usr/bin/env python3
# Auto-remediation script
import subprocess
import requests

def check_health():
    """Check application health"""
    try:
        response = requests.get('http://localhost:8080/health')
        return response.status_code == 200
    except:
        return False

def restart_service():
    """Restart application service"""
    subprocess.run(['systemctl', 'restart', 'myapp'])

if __name__ == "__main__":
    if not check_health():
        print("Application unhealthy, restarting...")
        restart_service()
```

**5. PagerDuty Integration:**
```yaml
# Alertmanager Configuration
receivers:
- name: 'pagerduty'
  pagerduty_configs:
  - service_key: '<pagerduty-service-key>'
    severity: 'critical'
```

---

## 4. Logging Without Filebeat on Worker Nodes

### Alternative Logging Options:

**1. Sidecar Container Pattern:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logging
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      
  - name: log-shipper  # Sidecar
    image: fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
    configMap:
      name: fluentd-config
  
  volumes:
  - name: logs
    emptyDir: {}
```

**2. Kubernetes Native Logging:**
```bash
# Check logs directly from API server
kubectl logs pod-name -n namespace

# Stream logs to external system
kubectl logs -f pod-name | aws s3 cp - s3://logs-bucket/

# Use kubectl with output
kubectl logs pod-name --since=1h > /tmp/logs.txt
```

**3. Application-Level Logging:**
```python
# Python application sending logs directly
import logging
import boto3

class S3LogHandler(logging.Handler):
    def __init__(self, bucket, prefix):
        self.s3 = boto3.client('s3')
        self.bucket = bucket
        self.prefix = prefix
        
    def emit(self, record):
        log_entry = self.format(record)
        key = f"{self.prefix}/{datetime.now():%Y%m%d_%H%M%S}.log"
        self.s3.put_object(Bucket=self.bucket, Key=key, Body=log_entry)
```

**4. DaemonSet Alternative - Use Existing Log Files:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logging-script
data:
  collect.sh: |
    #!/bin/bash
    while true; do
      kubectl logs --all-pods --all-namespaces --since=5m >> /var/log/k8s.log
      sleep 300
    done
```

**5. Cloud Provider Native Solutions:**
```yaml
# AWS CloudWatch Logs with Container Insights
apiVersion: v1
kind: ConfigMap
metadata:
  name: container-insights-config
  namespace: amazon-cloudwatch
data:
  container-insights.json: |
    {
      "metrics": {
        "namespace": "ContainerInsights"
      },
      "logs": {
        "collect": true
      }
    }
```

---

## 5. Security Protocols for Three-Tier Architecture

### Security Implementation:

**1. Network Security:**
```hcl
# VPC with isolated subnets
resource "aws_subnet" "web" {
  cidr_block = "10.0.1.0/24"
  tags = { Tier = "web" }
}

resource "aws_subnet" "app" {
  cidr_block = "10.0.2.0/24"
  tags = { Tier = "app" }
}

resource "aws_subnet" "db" {
  cidr_block = "10.0.3.0/24"
  tags = { Tier = "db" }
}

# Security Groups
resource "aws_security_group" "web" {
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "app" {
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
  }
}

resource "aws_security_group" "db" {
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}
```

**2. Encryption:**
```hcl
# EBS encryption
resource "aws_ebs_encryption_by_default" "enabled" {
  enabled = true
}

# RDS encryption
resource "aws_db_instance" "main" {
  storage_encrypted = true
  kms_key_id       = aws_kms_key.rds.arn
}

# TLS/SSL
resource "aws_acm_certificate" "app" {
  domain_name = "app.example.com"
  validation_method = "DNS"
}
```

**3. IAM and Access Control:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::app-bucket/*"
      ],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/16"
        }
      }
    }
  ]
}
```

**4. Monitoring and Auditing:**
```hcl
# CloudTrail for API logging
resource "aws_cloudtrail" "main" {
  name           = "app-trail"
  s3_bucket_name = aws_s3_bucket.trail.id
  
  event_selector {
    read_write_type           = "All"
    include_management_events = true
  }
}

# Config Rules
resource "aws_config_config_rule" "s3_encryption" {
  name = "s3-bucket-encryption"
  
  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }
}
```

---

## 6. Database Migration and Data Sync

### Migration Strategies:

**1. AWS DMS (Database Migration Service):**
```bash
# Create replication instance
aws dms create-replication-instance \
  --replication-instance-identifier dms-instance \
  --replication-instance-class dms.t3.medium

# Create source endpoint
aws dms create-endpoint \
  --endpoint-identifier source-db \
  --endpoint-type source \
  --engine-name postgres \
  --server-name onprem-db.example.com \
  --port 5432 \
  --username admin \
  --password secret

# Create target endpoint
aws dms create-endpoint \
  --endpoint-identifier target-db \
  --endpoint-type target \
  --engine-name postgres \
  --server-name rds-db.xxxxx.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --username admin \
  --password secret

# Create replication task
aws dms create-replication-task \
  --replication-task-identifier migrate-db \
  --source-endpoint-arn arn:aws:dms:...:endpoint/source-db \
  --target-endpoint-arn arn:aws:dms:...:endpoint/target-db \
  --replication-instance-arn arn:aws:dms:...:rep/dms-instance \
  --migration-type full-load-and-cdc
```

**2. Native Database Replication:**
```sql
-- PostgreSQL streaming replication
-- On primary
SELECT pg_create_physical_replication_slot('replica_slot');

-- Configure standby
primary_conninfo = 'host=primary port=5432 user=replicator password=secret'
primary_slot_name = 'replica_slot'
recovery_target_timeline = 'latest'
```

**3. Continuous Sync with CDC:**
```python
# Python script for data sync
import psycopg2
import boto3

def sync_data():
    # Connect to source
    source_conn = psycopg2.connect(host="onprem-db", database="app")
    
    # Connect to target
    target_conn = psycopg2.connect(host="rds-db", database="app")
    
    # Get last synced record
    last_id = get_last_synced_id()
    
    # Get new records
    cursor = source_conn.cursor()
    cursor.execute(f"SELECT * FROM users WHERE id > {last_id}")
    new_records = cursor.fetchall()
    
    # Insert into target
    for record in new_records:
        target_conn.cursor().execute(
            "INSERT INTO users VALUES (%s, %s, %s)",
            record
        )
    
    target_conn.commit()
```

---

## 7. Database POD Down - Data Impact

### Impact on Data:

**If using EmptyDir (Ephemeral):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db
spec:
  containers:
  - name: postgres
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    emptyDir: {}  # Data LOST when pod restarts
```

**If using PersistentVolume (Recommended):**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: "db"
  replicas: 3
  template:
    spec:
      containers:
      - name: postgres
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

**Data Protection Strategies:**
1. **PersistentVolumes:** Data survives pod restarts
2. **Replication:** Multiple copies across nodes
3. **Backups:** Regular snapshots
4. **StatefulSets:** Stable storage identity

---

## 8. Sticky Session and Redis Solution

### Sticky Session Problem:

**Issue:** Session data lost when pod restarts

**Solution 1: Redis for Session Storage:**
```python
# Python Flask with Redis sessions
from flask import Flask, session
from flask_session import Session
import redis

app = Flask(__name__)
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_REDIS'] = redis.Redis(host='redis-service', port=6379)
Session(app)

@app.route('/login')
def login():
    session['user_id'] = 123
    return "Logged in"

@app.route('/profile')
def profile():
    user_id = session.get('user_id')
    return f"User: {user_id}"
```

**Solution 2: Redis Deployment in K8s:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

**Load Balancer Configuration for Sticky Sessions:**
```yaml
# AWS ALB with sticky sessions
apiVersion: networking.k8s.io/v1
kind: Service
metadata:
  name: app-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-sticky-sessions: "true"
    service.beta.kubernetes.io/aws-load-balancer-sticky-sessions-type: "lb_cookie"
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

---

## 9. Load Balancer Sticky Session Issues

### Common Causes:

**1. Cookie Issues:**
- Cookie not set properly
- Cookie expired
- Client not accepting cookies

**2. Configuration Problems:**
```yaml
# Wrong LB configuration
annotations:
  service.beta.kubernetes.io/aws-load-balancer-sticky-sessions: "false"  # Should be true
  # Wrong cookie type
  service.beta.kubernetes.io/aws-load-balancer-sticky-sessions-type: "app_cookie"  # Wrong
```

**3. Backend Issues:**
- Application overwriting cookies
- Session timeout in application
- Multiple containers not sharing session

**4. LB Health Check Issues:**
- Instances failing health checks
- LB routing to different instances

**Fix:**
```yaml
# Correct configuration
apiVersion: v1
kind: Service
metadata:
  name: app
  annotations:
    alb.ingress.kubernetes.io/sticky-sessions: "true"
    alb.ingress.kubernetes.io/sticky-sessions-type: "lb_cookie"
    alb.ingress.kubernetes.io/session-cookie-name: "sessionid"
    alb.ingress.kubernetes.io/session-cookie-max-age: "3600"
spec:
  type: LoadBalancer
```

---

## 10. Multi-Region Clusters

### Yes, Multi-Region Clusters are Possible

**Management Options:**

**1. Separate Clusters per Region (Recommended):**
```yaml
# Region 1 (us-east-1)
apiVersion: v1
kind: Config
clusters:
- name: prod-us-east
  cluster:
    server: https://eks-us-east.example.com
    
# Region 2 (us-west-2)
- name: prod-us-west
  cluster:
    server: https://eks-us-west.example.com
```

**2. Cluster Management Tools:**
```bash
# Rancher - Multi-cluster management
# Anthos - Google's multi-cluster
# ArgoCD - GitOps for multiple clusters

# ArgoCD multi-cluster
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  destination:
    server: https://eks-us-east.example.com
    namespace: production
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-west
spec:
  destination:
    server: https://eks-us-west.example.com
    namespace: production
```

**3. Traffic Management:**
```yaml
# Route 53 for multi-region failover
resource "aws_route53_record" "app" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  set_identifier = "us-east"
  alias {
    name                   = aws_lb.us_east.dns_name
    zone_id                = aws_lb.us_east.zone_id
    evaluate_target_health = true
  }
}
```

---

## 11. Trading App - Availability, Scalability, Security

### High Availability Architecture:

**1. Availability:**
```hcl
# Multi-AZ deployment
resource "aws_autoscaling_group" "trading" {
  vpc_zone_identifier = [
    aws_subnet.az1.id,
    aws_subnet.az2.id,
    aws_subnet.az3.id
  ]
  
  min_size = 3
  max_size = 20
  desired_capacity = 6
  
  health_check_type = "ELB"
  health_check_grace_period = 300
}
```

**2. Scalability:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: trading-app-hpa
spec:
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        averageUtilization: 60
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        averageValue: 1000
```

**3. Security:**
```hcl
# WAF for web protection
resource "aws_wafv2_web_acl" "trading" {
  name = "trading-waf"
  
  rule {
    name     = "rate-limit"
    priority = 0
    action {
      block {}
    }
    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }
  }
}

# KMS for encryption
resource "aws_kms_key" "trading" {
  enable_key_rotation = true
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "Enable IAM User Permissions"
      Effect = "Allow"
      Principal = { AWS = "arn:aws:iam::123456789012:root" }
      Action = "kms:*"
      Resource = "*"
    }]
  })
}
```

---

## 12. Regular Cluster Backup

### Backup Strategies:

**1. etcd Backup:**
```bash
#!/bin/bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-*.db
```

**2. Velero for Complete Cluster Backup:**
```bash
# Install Velero
velero install \
  --provider aws \
  --bucket velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1

# Create backup
velero backup create full-backup --include-namespaces production

# Schedule regular backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production \
  --ttl 720h0m0s
```

**3. Resource Export:**
```python
#!/usr/bin/env python3
# Export all K8s resources
import subprocess
import json

def backup_all_resources():
    resources = [
        'deployments', 'services', 'configmaps', 'secrets',
        'ingresses', 'statefulsets', 'pvc', 'daemonsets'
    ]
    
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    
    for resource in resources:
        cmd = f"kubectl get {resource} --all-namespaces -o yaml"
        output = subprocess.check_output(cmd, shell=True)
        
        filename = f"/backup/{resource}_{timestamp}.yaml"
        with open(filename, 'wb') as f:
            f.write(output)
```

---

## 13. Python Script to List EC2 Instances with PROD Tag

```python
#!/usr/bin/env python3
"""
List EC2 instances with PROD tag
"""

import boto3
from datetime import datetime

def list_prod_instances():
    """List all EC2 instances with PROD tag"""
    
    # Create EC2 client
    ec2 = boto3.client('ec2')
    
    try:
        # Describe instances with filter
        response = ec2.describe_instances(
            Filters=[
                {
                    'Name': 'tag:Environment',
                    'Values': ['PROD', 'Prod', 'prod']
                },
                {
                    'Name': 'instance-state-name',
                    'Values': ['running']
                }
            ]
        )
        
        # Parse instances
        instances = []
        for reservation in response['Reservations']:
            for instance in reservation['Instances']:
                instance_info = {
                    'InstanceId': instance['InstanceId'],
                    'InstanceType': instance['InstanceType'],
                    'State': instance['State']['Name'],
                    'LaunchTime': instance['LaunchTime'].strftime('%Y-%m-%d %H:%M:%S'),
                    'PrivateIp': instance.get('PrivateIpAddress', 'N/A'),
                    'PublicIp': instance.get('PublicIpAddress', 'N/A'),
                    'Name': next(
                        (tag['Value'] for tag in instance.get('Tags', []) 
                         if tag['Key'] == 'Name'),
                        'Unnamed'
                    )
                }
                instances.append(instance_info)
        
        return instances
        
    except Exception as e:
        print(f"Error listing instances: {e}")
        return []

def print_instances(instances):
    """Print instance details"""
    print(f"\n{'='*80}")
    print(f"PROD EC2 Instances - {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"{'='*80}")
    print(f"{'Instance ID':<20} {'Name':<20} {'Type':<15} {'State':<10} {'Private IP':<15}")
    print(f"{'-'*80}")
    
    for instance in instances:
        print(f"{instance['InstanceId']:<20} "
              f"{instance['Name']:<20} "
              f"{instance['InstanceType']:<15} "
              f"{instance['State']:<10} "
              f"{instance['PrivateIp']:<15}")
    
    print(f"{'='*80}")
    print(f"Total: {len(instances)} instances")

def export_to_csv(instances):
    """Export instances to CSV"""
    import csv
    
    filename = f"prod_instances_{datetime.now().strftime('%Y%m%d')}.csv"
    
    with open(filename, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=[
            'InstanceId', 'Name', 'InstanceType', 'State', 
            'PrivateIp', 'PublicIp', 'LaunchTime'
        ])
        writer.writeheader()
        writer.writerows(instances)
    
    print(f"\nExported to {filename}")

if __name__ == "__main__":
    instances = list_prod_instances()
    print_instances(instances)
    
    # Optional: Export to CSV
    if instances:
        export_to_csv(instances)
```

---

## 14. Managing 200 EC2 Instances Across 10 Accounts

### AWS Organizations + Systems Manager

**Architecture:**
```
AWS Organizations
├── Management Account
├── Account 1 (10 EC2)
├── Account 2 (10 EC2)
├── ...
└── Account 10 (10 EC2)
```

### Step 1: Set Up AWS Organizations
```bash
# Create organization
aws organizations create-organization

# Create accounts
aws organizations create-account \
  --email admin1@example.com \
  --account-name "Team1-Account"

# Or use Terraform
resource "aws_organizations_account" "team1" {
  name  = "team1-account"
  email = "team1@example.com"
}
```

### Step 2: AWS Systems Manager for Management
```hcl
# Enable Systems Manager
resource "aws_ssm_activation" "main" {
  name               = "ec2-activation"
  iam_role           = aws_iam_role.ssm.arn
  registration_limit = 200
}
```

### Step 3: Install SSM Agent on EC2
```bash
# User data script
#!/bin/bash
# Install SSM Agent
cd /tmp
wget https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
sudo rpm -Uvh amazon-ssm-agent.rpm
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### Step 4: Manage All Instances
```python
#!/usr/bin/env python3
# Manage instances across accounts
import boto3

def manage_all_instances(command):
    """Execute command on all instances"""
    
    accounts = [
        {'name': 'account1', 'profile': 'acct1'},
        {'name': 'account2', 'profile': 'acct2'},
        # ... all 10 accounts
    ]
    
    for account in accounts:
        session = boto3.Session(profile_name=account['profile'])
        ssm = session.client('ssm')
        
        # Send command to all instances
        response = ssm.send_command(
            InstanceIds=['*'],  # All instances
            DocumentName='AWS-RunShellScript',
            Parameters={'commands': [command]}
        )
        print(f"Command sent to {account['name']}: {response['Command']['CommandId']}")
```

### Step 5: Monitoring with CloudWatch
```hcl
# CloudWatch Dashboard for all instances
resource "aws_cloudwatch_dashboard" "all_instances" {
  dashboard_name = "all-ec2-instances"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/EC2", "CPUUtilization", { "stat": "Average" }]
          ]
          period = 300
          stat   = "Average"
          region = "us-east-1"
          title  = "CPU Utilization"
        }
      }
    ]
  })
}
```

---

## 15. Connect to DB in Private Subnet Without NAT/Bastion

### Alternative Options:

**1. VPC Endpoints (AWS PrivateLink):**
```hcl
resource "aws_vpc_endpoint" "db" {
  vpc_id             = aws_vpc.main.id
  service_name       = "com.amazonaws.us-east-1.rds"
  vpc_endpoint_type  = "Interface"
  subnet_ids         = [aws_subnet.private.id]
  security_group_ids = [aws_security_group.endpoint.id]
  
  private_dns_enabled = true
}
```

**2. AWS Direct Connect:**
```bash
# Direct connection from on-prem to VPC
aws directconnect create-connection \
  --connection-name "onprem-to-aws" \
  --bandwidth "1Gbps" \
  --location "EqDC2"
```

**3. VPN Connection:**
```hcl
resource "aws_vpn_connection" "main" {
  vpn_gateway_id      = aws_vpn_gateway.main.id
  customer_gateway_id = aws_customer_gateway.main.id
  type                = "ipsec.1"
  
  static_routes_only = true
}
```

**4. AWS Site-to-Site VPN:**
```bash
# Create site-to-site VPN
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-12345678 \
  --vpn-gateway-id vgw-12345678
```

**5. Database Proxy (RDS Proxy):**
```hcl
resource "aws_db_proxy" "main" {
  name                   = "db-proxy"
  engine_family          = "POSTGRESQL"
  role_arn              = aws_iam_role.proxy.arn
  vpc_subnet_ids        = [aws_subnet.private.id]
  
  auth {
    auth_scheme = "SECRETS"
    secret_arn  = aws_secretsmanager_secret.db.arn
  }
}
```

---

## 16. OSI Model Explanation

### 7 Layers of OSI Model:

**Layer 7 - Application Layer:**
- Provides network services to applications
- Protocols: HTTP, HTTPS, FTP, SMTP, DNS
- Example: Web browser requesting a page

**Layer 6 - Presentation Layer:**
- Data formatting, encryption, compression
- Protocols: SSL/TLS, JPEG, ASCII
- Example: HTTPS encryption

**Layer 5 - Session Layer:**
- Establishes, manages, terminates sessions
- Protocols: NetBIOS, RPC
- Example: Login session

**Layer 4 - Transport Layer:**
- End-to-end delivery, reliability, flow control
- Protocols: TCP, UDP
- Example: TCP ensures data delivery

**Layer 3 - Network Layer:**
- Routing, logical addressing
- Protocols: IP, ICMP, OSPF
- Example: Router forwarding packets

**Layer 2 - Data Link Layer:**
- Physical addressing, error detection
- Protocols: Ethernet, MAC addresses
- Example: Switch forwarding frames

**Layer 1 - Physical Layer:**
- Raw bit transmission
- Cables, connectors, electrical signals
- Example: Network cable

**Data Flow:**
```
Application → Presentation → Session → Transport → Network → Data Link → Physical
(Data)      (Data)         (Data)    (Segment)  (Packet)  (Frame)    (Bits)
```

---

## 17. Difference Between Directory and Mount

### Directory:
- A folder in filesystem
- Contains files and subdirectories
- Created with `mkdir` command
- Part of existing filesystem

```bash
# Create directory
mkdir /data
# This creates a folder in the current filesystem
```

### Mount:
- Attaches a filesystem to a directory
- The directory becomes mount point
- Created with `mount` command
- Represents separate filesystem

```bash
# Mount filesystem
mount /dev/sdb1 /data
# Now /data shows content of /dev/sdb1
```

### Key Differences:

| Directory | Mount |
|-----------|-------|
| Logical container | Filesystem attachment |
| Simple creation | Requires filesystem |
| Part of parent FS | Separate FS |
| `mkdir` command | `mount` command |
| Always available | Can be mounted/unmounted |

### Example:
```bash
# 1. Create directory
mkdir /mnt/data

# 2. Directory is empty part of root filesystem
ls -la /mnt/data
# total 0

# 3. Mount filesystem
mount /dev/sdb1 /mnt/data

# 4. Now shows content of /dev/sdb1
ls -la /mnt/data
# Shows files from mounted device
```

---

## 18. Local vs Variable in Terraform

### Local Values:
- Computed values used within module
- Not exposed to external modules
- Can reference other locals, variables, resources

```hcl
locals {
  environment = "production"
  instance_name = "${local.environment}-web-server"
  
  common_tags = {
    Environment = local.environment
    ManagedBy
