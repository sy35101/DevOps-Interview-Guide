# Amazon

**Exp-9years(Devops Consulatnt)**

- Sending log files from EC2 to S3, what are the steps ?
- Limiting the resource usage in k8s not through deployment.yam → Through namespace
- Three tier architecture
- Updating worker nodes in k8s
- I’m an admin but I don’t have access to the S3 bucket ? → IAM permission boundary
- Various stages of CI/CD
- How will you build the image during CI and how will you manage it ?
- You have an S3 bucket at us-south-1, is it possible to access that bucket from us-east-1 ?
- Write a Terraform code to create VPC, subnet, EC2, S3 bucket.
- Purpose of using CNI in K8s
- I need to send a logs from EC2 to S3, create an automation part where we need to take the log file, check the CPU metrics and send an alarm through cloud watch to user's.
- Is it possible to create NAT gateway in private subnet ?
- How to configure Cluster auto-scaling, how to do that


--
Here are direct, interview-ready answers with accurate technical detail.

---

## 1. Sending Log Files from EC2 to S3

### Method 1: AWS CloudWatch Agent (Recommended)

**Step 1: Create IAM Role for EC2**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

**Step 2: Install CloudWatch Agent**
```bash
# Download and install
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb

# Or using yum (Amazon Linux/RHEL)
sudo yum install amazon-cloudwatch-agent
```

**Step 3: Configure CloudWatch Agent**
```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "logfile": "/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/application.log",
            "log_group_name": "production-application",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          },
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "nginx-access",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    }
  }
}
```

**Step 4: Start CloudWatch Agent**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
  -s
```

### Method 2: AWS CLI with Cron Job

**Step 1: Create Script**
```bash
#!/bin/bash
# /usr/local/bin/send-logs-to-s3.sh

LOG_FILE="/var/log/application.log"
S3_BUCKET="my-log-bucket"
DATE=$(date +%Y%m%d_%H%M%S)
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

# Copy log to temporary location
cp "$LOG_FILE" "/tmp/${INSTANCE_ID}_${DATE}.log"

# Upload to S3
aws s3 cp "/tmp/${INSTANCE_ID}_${DATE}.log" "s3://${S3_BUCKET}/logs/${INSTANCE_ID}/"

# Clean up
rm "/tmp/${INSTANCE_ID}_${DATE}.log"
```

**Step 2: Add to Crontab**
```bash
# Edit crontab
crontab -e

# Add line (every hour)
0 * * * * /usr/local/bin/send-logs-to-s3.sh
```

### Method 3: Using S3 Lifecycle with CloudWatch Logs

```bash
# Configure CloudWatch Logs to export to S3
aws logs create-export-task \
  --task-name "export-$(date +%Y%m%d)" \
  --log-group-name "/var/log/application" \
  --from 0 \
  --to $(date +%s%3N) \
  --destination "my-log-bucket" \
  --destination-prefix "logs/$(date +%Y%m%d)"
```

---

## 2. Limiting Resource Usage in K8s Through Namespace

### Using ResourceQuota and LimitRange:

**Step 1: Create Namespace**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    team: team-a
    environment: production
```

**Step 2: Apply ResourceQuota**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    # Compute Resources
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    
    # Storage Resources
    requests.storage: "500Gi"
    persistentvolumeclaims: "10"
    
    # Object Counts
    pods: "50"
    services: "10"
    secrets: "20"
    configmaps: "20"
    replicationcontrollers: "20"
    
    # Extended Resources
    requests.nvidia.com/gpu: "4"
    limits.nvidia.com/gpu: "4"
```

**Step 3: Apply LimitRange for Default Values**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-limits
  namespace: team-a
spec:
  limits:
  # Container limits
  - type: Container
    max:
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
    maxLimitRequestRatio:
      cpu: "4"
      memory: "4"
  
  # Pod limits
  - type: Pod
    max:
      cpu: "8"
      memory: "16Gi"
    min:
      cpu: "200m"
      memory: "256Mi"
  
  # PVC limits
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"
    min:
      storage: "1Gi"
```

