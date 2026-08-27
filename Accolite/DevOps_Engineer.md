# Accolite

**Exp---- 3yr DevOps Engineer**

- Write a script to monitor a directory and print the names of new files added every minute. -->In Python
- What is the difference between set and list in python(Counter question of the above)
- You have two tables:
- Customers(customer_id, customer_name)
- Orders(order_id, customer_id, order_date, amount)
- Write a SQL query to find the names of customers who placed more than 3 orders in the last 90 days.
- Write a Python function that takes a list of dictionaries representing job logs. The function should return a list of job IDs where the "status" is "FAILED".
- Example:
- logs = [
- ("job_id": 101, "status":
- "SUCCESS", "timestamp":
- "2025-06-10T10:00:00"), ("job_id": 102, "status":
- "FAILED", "timestamp":
- "2025-06-10T10:05:00"), ("job_jd": 103, "status":
- "FAILED", "timestamp":
- *2025-06-10T10:10:00"),
- "job_jd": 104, "status":
- "SUCCESS", "imestamp";
- "2026-06-10T10:16:00")]

- What is CICD and explain in briefly
- Explain  kubernetes Architecture and component and their uses.
- Note -: Guy,s in Every question, they asked to explain in briefly and the code and how it using and what will be outputs.


---
Here are direct, interview-ready answers with accurate technical detail.

---

## 1. Python Script to Monitor Directory for New Files

```python
#!/usr/bin/env python3
"""
Directory Monitor Script
Monitors a specified directory and prints names of new files added every minute.
"""

import os
import time
import argparse
from datetime import datetime
import logging

class DirectoryMonitor:
    def __init__(self, directory_path, interval_seconds=60):
        """
        Initialize Directory Monitor
        
        Args:
            directory_path: Path to monitor
            interval_seconds: How often to check (default: 60 seconds)
        """
        self.directory_path = directory_path
        self.interval_seconds = interval_seconds
        self.known_files = set()
        self.setup_logging()
    
    def setup_logging(self):
        """Configure logging"""
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s'
        )
        self.logger = logging.getLogger(__name__)
    
    def get_current_files(self):
        """Get set of all files in directory"""
        try:
            return set(os.listdir(self.directory_path))
        except Exception as e:
            self.logger.error(f"Error reading directory: {e}")
            return set()
    
    def monitor(self):
        """Main monitoring loop"""
        self.logger.info(f"Starting to monitor: {self.directory_path}")
        self.logger.info(f"Checking every {self.interval_seconds} seconds")
        
        # Initialize with existing files (don't report them as new)
        self.known_files = self.get_current_files()
        self.logger.info(f"Initial files found: {len(self.known_files)}")
        
        try:
            while True:
                # Wait for specified interval
                time.sleep(self.interval_seconds)
                
                # Get current files
                current_files = self.get_current_files()
                
                # Find new files
                new_files = current_files - self.known_files
                
                # Report new files
                if new_files:
                    for file in new_files:
                        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                        print(f"[{timestamp}] NEW FILE: {file}")
                        self.logger.info(f"New file detected: {file}")
                
                # Update known files
                self.known_files = current_files
                
        except KeyboardInterrupt:
            self.logger.info("Monitoring stopped by user")
            print("\nMonitoring stopped")

def main():
    """Main function"""
    parser = argparse.ArgumentParser(description='Monitor directory for new files')
    parser.add_argument('directory', help='Directory path to monitor')
    parser.add_argument('--interval', type=int, default=60,
                       help='Check interval in seconds (default: 60)')
    
    args = parser.parse_args()
    
    # Validate directory
    if not os.path.exists(args.directory):
        print(f"Error: Directory '{args.directory}' does not exist")
        return
    
    # Create and start monitor
    monitor = DirectoryMonitor(args.directory, args.interval)
    monitor.monitor()

if __name__ == "__main__":
    main()
```

**Simplified Version (for interview):**
```python
import os
import time

def monitor_directory(directory_path, interval=60):
    """Monitor directory and print new files every minute"""
    
    # Get initial file list
    known_files = set(os.listdir(directory_path))
    print(f"Monitoring {directory_path}. Initial files: {len(known_files)}")
    
    while True:
        # Wait 60 seconds
        time.sleep(interval)
        
        # Get current files
        current_files = set(os.listdir(directory_path))
        
        # Find new files
        new_files = current_files - known_files
        
        # Print new files
        if new_files:
            for file in new_files:
                print(f"New file detected: {file} at {time.ctime()}")
        
        # Update known files
        known_files = current_files

# Usage
if __name__ == "__main__":
    monitor_directory("/path/to/directory")
```

**Output Example:**
```
Monitoring /var/data. Initial files: 5
New file detected: report_2024.pdf at Mon Jan 15 10:01:00 2024
New file detected: image_123.jpg at Mon Jan 15 10:02:00 2024
```

