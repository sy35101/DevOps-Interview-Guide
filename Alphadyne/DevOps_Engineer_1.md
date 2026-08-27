# Alphadyne

**YOE---> 5 Years**

- Interviewer introduced himself and explained the role, team setup, and expectations and asked me to itroduce
- Application is hosted on a public EC2 instance.How to migrate it to a private subnet following AWS best practices (security, networking, HA).
- How to provide HTTPS access to an application hosted in a private subnet.
- Discussion around ALB/NLB, ACM certificates, routing, and security groups.
- Infrastructure changes I have worked on. Deep discussion on architecture changes, optimizations, and real challenges faced.
- One major production Kubernetes issue that I solved and still remember.
- Root cause analysis, troubleshooting approach, fix, and preventive measures.
- Branching strategy discussion in depth.
- Why this strategy was chosen
- How it supports multiple environments, releases, and hotfixes.
- New application scenario:Developer has written only source code As a DevOps engineer, how to design CI/CD...Deploying to DEV / QA / PROD using best practices.
- Multibranch pipeline in Jenkins: How to configure it What problems it solves Why it is preferred over a normal SCM pipeline.



--
Here are direct, interview-ready answers with accurate technical detail.

---

## 1. Migrating Application from Public EC2 to Private Subnet

### Architecture Before:
```
Internet → EC2 (Public IP) → Application
```

### Architecture After (Best Practices):
```
Internet → Route 53 → ALB (Public Subnets) → EC2 (Private Subnet) → Application
```

### Migration Steps:

**Step 1: Create Private Subnets**
```hcl
# Terraform example
resource "aws_subnet" "private" {
  count             = 2  # Multiple AZs for HA
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.10.${count.index}.0/24"
  availability_zone = element(["us-east-1a", "us-east-1b"], count.index)
  
  tags = {
    Name = "private-subnet-${count.index + 1}"
  }
}
```

**Step 2: Create NAT Gateway for Outbound Traffic**
```hcl
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

**Step 3: Create Launch Template with Private Networking**
```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "app-private-"
  image_id      = "ami-12345678"
  instance_type = "t3.medium"
  
  network_interfaces {
    associate_public_ip_address = false  # Key change
    security_groups             = [aws_security_group.app.id]
    subnet_id                   = aws_subnet.private[0].id
  }
  
  user_data = base64encode(<<-EOF
    #!/bin/bash
    # Application setup
    systemctl start myapp
  EOF
  )
  
  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "app-private"
      Environment = "production"
    }
  }
}
```

**Step 4: Create Auto Scaling Group for HA**
```hcl
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  vpc_zone_identifier = aws_subnet.private[*].id
  min_size            = 2
  max_size            = 6
  desired_capacity    = 2
  
  health_check_type         = "ELB"
  health_check_grace_period = 300
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  tag {
    key                 = "Name"
    value               = "app-private"
    propagate_at_launch = true
  }
}
```

**Step 5: Create Security Groups**
```hcl
# Application SG
resource "aws_security_group" "app" {
  name        = "app-sg"
  description = "Application security group"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # Only from ALB
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]  # Outbound via NAT
  }
}

# ALB SG
resource "aws_security_group" "alb" {
  name        = "alb-sg"
  description = "ALB security group"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # From internet
  }
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # For HTTP to HTTPS redirect
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**Step 6: Migrate Application**
```bash
# 1. Create AMI from current EC2
aws ec2 create-image --instance-id i-1234567890abcdef0 \
  --name "app-ami-$(date +%Y%m%d)" \
  --no-reboot

# 2. Test new setup in parallel
# 3. Update DNS to point to ALB
# 4. Monitor and decommission old instance
```

---

## 2. HTTPS Access for Application in Private Subnet

### Complete HTTPS Setup:

**Step 1: Request ACM Certificate**
```hcl
resource "aws_acm_certificate" "app" {
  domain_name       = "app.example.com"
  validation_method = "DNS"
  
  subject_alternative_names = [
    "*.example.com"
  ]
  
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.app.domain_validation_options : 
    dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }
  
  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.main.zone_id
}
```