**Step 4: Verify Quota Usage**
```bash
# Check quota
kubectl describe resourcequota team-a-quota -n team-a

# Check limit range
kubectl describe limitrange team-a-limits -n team-a

# Check current usage
kubectl get resourcequota team-a-quota -n team-a -o yaml
```

**Step 5: Test Enforcement**
```yaml
# This deployment will fail if quota exceeded
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: team-a
spec:
  replicas: 100  # Will fail if pod quota is 50
  template:
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: "500m"  # Will fail if CPU quota exceeded
            memory: "1Gi"
```

---

## 3. Three-Tier Architecture

### Architecture Overview:
```
┌─────────────────────────────────────────┐
│         PRESENTATION TIER              │
│  (Web Servers - NGINX, Apache)        │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         APPLICATION TIER               │
│  (App Servers - Node.js, Java)        │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         DATA TIER                      │
│  (Databases - PostgreSQL, MySQL)      │
└─────────────────────────────────────────┘
```

### Kubernetes Implementation:

**1. Presentation Tier:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    tier: presentation
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: presentation
  template:
    metadata:
      labels:
        tier: presentation
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    tier: presentation
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

**2. Application Tier:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    tier: application
spec:
  replicas: 5
  selector:
    matchLabels:
      tier: application
  template:
    metadata:
      labels:
        tier: application
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "database-service"
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    tier: application
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

**3. Data Tier:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  labels:
    tier: data
spec:
  serviceName: "database"
  replicas: 3
  selector:
    matchLabels:
      tier: data
  template:
    metadata:
      labels:
        tier: data
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
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
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
spec:
  selector:
    tier: data
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
  clusterIP: None  # Headless for StatefulSet
```

### Network Policies for Isolation:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tier-isolation
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: presentation
    ports:
    - port: 80
  - from:
    - podSelector:
        matchLabels:
          tier: application
    ports:
    - port: 8080
  - from:
    - podSelector:
        matchLabels:
          tier: data
    ports:
    - port: 5432
```

---

## 4. Updating Worker Nodes in Kubernetes

### Safe Node Update Process:

**Step 1: Check Current Version**
```bash
# Check node versions
kubectl get nodes
# NAME                  STATUS   VERSION
# node-1                Ready    v1.27.0
# node-2                Ready    v1.27.0
```

**Step 2: Cordon and Drain Node**
```bash
# Mark node as unschedulable
kubectl cordon node-1

# Drain pods from node
kubectl drain node-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60

# Check node status
kubectl get nodes
# node-1 shows SchedulingDisabled
```

**Step 3: Update Node**
```bash
# AWS EKS Example
# Create new launch template version with new AMI

# Get current launch template
aws ec2 describe-launch-template-versions \
  --launch-template-name my-node-template

# Create new version
aws ec2 create-launch-template-version \
  --launch-template-name my-node-template \
  --source-version 1 \
  --launch-template-data '{"ImageId":"ami-new-version"}'

# Terminate old node (ASG will replace with new version)
aws ec2 terminate-instances --instance-ids i-1234567890

# Or update node group
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup \
  --release-version 1.28.0
```

**Step 4: Verify New Node**
```bash
# Check new node is Ready
kubectl get nodes

# Uncordon if needed
kubectl uncordon node-1
```

**Step 5: Repeat for Other Nodes**
```bash
# For each node in cluster
for node in $(kubectl get nodes -o name | grep -v master); do
  echo "Updating $node"
  kubectl cordon $node
  kubectl drain $node --ignore-daemonsets --delete-emptydir-data
  # Update node
  kubectl uncordon $node
done
```

### Automated Node Update (AWS EKS):
```bash
# Update EKS node group
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup \
  --release-version 1.28.0

# Check update status
aws eks describe-update \
  --name my-cluster \
  --update-id <update-id>
```

---

## 5. IAM Permission Boundary for S3 Access

### Problem: Admin can't access S3 bucket

### Solution: IAM Permission Boundary

**Step 1: Create Permission Boundary Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::specific-bucket",
        "arn:aws:s3:::specific-bucket/*"
      ]
    },
    {
      "Effect": "Deny",
      "Action": [
        "s3:DeleteBucket",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": "*"
    }
  ]
}
```

**Step 2: Attach Permission Boundary to IAM Role**
```bash
# Create permission boundary
aws iam create-policy \
  --policy-name S3PermissionBoundary \
  --policy-document file://permission-boundary.json