---

## 2. Difference Between Set and List in Python

**List:**
- **Ordered:** Elements maintain insertion order
- **Mutable:** Can add, remove, modify elements
- **Allows duplicates:** Can have same value multiple times
- **Indexed:** Access by position (list[0], list[1])
- **Syntax:** `[1, 2, 3, 4]`
- **Performance:** O(n) for membership testing
- **Use case:** When order matters or duplicates needed

**Set:**
- **Unordered:** No guaranteed order
- **Mutable:** Can add, remove elements
- **No duplicates:** Automatically removes duplicates
- **Not indexed:** Cannot access by position
- **Syntax:** `{1, 2, 3, 4}` or `set([1, 2, 3, 4])`
- **Performance:** O(1) for membership testing (hash-based)
- **Use case:** When uniqueness matters or fast lookup needed

**Example:**
```python
# List
my_list = [1, 2, 2, 3, 4, 4, 5]
print(my_list)  # [1, 2, 2, 3, 4, 4, 5]
print(my_list[0])  # 1 (can access by index)

# Set
my_set = {1, 2, 2, 3, 4, 4, 5}
print(my_set)  # {1, 2, 3, 4, 5} (duplicates removed)
# print(my_set[0])  # Error! Sets don't support indexing

# Performance comparison
import time

large_list = list(range(1000000))
large_set = set(range(1000000))

# List membership test
start = time.time()
999999 in large_list
list_time = time.time() - start

# Set membership test
start = time.time()
999999 in large_set
set_time = time.time() - start

print(f"List lookup: {list_time:.6f} seconds")
print(f"Set lookup: {set_time:.6f} seconds")
# Set is typically much faster
```

---

## 3. SQL Query for Customers with More Than 3 Orders in Last 90 Days

```sql
-- Query to find customers who placed more than 3 orders in last 90 days
SELECT 
    c.customer_id,
    c.customer_name,
    COUNT(o.order_id) as order_count
FROM 
    Customers c
INNER JOIN 
    Orders o ON c.customer_id = o.customer_id
WHERE 
    o.order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    -- For SQL Server: WHERE o.order_date >= DATEADD(day, -90, GETDATE())
    -- For PostgreSQL: WHERE o.order_date >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY 
    c.customer_id, 
    c.customer_name
HAVING 
    COUNT(o.order_id) > 3
ORDER BY 
    order_count DESC;
```

**Alternative Version (with subquery):**
```sql
SELECT 
    customer_name
FROM 
    Customers
WHERE 
    customer_id IN (
        SELECT 
            customer_id
        FROM 
            Orders
        WHERE 
            order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
        GROUP BY 
            customer_id
        HAVING 
            COUNT(*) > 3
    );
```

**Example Output:**
```
customer_id | customer_name | order_count
------------|---------------|------------
101         | John Smith    | 5
205         | Jane Doe      | 4
308         | Bob Johnson   | 4
```

---

## 4. Python Function to Find Failed Jobs

```python
def get_failed_jobs(job_logs):
    """
    Extract job IDs where status is "FAILED"
    
    Args:
        job_logs: List of dictionaries containing job information
    
    Returns:
        List of job IDs with FAILED status
    """
    failed_jobs = []
    
    for job in job_logs:
        if job.get("status") == "FAILED":
            failed_jobs.append(job.get("job_id"))
    
    return failed_jobs

# Alternative using list comprehension (more Pythonic)
def get_failed_jobs_v2(job_logs):
    """Extract failed job IDs using list comprehension"""
    return [job["job_id"] for job in job_logs if job.get("status") == "FAILED"]

# Function with error handling
def get_failed_jobs_robust(job_logs):
    """
    Robust version with error handling
    """
    failed_jobs = []
    
    for job in job_logs:
        try:
            if job["status"] == "FAILED":
                failed_jobs.append(job["job_id"])
        except KeyError as e:
            print(f"Missing key in job log: {e}")
            continue
    
    return failed_jobs

# Test data
logs = [
    {"job_id": 101, "status": "SUCCESS", "timestamp": "2025-06-10T10:00:00"},
    {"job_id": 102, "status": "FAILED", "timestamp": "2025-06-10T10:05:00"},
    {"job_id": 103, "status": "FAILED", "timestamp": "2025-06-10T10:10:00"},
    {"job_id": 104, "status": "SUCCESS", "timestamp": "2025-06-10T10:15:00"}
]

# Test the function
result = get_failed_jobs(logs)
print(f"Failed job IDs: {result}")
# Output: Failed job IDs: [102, 103]
```

---

## 5. What is CI/CD? (Brief Explanation)