**Step 2: Create ALB with HTTPS Listener**
```hcl
resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id  # ALB in public subnets
  
  enable_deletion_protection = true
  
  tags = {
    Name = "app-alb"
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate.app.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.app.arn
  port              = "80"
  protocol          = "HTTP"
  
  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

**Step 3: Create Target Group**
```hcl
resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  
  health_check {
    enabled             = true
    path                = "/health"
    port                = "traffic-port"
    healthy_threshold   = 3
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    matcher             = "200"
  }
  
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
    enabled         = true
  }
}
```

**Step 4: Attach Instances to Target Group**
```hcl
resource "aws_autoscaling_attachment" "app" {
  autoscaling_group_name = aws_autoscaling_group.app.id
  lb_target_group_arn    = aws_lb_target_group.app.arn
}
```

**Step 5: Create Route 53 Record**
```hcl
resource "aws_route53_record" "app" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
  
  alias {
    name                   = aws_lb.app.dns_name
    zone_id                = aws_lb.app.zone_id
    evaluate_target_health = true
  }
}
```

### NLB vs ALB Decision:

**Use ALB when:**
- HTTP/HTTPS traffic
- Path-based routing
- Host-based routing
- WebSocket support
- Need for WAF integration

**Use NLB when:**
- TCP/UDP traffic
- Extremely high performance requirements
- Static IP addresses needed
- TLS passthrough required
- Non-HTTP protocols

**NLB for HTTPS:**
```hcl
resource "aws_lb" "nlb" {
  name               = "app-nlb"
  internal           = false
  load_balancer_type = "network"
  subnets            = aws_subnet.public[*].id
  
  enable_deletion_protection = true
}

resource "aws_lb_listener" "nlb_tls" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = "443"
  protocol          = "TLS"
  certificate_arn   = aws_acm_certificate.app.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tcp.arn
  }
}
```

---

## 3. Major Production Kubernetes Issue I Solved

### Issue: Memory Leak Causing Pod OOMKilled

**Symptom:**
- Pods restarting every 2-3 hours
- Intermittent 502 errors
- High memory usage in Grafana

**Timeline:**
```
09:00 - Alert: Pod restart count > 5 in 2 hours
09:15 - On-call engineer starts investigation
09:30 - Incident declared as P2
10:00 - I joined investigation
11:00 - Root cause identified
12:00 - Fix deployed
13:00 - Monitoring confirmed stable
```

**Initial Investigation:**
```bash
# Check pod status
kubectl get pods -n production
# Output shows multiple restarts

NAME                    READY   STATUS    RESTARTS   AGE
app-7d8f9c5b6-xk2m    1/1     Running   12         5h
app-7d8f9c5b6-pl4n    1/1     Running   9          4h
app-7d8f9c5b6-mn8v    0/1     OOMKilled 15         3h

# Describe pod
kubectl describe pod app-7d8f9c5b6-xk2m -n production
# State: OOMKilled
# Last State: Terminated (OOMKilled)
# Exit Code: 137

# Check logs
kubectl logs app-7d8f9c5b6-xk2m --previous -n production
# MemoryError: unable to allocate array
```

**Memory Trend Analysis:**
```bash
# Using Prometheus query
# container_memory_usage_bytes{pod="app-*"} 
# Showed gradual increase over time (sawtooth pattern)
```

**Root Cause:**
- Java application had a connection leak in database error path
- Every failed connection added to connection pool without closing
- Memory accumulated until OOM
- Connection pool maxed at 100 connections

**Code Fix:**
```java
// Buggy code
public DatabaseConnection getConnection() {
    try {
        return connectionPool.getConnection();
    } catch (SQLException e) {
        // Missing connection.close() in error path
        return null;  // BUG: Connection leaked
    }
}

// Fixed code
public DatabaseConnection getConnection() {
    Connection conn = null;
    try {
        conn = connectionPool.getConnection();
        return conn;
    } catch (SQLException e) {
        if (conn != null) {
            conn.close();  // FIX: Properly close
        }
        throw new DatabaseException("Failed to get connection", e);
    }
}
```

**Immediate Fix:**
```yaml
# Increase memory temporarily
resources:
  limits:
    memory: "2Gi"  # Increased from 1Gi
  requests:
    memory: "1Gi"
```

**Permanent Fixes:**
1. **Code fix:** Proper connection management
2. **Monitoring:** Added JVM heap metrics
3. **Alerting:** Memory trend alerts (not just threshold)
4. **Load testing:** Simulated error scenarios

**Preventive Measures:**
```yaml
# HPA based on memory
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