# Attach to role
aws iam put-role-permissions-boundary \
  --role-name AdminRole \
  --permissions-boundary-arn arn:aws:iam::123456789012:policy/S3PermissionBoundary
```

**Step 3: Test Access**
```bash
# Try accessing allowed bucket
aws s3 ls s3://specific-bucket/  # Should work

# Try accessing other bucket
aws s3 ls s3://other-bucket/  # Should fail with AccessDenied
```

### Terraform Implementation:
```hcl
resource "aws_iam_policy" "permission_boundary" {
  name = "S3PermissionBoundary"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::specific-bucket",
          "arn:aws:s3:::specific-bucket/*"
        ]
      }
    ]
  })
}

resource "aws_iam_role" "admin" {
  name = "AdminRole"
  
  permissions_boundary = aws_iam_policy.permission_boundary.arn
}
```

---

## 6. Various Stages of CI/CD

### Complete CI/CD Pipeline Stages:

**1. Source Code Management:**
```bash
# Git operations
git clone repository
git checkout feature-branch
git commit -m "Changes"
git push origin feature-branch
```

**2. Code Build:**
```groovy
stage('Build') {
    steps {
        sh 'mvn clean package'
        // or
        sh 'npm run build'
        // or
        sh 'docker build -t myapp:latest .'
    }
}
```

**3. Unit Testing:**
```groovy
stage('Unit Tests') {
    steps {
        sh 'mvn test'
        sh 'pytest tests/'
        sh 'npm test'
    }
}
```

**4. Code Quality Analysis:**
```groovy
stage('Code Quality') {
    steps {
        sh 'mvn sonar:sonar -Dsonar.projectKey=myapp'
        sh 'eslint .'
        sh 'pylint app/'
    }
}
```

**5. Security Scanning:**
```groovy
stage('Security Scan') {
    steps {
        sh 'trivy image myapp:latest'
        sh 'dependency-check --scan .'
        sh 'owasp-zap scan http://localhost:8080'
    }
}
```

**6. Integration Testing:**
```groovy
stage('Integration Tests') {
    steps {
        sh 'mvn integration-test'
        sh 'docker-compose up -d'
        sh 'pytest integration/'
    }
}
```

**7. Artifact Management:**
```groovy
stage('Push Artifact') {
    steps {
        sh 'docker push registry/myapp:latest'
        sh 'mvn deploy'
        sh 'npm publish'
    }
}
```

**8. Deployment to Environments:**
```groovy
stage('Deploy to DEV') {
    steps {
        sh 'kubectl apply -f k8s/dev/'
    }
}

stage('Deploy to QA') {
    steps {
        sh 'kubectl apply -f k8s/qa/'
    }
}

stage('Deploy to PROD') {
    input {
        message "Deploy to Production?"
    }
    steps {
        sh 'kubectl apply -f k8s/prod/'
    }
}
```

**9. Post-Deployment Verification:**
```groovy
stage('Smoke Tests') {
    steps {
        sh 'curl -f http://app.example.com/health'
        sh 'pytest smoke-tests/'
    }
}
```

**10. Monitoring and Feedback:**
```groovy
stage('Notify') {
    steps {
        slackSend(
            channel: '#deployments',
            message: "Deployment successful"
        )
    }
}
```

---

## 7. Building and Managing Images During CI

### Image Build Strategy:

**1. Multi-Stage Docker Build:**
```dockerfile
# Dockerfile
# Build stage
FROM maven:3.8-openjdk-11 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src/ src/
RUN mvn package -DskipTests