**CI/CD** stands for **Continuous Integration** and **Continuous Delivery/Deployment**.

**Continuous Integration (CI):**
- Developers frequently merge code changes to shared repository
- Each merge triggers automated build and testing
- Catches bugs early
- Ensures code quality
- Tools: Jenkins, GitLab CI, GitHub Actions, CircleCI

**Continuous Delivery (CD):**
- Automatically prepares code for release
- Every change is deployable
- Requires manual approval for production deployment
- Ensures reliable releases

**Continuous Deployment (CD):**
- Automatically deploys to production after passing tests
- No manual intervention
- Requires comprehensive automated testing
- Faster feature delivery

**CI/CD Pipeline Flow:**
```
Code Commit → Build → Unit Tests → Integration Tests → Package → Deploy to Staging → 
Smoke Tests → Security Scan → Deploy to Production → Monitor
```

**Example Jenkins Pipeline:**
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t myapp:latest .'
            }
        }
        stage('Push to Registry') {
            steps {
                sh 'docker push myregistry/myapp:latest'
            }
        }
        stage('Deploy to Staging') {
            steps {
                sh 'kubectl apply -f k8s/staging/'
            }
        }
        stage('Deploy to Production') {
            when {
                expression { 
                    return env.BRANCH_NAME == 'main'
                }
            }
            steps {
                sh 'kubectl apply -f k8s/production/'
            }
        }
    }
}
```

**Benefits:**
- Faster time to market
- Early bug detection
- Consistent deployments
- Reduced manual errors
- Better collaboration
- Faster rollback capability

---

## 6. Kubernetes Architecture and Components

**Master Node (Control Plane) Components:**

**1. API Server (kube-apiserver)**
- Central management point for all operations
- Validates and processes REST requests
- Frontend to the cluster
- All components communicate through it
- Example: `kubectl` commands go to API server

**2. etcd**
- Distributed key-value store
- Stores cluster state and configuration
- Source of truth for cluster
- Consistent and highly available
- Example: Stores pod definitions, service endpoints

**3. Scheduler (kube-scheduler)**
- Assigns pods to nodes
- Considers resource requirements
- Evaluates node capacity, affinity rules
- Watches for new unassigned pods
- Example: Schedules pod to node with enough memory

**4. Controller Manager (kube-controller-manager)**
- Runs controller processes
- Node controller, replication controller, endpoints controller
- Maintains desired state
- Watches for changes and responds
- Example: Ensures specified number of replicas running

**5. Cloud Controller Manager**
- Interfaces with cloud provider APIs
- Manages cloud-specific resources
- Node lifecycle, load balancers, routes
- Example: Creates cloud load balancer for service

**Worker Node Components:**

**1. Kubelet**
- Agent running on every node
- Ensures containers running in pods
- Communicates with API server
- Reports node and pod status
- Example: Starts container when pod assigned

**2. Kube-proxy**
- Network proxy on each node
- Maintains network rules
- Enables pod communication
- Load balancing for services
- Example: Routes service traffic to appropriate pod

**3. Container Runtime**
- Software to run containers
- Docker, containerd, CRI-O
- Pulls images and starts containers
- Example: containerd runs the actual container

**Pods:**
- Smallest deployable unit
- One or more containers
- Shared storage and network
- Ephemeral

**Architecture Diagram:**
```
┌─────────────────────────────────────────┐
│           CONTROL PLANE                 │
│  ┌──────────┐  ┌──────────────────┐   │
│  │ API      │  │ etcd             │   │
│  │ Server   │  │ (State Store)    │   │
│  └──────────┘  └──────────────────┘   │
│  ┌──────────┐  ┌──────────────────┐   │
│  │ Scheduler│  │ Controller       │   │
│  │          │  │ Manager          │   │
│  └──────────┘  └──────────────────┘   │
└─────────────────────────────────────────┘
                    │
                    │
┌─────────────────────────────────────────┐
│           WORKER NODE 1                 │
│  ┌──────────┐  ┌──────────────────┐   │
│  │ Kubelet  │  │ Kube-proxy       │   │
│  └──────────┘  └──────────────────┘   │
│  ┌──────────────────────────────┐     │
│  │  Pod1          Pod2          │     │
│  │  ┌──────┐     ┌──────┐      │     │
│  │  │Cont1 │     │Cont1 │      │     │
│  │  └──────┘     └──────┘      │     │
│  └──────────────────────────────┘     │
└─────────────────────────────────────────┘
```

**How Components Work Together:**
1. User runs `kubectl create deployment`
2. API server receives request
3. Scheduler assigns pod to node
4. Kubelet on node receives notification
5. Container runtime pulls image and starts container
6. Kube-proxy configures networking
7. Controller manager maintains desired state
8. etcd stores all configuration