---

## 4. Branching Strategy Discussion

### GitFlow Branching Strategy (Chosen for our organization)

**Branch Structure:**
```
main (production)
├── develop (integration)
│   ├── feature/feature-1
│   ├── feature/feature-2
│   └── feature/feature-3
├── release/v1.2.0
└── hotfix/hotfix-1
```

**Branches:**

1. **main**
   - Always production-ready
   - Tagged with version numbers
   - Protected branch (no direct pushes)

2. **develop**
   - Integration branch
   - Latest development changes
   - Auto-deploys to DEV environment

3. **feature/***
   - Created from develop
   - Individual features
   - Merged back to develop after review

4. **release/v***
   - Created from develop
   - Preparation for production
   - Deployed to QA/Staging
   - Bug fixes only

5. **hotfix/***
   - Created from main
   - Emergency fixes
   - Merged to main and develop

### Why GitFlow was Chosen:

**Advantages:**
- Clear separation of concerns
- Supports multiple environments
- Parallel development
- Controlled releases
- Quick hotfixes
- Clear versioning

**Environment Mapping:**
- `develop` → DEV environment
- `release/*` → QA/Staging environment
- `main` → Production environment
- `hotfix/*` → Immediate production fix

### Alternative: Trunk-Based Development

**Simpler alternative for smaller teams:**
```
main → Production
├── feature-branches (short-lived)
```

### Implementation with Jenkins Multibranch Pipeline:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
                sh 'mvn integration-test'
            }
        }
        
        stage('Code Quality') {
            steps {
                sh 'mvn sonar:sonar'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
            }
        }
        
        stage('Push to Registry') {
            steps {
                sh 'docker push registry/myapp:${BUILD_NUMBER}'
            }
        }
        
        stage('Deploy to DEV') {
            when {
                branch 'develop'
            }
            steps {
                sh 'kubectl apply -f k8s/dev/'
            }
        }
        
        stage('Deploy to QA') {
            when {
                branch 'release/*'
            }
            steps {
                sh 'kubectl apply -f k8s/qa/'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                sh 'kubectl apply -f k8s/prod/'
            }
        }
    }
    
    post {
        success {
            emailext to: 'team@example.com',
                     subject: "Build Successful: ${env.JOB_NAME}",
                     body: "Build ${env.BUILD_NUMBER} succeeded"
        }
        failure {
            emailext to: 'team@example.com',
                     subject: "Build Failed: ${env.JOB_NAME}",
                     body: "Build ${env.BUILD_NUMBER} failed"
        }
    }
}
```

---

## 5. CI/CD Design for New Application

### Scenario: Developer has source code only

### Complete CI/CD Pipeline Design:

**Architecture:**
```
GitHub → Jenkins (CI) → Docker Registry → ArgoCD (CD) → Kubernetes
         ↓                    ↓
      Testing              Scanning
         ↓                    ↓
      Quality              Security
```

### Step-by-Step Implementation:

**Step 1: Source Code Structure**
```
myapp/
├── src/
├── Dockerfile
├── docker-compose.yml
├── Jenkinsfile
├── k8s/
│   ├── dev/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── qa/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── prod/
│       ├── deployment.yaml
│       └── service.yaml
├── terraform/
└── README.md
```

**Step 2: Jenkins Multibranch Pipeline**

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8-openjdk-11
    command: ['cat']
    tty: true
  - name: docker
    image: docker:20
    command: ['cat']
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
'''
        }
    }
    
    environment {
        DOCKER_REGISTRY = 'myregistry.com'
        APP_NAME = 'myapp'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                container('maven') {
                    sh 'mvn test'
                }
            }
        }
        
        stage('Code Quality') {
            steps {
                container('maven') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=myapp'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh "docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} ."
                    sh "docker tag ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} ${DOCKER_REGISTRY}/${APP_NAME}:latest"
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                container('docker') {
                    sh "trivy image ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
                }
            }
        }
        
        stage('Push to Registry') {
            steps {
                container('docker') {
                    sh "docker push ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
                    sh "docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest"
                }
            }
        }
        
        stage('Deploy to DEV') {
            when {
                branch 'develop'
            }
            steps {
                sh "sed -i 's|IMAGE_TAG|${BUILD_NUMBER}|g' k8s/dev/deployment.yaml"
                sh 'kubectl apply -f k8s/dev/'
            }
        }
        
        stage('Deploy to QA') {
            when {
                branch 'release/*'
            }
            steps {
                sh "sed -i 's|IMAGE_TAG|${BUILD_NUMBER}|g' k8s/qa/deployment.yaml"
                sh 'kubectl apply -f k8s/qa/'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to Production?"
                ok "Yes, deploy"
            }
            steps {
                sh "sed -i 's|IMAGE_TAG|${BUILD_NUMBER}|g' k8s/prod/deployment.yaml"
                sh 'kubectl apply -f k8s/prod/'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                channel: '#ci-cd',
                color: 'good',
                message: "Build ${env.JOB_NAME} #${env.BUILD_NUMBER} succeeded"
            )
        }
        failure {
            slackSend(
                channel: '#ci-cd',
                color: 'danger',
                message: "Build ${env.JOB_NAME} #${env.BUILD_NUMBER} failed"
            )
        }
    }
}
```

**Step 3: Kubernetes Deployment Files**

**DEV Environment:**
```yaml
# k8s/dev/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry.com/myapp:IMAGE_TAG
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

**PROD Environment (with HA):**
```yaml
# k8s/prod/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
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
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: myapp
            topologyKey: topology.kubernetes.io/zone
      containers:
      - name: myapp
        image: myregistry.com/myapp:IMAGE_TAG
        ports:
        - containerPort: 8080
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
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

---

## 6. Multibranch Pipeline vs Normal SCM Pipeline

### Normal SCM Pipeline:
```groovy
// Jenkinsfile - Single branch
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                checkout scm  // Checkout specific branch configured in Jenkins
                sh 'mvn build'
            }
        }
    }
}
```

**Limitations:**
- One pipeline per branch
- Manual job creation for each branch
- No automatic branch detection
- Harder to manage multiple environments

### Multibranch Pipeline:
```groovy
// Auto-detects and creates pipeline for each branch
// Jenkinsfile in each branch
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                checkout scm  // Auto-checkout the detected branch
                sh 'mvn build'
            }
        }
        stage('Deploy to Environment') {
            when {
                expression { 
                    return env.BRANCH_NAME == 'main' || 
                           env.BRANCH_NAME.startsWith('release/')
                }
            }
            steps {
                sh 'deploy'
            }
        }
    }
}
```

### Configuration in Jenkins:

**Normal Pipeline:**
```
Jenkins Dashboard → New Item → Pipeline → Configure
- Pipeline Definition: Pipeline script from SCM
- SCM: Git
- Repository URL: https://github.com/org/repo.git
- Branch Specifier: */main  (single branch)
```

**Multibranch Pipeline:**
```
Jenkins Dashboard → New Item → Multibranch Pipeline → Configure
- Branch Sources: Git
- Repository URL: https://github.com/org/repo.git
- Discover branches: All branches (auto-detection)
- Discover pull requests: From origin
```

### Problems Multibranch Solves:

1. **Automatic Branch Detection:**
   - New branch pushed → Pipeline auto-created
   - Deleted branch → Pipeline auto-removed

2. **Consistent CI Across Branches:**
   - Every branch gets same pipeline
   - PR testing automatic

3. **Environment Isolation:**
   ```groovy
   stage('Deploy') {
       when {
           expression {
               if (env.BRANCH_NAME == 'main') {
                   return true  // Deploy to prod
               } else if (env.BRANCH_NAME == 'develop') {
                   return true  // Deploy to dev
               }
               return false
           }
       }
       steps {
           script {
               def namespace = env.BRANCH_NAME == 'main' ? 'production' : 
                               env.BRANCH_NAME == 'develop' ? 'dev' : 'test'
               sh "kubectl apply -f k8s/ -n ${namespace}"
           }
       }
   }
   ```

4. **Parallel Feature Development:**
   - Each feature branch gets own pipeline
   - No interference between branches

5. **Pull Request Validation:**
   - Automatic PR builds
   - Integration testing before merge

**Why Prefer Multibranch:**
- Scalability: Handles 100s of branches
- Automation: No manual job creation
- GitOps: Pipeline defined in code
- Flexibility: Different branches for different purposes
- Visibility: Dashboard shows all branches

---

These answers demonstrate real-world experience and best practices. Customize with your specific tools and experiences during the interview.