# Runtime stage
FROM openjdk:11-jre-slim
WORKDIR /app
COPY --from=builder /app/target/myapp.jar app.jar
EXPOSE 8080
USER 1000
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**2. Jenkins Pipeline for Image Build:**
```groovy
pipeline {
    environment {
        REGISTRY = 'myregistry.com'
        IMAGE_NAME = 'myapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Build Image') {
            steps {
                script {
                    docker.build("${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('Tag Images') {
            steps {
                script {
                    // Tag with multiple tags
                    sh "docker tag ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:latest"
                    sh "docker tag ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT}"
                    sh "docker tag ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:${BRANCH_NAME}"
                }
            }
        }
        
        stage('Scan Image') {
            steps {
                sh "trivy image ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
        
        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'registry-credentials') {
                        docker.image("${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}").push()
                        docker.image("${REGISTRY}/${IMAGE_NAME}:latest").push()
                    }
                }
            }
        }
    }
}
```

**3. Image Versioning Strategy:**
```bash
# Semantic versioning
IMAGE_TAG="v1.2.3"

# Git commit based
IMAGE_TAG="${GIT_COMMIT:0:7}"

# Build number
IMAGE_TAG="${BUILD_NUMBER}"

# Date-based
IMAGE_TAG="$(date +%Y%m%d-%H%M%S)"
```

**4. Image Cleanup Policy:**
```bash
#!/bin/bash
# Cleanup old images
# Keep last 10 images, delete rest

aws ecr list-images \
  --repository-name myapp \
  --query 'imageIds[?type=="imageTag"]' \
  --output json | \
  jq 'sort_by(.imageTag) | reverse | .[10:]' | \
  jq -r '.[].imageDigest' | \
  while read digest; do
    aws ecr batch-delete-image \
      --repository-name myapp \
      --image-ids imageDigest=${digest}
  done
```

---

## 8. Access S3 Bucket from Different Region

### Cross-Region S3 Access:

**Answer: Yes, it is possible**

S3 buckets are global resources. You can access them from any region.

**Methods:**

**1. Direct Access (Default):**
```bash
# From us-east-1 EC2, access bucket in us-south-1
aws s3 ls s3://my-bucket-in-us-south-1/

# This works because S3 is global
```

**2. Using Regional Endpoint:**
```bash
# Specify region explicitly
aws s3api get-object \
  --bucket my-bucket \
  --key file.txt \
  --region us-south-1 \
  local-file.txt
```

**3. VPC Endpoint for Private Access:**
```hcl
# Create VPC endpoint in us-east-1
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.us-east-1.s3"
  
  route_table_ids = [aws_route_table.private.id]
}
```

**Considerations:**
- **Latency:** Cross-region access has higher latency
- **Cost:** Data transfer between regions has cost
- **Performance:** Use CloudFront or S3 Transfer Acceleration for better performance
- **Best Practice:** Use S3 replication for frequently accessed data

**Terraform for Cross-Region Replication:**
```hcl
resource "aws_s3_bucket" "source" {
  provider = aws.us_south_1
  bucket   = "source-bucket"
  
  versioning {
    enabled = true
  }
}

resource "aws_s3_bucket" "destination" {
  provider = aws.us_east_1
  bucket   = "destination-bucket"
  
  versioning {
    enabled = true
  }
}

resource "aws_s3_bucket_replication_configuration" "replication" {
  provider = aws.us_south_1
  
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.source.id
  
  rule {
    id     = "replicate-all"
    status = "Enabled"
    
    destination {
      bucket        = aws_s3_bucket.destination.arn
      storage_class = "STANDARD"
    }
  }
}
```

---

## 9. Terraform Code for VPC, Subnet, EC2, S3

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
}

provider "aws" {
  region = "us-east-1"
}

# main.tf
# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "main-vpc"
    Environment = "production"
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "public-subnet"
    Type = "public"
  }
}

# Private Subnet
resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1a"
  
  tags = {
    Name = "private-subnet"
    Type = "private"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "main-igw"
  }
}

# Route Table (Public)
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "public-rt"
  }
}

# Route Table Association (Public)
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Elastic IP for NAT Gateway
resource "aws_eip" "nat" {
  domain = "vpc"
}

# NAT Gateway
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public.id
  
  tags = {
    Name = "main-nat"
  }
}

# Route Table (Private)
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
  
  tags = {
    Name = "private-rt"
  }
}

# Route Table Association (Private)
resource "aws_route_table_association" "private" {
  subnet_id      = aws_subnet.private.id
  route_table_id = aws_route_table.private.id
}

# Security Group for EC2
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web server"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "web-sg"
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = "ami-0c7217cdde317cfec"  # Ubuntu 22.04 LTS
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  root_block_device {
    volume_size = 20
    volume_type = "gp3"
  }
  
  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
    systemctl enable nginx
  EOF
  
  tags = {
    Name = "web-server"
    Environment = "production"
  }
}

# S3 Bucket
resource "aws_s3_bucket" "logs" {
  bucket = "my-application-logs-20240101"  # Must be globally unique
  
  tags = {
    Name        = "application-logs"
    Environment = "production"
  }
}

# S3 Bucket Versioning
resource "aws_s3_bucket_versioning" "logs" {
  bucket = aws_s3_bucket.logs.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# S3 Bucket Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# S3 Bucket Lifecycle
resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id
  
  rule {
    id     = "delete-old-logs"
    status = "Enabled"
    
    expiration {
      days = 90
    }
    
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
  }
}

# S3 Bucket Policy
resource "aws_s3_bucket_policy" "logs" {
  bucket = aws_s3_bucket.logs.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.ec2_role.arn
        }
        Action = [
          "s3:PutObject",
          "s3:GetObject"
        ]
        Resource = [
          aws_s3_bucket.logs.arn,
          "${aws_s3_bucket.logs.arn}/*"
        ]
      }
    ]
  })
}

# Outputs
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_id" {
  value = aws_subnet.public.id
}

output "private_subnet_id" {
  value = aws_subnet.private.id
}

output "ec2_public_ip" {
  value = aws_instance.web.public_ip
}

output "s3_bucket_name" {
  value = aws_s3_bucket.logs.id
}
```

---

## 10. Purpose of CNI in Kubernetes

### Container Network Interface (CNI):

**Purpose:**
- Provides networking for containers/pods
- Assigns IP addresses to pods
- Enables pod-to-pod communication
- Implements network policies
- Manages network isolation

**Popular CNI Plugins:**

1. **Calico:** Network policies, BGP routing
2. **Flannel:** Simple overlay network
3. **Weave:** Mesh networking
4. **Cilium:** eBPF-based, service mesh
5. **AWS VPC CNI:** Native AWS networking

**How CNI Works:**
```yaml
# CNI Configuration
apiVersion: v1
kind: Config
metadata:
  name: cni-config
data:
  cni-conf.json: |
    {
      "cniVersion": "0.4.0",
      "name": "mynet",
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "subnet": "10.244.0.0/16",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ]
      }
    }
```

**Pod Networking Flow:**
```
Pod → veth pair → CNI bridge → Node network → Other nodes
```

---

## 11. Automation: EC2 Logs to S3 with CPU Monitoring and Alarms

### Complete Automation Script:

```python
#!/usr/bin/env python3
"""
EC2 Log Management and Monitoring Automation
1. Collect application logs
2. Upload to S3
3. Check CPU metrics
4. Send CloudWatch alarms
"""

import boto3
import os
import subprocess
import json
from datetime import datetime, timedelta

class EC2LogManager:
    def __init__(self, bucket_name, instance_id):
        self.s3 = boto3.client('s3')
        self.cloudwatch = boto3.client('cloudwatch')
        self.sns = boto3.client('sns')
        self.bucket_name = bucket_name
        self.instance_id = instance_id
        
    def collect_logs(self):
        """Collect logs from various sources"""
        log_files = {
            '/var/log/application.log': 'application',
            '/var/log/nginx/access.log': 'nginx-access',
            '/var/log/nginx/error.log': 'nginx-error',
            '/var/log/syslog': 'system'
        }
        
        collected_logs = []
        for file_path, log_type in log_files.items():
            if os.path.exists(file_path):
                timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
                log_name = f"{log_type}_{timestamp}.log"
                collected_logs.append((file_path, log_name))
        
        return collected_logs
    
    def upload_to_s3(self, logs):
        """Upload logs to S3"""
        uploaded_files = []
        date_prefix = datetime.now().strftime('%Y/%m/%d')
        
        for file_path, log_name in logs:
            s3_key = f"logs/{self.instance_id}/{date_prefix}/{log_name}"
            try:
                self.s3.upload_file(file_path, self.bucket_name, s3_key)
                uploaded_files.append(s3_key)
                print(f"Uploaded: {s3_key}")
            except Exception as e:
                print(f"Failed to upload {file_path}: {e}")
        
        return uploaded_files
    
    def check_cpu_metrics(self):
        """Check CPU utilization metrics"""
        response = self.cloudwatch.get_metric_statistics(
            Namespace='AWS/EC2',
            MetricName='CPUUtilization',
            Dimensions=[
                {
                    'Name': 'InstanceId',
                    'Value': self.instance_id
                },
            ],
            StartTime=datetime.utcnow() - timedelta(minutes=30),
            EndTime=datetime.utcnow(),
            Period=300,
            Statistics=['Average', 'Maximum']
        )
        
        return response['Datapoints']
    
    def create_cpu_alarm(self, threshold=80):
        """Create CloudWatch alarm for high CPU"""
        response = self.cloudwatch.put_metric_alarm(
            AlarmName=f'HighCPU-{self.instance_id}',
            ComparisonOperator='GreaterThanThreshold',
            EvaluationPeriods=2,
            MetricName='CPUUtilization',
            Namespace='AWS/EC2',
            Period=300,
            Statistic='Average',
            Threshold=threshold,
            ActionsEnabled=True,
            AlarmDescription='Alarm when CPU exceeds 80%',
            Dimensions=[
                {
                    'Name': 'InstanceId',
                    'Value': self.instance_id
                },
            ],
            AlarmActions=[
                'arn:aws:sns:us-east-1:123456789012:ops-team'
            ]
        )
        return response
    
    def send_sns_notification(self, subject, message):
        """Send SNS notification"""
        response = self.sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:ops-team',
            Subject=subject,
            Message=message
        )


1. NAT Gateway in Private Subnet

Technically possible but NOT recommended and defeats the purpose:

A NAT Gateway should be placed in a public subnet because:

· Private subnets don't have direct internet access
· NAT Gateway needs internet connectivity to forward traffic
· It requires a route to Internet Gateway (IGW)

Incorrect setup (NAT in private subnet):

```
Private Subnet → NAT Gateway → ??? (no route to internet)
```

Correct setup:

```
Private Subnet → NAT Gateway (in Public Subnet) → Internet Gateway → Internet
```

2. Cluster Auto-scaling Configuration

For Kubernetes (EKS/GKE/AKS):

Step 1: Install Cluster Autoscaler

For AWS EKS:

```bash
# Download the Cluster Autoscaler manifest
curl -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Edit the manifest and set your cluster name
sed -i 's/<YOUR CLUSTER NAME>/my-cluster-name/g' cluster-autoscaler-autodiscover.yaml

# Apply the manifest
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

Step 2: Configure IAM Policy (EKS example)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*"
        }
    ]
}
```

Step 3: Tag Auto Scaling Groups

Add these tags to your Auto Scaling Groups:

```bash
k8s.io/cluster-autoscaler/enabled: true
k8s.io/cluster-autoscaler/my-cluster-name: owned
```

Step 4: Create Autoscaling Policies

Example HPA (Horizontal Pod Autoscaler):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

For AWS ECS:

```bash
# Create Auto Scaling policy
aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --scalable-dimension ecs:service:DesiredCount \
    --resource-id service/my-cluster/my-service \
    --min-capacity 2 \
    --max-capacity 10

# Create scaling policy
aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --scalable-dimension ecs:service:DesiredCount \
    --resource-id service/my-cluster/my-service \
    --policy-name cpu-scaling \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration '{
        "TargetValue": 70.0,
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
        },
        "ScaleOutCooldown": 60,
        "ScaleInCooldown": 300
    }'
```

Best Practices for Auto-scaling:

1. Set appropriate min/max limits
2. Use multiple metrics (CPU, memory, custom metrics)
3. Configure cooldown periods to prevent flapping
4. Test scaling scenarios
5. Monitor scaling events
6. Use node pool taints/tolerations for specialized workloads
