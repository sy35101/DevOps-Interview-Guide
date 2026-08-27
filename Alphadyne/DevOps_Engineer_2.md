# Alphadyne

**YOE-->**

- Alphadyne assessment ->>
- multiple choice -
- tool should a company use to automatedeploy configuration managing vm and container
- module used to install nginx web server when automating using ansible in new server
- ansible module u useto ensure specific package is installed on a new websrrver while configuring the server
- ansible module u useto ensure specific package is installed on a new websrrver while configuring the server
- Object used to manage kubernetes cluster and ensure pod always has specific resources
- Best pick command to monitor disk write and reads
- i have pub key and i want a dev to provide access to a server. where do u store pub key
- User password on modern linux location
- Coding Task (Terraform on Azure):
- Define an Azure Virtual Network (VNet) with a given IP range
- Create multiple subnets for different layers (bastion, application, database)
- Configure Network Interfaces (NICs) and attach them to the correct subnets
- Provision Virtual Machines (VMs) using those NICs, with username and password authentication
- Create Public IPs where required
- Set up an Azure Bastion Host for secure access to the VMs
- Configure an Azure Load Balancer to route traffic to the application VMs







---
Here are direct, interview-ready answers with accurate technical detail.

---

## 1. Tool for Automated Deployment, Configuration Management of VMs and Containers

**Primary Tools:**

**For VMs:**
- **Ansible** - Agentless, YAML-based, push model
- **Terraform** - Infrastructure as Code
- **Puppet/Chef** - Agent-based, pull model
- **SaltStack** - Agent-based, fast

**For Containers:**
- **Kubernetes** - Container orchestration
- **Docker Compose** - Multi-container local/development
- **Helm** - Package manager for Kubernetes
- **Nomad** - Alternative orchestrator

**Combined Approach (Most Common):**
```
Terraform → Provision Infrastructure (VMs, Networks)
    ↓
Ansible → Configure VMs (Install packages, configure services)
    ↓
Docker → Package applications
    ↓
Kubernetes → Orchestrate containers
    ↓
Helm → Deploy applications to Kubernetes
```

**Real-World Stack Example:**
```yaml
# Infrastructure as Code
terraform apply -auto-approve

# Configuration Management
ansible-playbook -i inventory playbooks/webserver.yml

# Container Build
docker build -t myapp:latest .

# Container Deployment
helm upgrade --install myapp ./chart
```

---

## 2. Ansible Module to Install Nginx Web Server

**Using `ansible.builtin.package` (Recommended for cross-platform):**
```yaml
- name: Install Nginx using package module
  ansible.builtin.package:
    name: nginx
    state: present
```

**Using `ansible.builtin.apt` (Debian/Ubuntu):**
```yaml
- name: Install Nginx using apt module
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: yes
```

**Using `ansible.builtin.yum` (RHEL/CentOS):**
```yaml
- name: Install Nginx using yum module
  ansible.builtin.yum:
    name: nginx
    state: present
```

**Using `ansible.builtin.dnf` (Fedora/RHEL 8+):**
```yaml
- name: Install Nginx using dnf module
  ansible.builtin.dnf:
    name: nginx
    state: present
```

**Complete Playbook Example:**
```yaml
---
- name: Configure Web Server
  hosts: webservers
  become: yes
  
  tasks:
    - name: Install Nginx
      ansible.builtin.package:
        name: nginx
        state: present
    
    - name: Start Nginx service
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes
    
    - name: Deploy custom index.html
      ansible.builtin.copy:
        src: files/index.html
        dest: /usr/share/nginx/html/index.html
        owner: root
        group: root
        mode: '0644'
```

---

## 3. Ansible Module to Ensure Specific Package is Installed

**Primary Module: `ansible.builtin.package`**

**Usage:**
```yaml
- name: Ensure specific package is installed
  ansible.builtin.package:
    name: "{{ package_name }}"
    state: present
```

**Examples with Different Packages:**
```yaml
- name: Ensure multiple packages are installed
  ansible.builtin.package:
    name:
      - git
      - curl
      - wget
      - vim
    state: present

- name: Ensure specific version is installed
  ansible.builtin.package:
    name: python3=3.8.10
    state: present

- name: Ensure package is at latest version
  ansible.builtin.package:
    name: nginx
    state: latest
```

**Distribution-Specific Modules:**
```yaml
# For Debian/Ubuntu
- name: Install package using apt
  ansible.builtin.apt:
    name: package-name
    state: present
    update_cache: yes

# For RHEL/CentOS
- name: Install package using yum
  ansible.builtin.yum:
    name: package-name
    state: present

# For Fedora
- name: Install package using dnf
  ansible.builtin.dnf:
    name: package-name
    state: present

# For Alpine
- name: Install package using apk
  community.general.apk:
    name: package-name
    state: present
```

---

## 4. Object Used to Manage Kubernetes Cluster and Ensure Pod Resources

**Kubernetes Objects for Resource Management:**

**1. ResourceQuota (Namespace-level):**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
    pods: "50"
```

**2. LimitRange (Pod-level default resources):**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: production
spec:
  limits:
  - max:
      cpu: "2"
      memory: "4Gi"
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
```

**3. Resource Requests and Limits in Pod/Deployment:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

**4. PodDisruptionBudget (Ensure availability):**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

**5. HorizontalPodAutoscaler (Auto-scale resources):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
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

---

## 5. Best Command to Monitor Disk Writes and Reads

**Best Tools:**

**1. `iostat` (Most comprehensive):**
```bash
# Basic usage (updates every 2 seconds)
iostat -x 2

# Show only device statistics
iostat -d 2

# Extended statistics with more details
iostat -dx 2

# Show specific device
iostat -x /dev/sda 2

# Output explanation:
# r/s = read requests per second
# w/s = write requests per second
# rkB/s = kilobytes read per second
# wkB/s = kilobytes written per second
# await = average wait time (ms)
# %util = device utilization
```

**2. `vmstat` (Virtual memory statistics):**
```bash
# Show disk I/O
vmstat -d

# Continuous monitoring
vmstat 2

# Include timestamps
vmstat -t 2
```

**3. `iotop` (Real-time I/O by process):**
```bash
# Interactive monitoring
sudo iotop

# Batch mode (non-interactive)
sudo iotop -b -n 10

# Show only processes doing I/O
sudo iotop -o

# Show specific user's I/O
sudo iotop -u username
```

**4. `dstat` (Versatile statistics):**
```bash
# Show disk I/O
dstat -d

# Show disk read/write separately
dstat -d --disk-util

# Combine with other metrics
dstat -cdl 2
```

**5. `pidstat` (Per-process I/O):**
```bash
# Show I/O per process
pidstat -d 2

# Show specific PID
pidstat -d -p 1234 2

# Show all processes
pidstat -dl 2
```

**Example Output from iostat:**
```
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    5.00   10.00   100.00   200.00    40.00     0.15   10.00    5.00   12.50   1.00   1.50
```

---

## 6. Where to Store Public Key for Server Access

**Location: `~/.ssh/authorized_keys`**

**Complete Setup:**

**1. File Location:**
```bash
# For regular user
/home/username/.ssh/authorized_keys

# For root user
/root/.ssh/authorized_keys
```

**2. Add Public Key:**
```bash
# Method 1: Using ssh-copy-id
ssh-copy-id -i ~/.ssh/id_rsa.pub user@server

# Method 2: Manual append
cat ~/.ssh/id_rsa.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Method 3: Manual copy
scp ~/.ssh/id_rsa.pub user@server:/tmp/pubkey
ssh user@server "cat /tmp/pubkey >> ~/.ssh/authorized_keys && rm /tmp/pubkey"
```

**3. Set Correct Permissions:**
```bash
# On server
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

**4. Verify Setup:**
```bash
ssh -i ~/.ssh/id_rsa user@server
```

**Ansible Way:**
```yaml
- name: Add public key to authorized_keys
  ansible.posix.authorized_key:
    user: "{{ username }}"
    state: present
    key: "{{ lookup('file', '/path/to/public_key.pub') }}"
```

**Best Practices:**
- Store only public keys on servers
- Private keys stay on user's machine
- Use SSH key management tools for scale
- Rotate keys regularly
- Use SSH certificates for large deployments

---

## 7. User Password Location on Modern Linux

**Location: `/etc/shadow`**

**Details:**

**1. Password Storage:**
```bash
# View shadow file (requires root)
sudo cat /etc/shadow

# Example entry
username:$6$salt$hashed_password:18000:0:99999:7:::
```

**Fields Explanation:**
```
username:password:last_changed:min_age:max_age:warn:inactive:expire:reserved
```

**2. Related Files:**
```bash
# User account information
/etc/passwd

# User passwords (hashed)
/etc/shadow

# Group information
/etc/group

# Group passwords
/etc/gshadow
```

**3. Check Password Status:**
```bash
# Check password aging
chage -l username

# Check password hash algorithm
sudo grep username /etc/shadow | cut -d: -f2 | cut -d$ -f2
# 1 = MD5
# 5 = SHA-256
# 6 = SHA-512
# y = yescrypt

# Verify user exists
getent passwd username
```

**4. Password Hash Format:**
```
$6$salt$hash
$ = delimiter
6 = SHA-512 algorithm
salt = random salt
hash = hashed password
```

**Security Notes:**
- Passwords are always hashed (never plain text)
- Modern systems use SHA-512 or yescrypt
- Salt is unique per user
- Root access required to read /etc/shadow

---

## 8. Terraform on Azure - Complete Infrastructure

```hcl
# Configure Azure Provider
terraform {
  required_version = ">= 1.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "rg-production"
  location = "East US"
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Virtual Network
resource "azurerm_virtual_network" "main" {
  name                = "vnet-production"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  
  tags = {
    Environment = "Production"
  }
}

# Subnets
resource "azurerm_subnet" "bastion" {
  name                 = "subnet-bastion"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "application" {
  name                 = "subnet-application"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]
  
  delegation {
    name = "delegation"
    service_delegation {
      name = "Microsoft.Web/serverFarms"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/action",
      ]
    }
  }
}

resource "azurerm_subnet" "database" {
  name                 = "subnet-database"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.3.0/24"]
  
  enforce_private_link_endpoint_network_policies = true
}

# Azure Bastion Subnet (required for Bastion Host)
resource "azurerm_subnet" "azure_bastion" {
  name                 = "AzureBastionSubnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.4.0/27"]  # Minimum /27 for Azure Bastion
}

# Public IPs
resource "azurerm_public_ip" "bastion" {
  name                = "pip-bastion"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_public_ip" "load_balancer" {
  name                = "pip-lb"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

# Network Interfaces
resource "azurerm_network_interface" "app" {
  name                = "nic-app-01"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.application.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_network_interface" "db" {
  name                = "nic-db-01"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.database.id
    private_ip_address_allocation = "Dynamic"
  }
}

# Network Security Groups
resource "azurerm_network_security_group" "app" {
  name                = "nsg-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "Allow-HTTP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "Allow-HTTPS"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_network_security_group" "db" {
  name                = "nsg-db"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "Allow-From-App"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "5432"
    source_address_prefixes    = ["10.0.2.0/24"]
    destination_address_prefix = "*"
  }
  
  security_rule {
    name                       = "Deny-All"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

# Associate NSG with Subnets
resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id                 = azurerm_subnet.application.id
  network_security_group_id = azurerm_network_security_group.app.id
}

resource "azurerm_subnet_network_security_group_association" "db" {
  subnet_id                 = azurerm_subnet.database.id
  network_security_group_id = azurerm_network_security_group.db.id
}

# Virtual Machines
resource "azurerm_linux_virtual_machine" "app" {
  name                = "vm-app-01"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B2s"
  
  admin_username                  = "azureuser"
  admin_password                  = "P@ssw0rd123!"  # Use better password in production
  disable_password_authentication = false
  
  network_interface_ids = [
    azurerm_network_interface.app.id,
  ]
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }
  
  tags = {
    Environment = "Production"
    Role        = "Application"
  }
}

resource "azurerm_linux_virtual_machine" "db" {
  name                = "vm-db-01"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B2s"
  
  admin_username                  = "azureuser"
  admin_password                  = "P@ssw0rd123!"  # Use better password in production
  disable_password_authentication = false
  
  network_interface_ids = [
    azurerm_network_interface.db.id,
  ]
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }
  
  tags = {
    Environment = "Production"
    Role        = "Database"
  }
}

# Azure Bastion Host
resource "azurerm_bastion_host" "main" {
  name                = "bastion-host"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                 = "configuration"
    subnet_id            = azurerm_subnet.azure_bastion.id
    public_ip_address_id = azurerm_public_ip.bastion.id
  }
  
  tags = {
    Environment = "Production"
  }
}

# Load Balancer
resource "azurerm_lb" "main" {
  name                = "lb-production"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "Standard"

  frontend_ip_configuration {
    name                 = "PublicIPAddress"
    public_ip_address_id = azurerm_public_ip.load_balancer.id
  }
}

# Backend Pool
resource "azurerm_lb_backend_address_pool" "main" {
  loadbalancer_id = azurerm_lb.main.id
  name            = "BackendPool"
}

# Associate NIC with Backend Pool
resource "azurerm_network_interface_backend_address_pool_association" "app" {
  network_interface_id    = azurerm_network_interface.app.id
  ip_configuration_name   = "internal"
  backend_address_pool_id = azurerm_lb_backend_address_pool.main.id
}

# Health Probe
resource "azurerm_lb_probe" "main" {
  loadbalancer_id = azurerm_lb.main.id
  name            = "http-probe"
  port            = 80
  protocol        = "Http"
  request_path    = "/health"
}

# Load Balancer Rule
resource "azurerm_lb_rule" "main" {
  loadbalancer_id                = azurerm_lb.main.id
  name                           = "http-rule"
  protocol                       = "Tcp"
  frontend_port                  = 80
  backend_port                   = 80
  frontend_ip_configuration_name = "PublicIPAddress"
  backend_address_pool_id        = azurerm_lb_backend_address_pool.main.id
  probe_id                       = azurerm_lb_probe.main.id
}

# Output Values
output "resource_group_name" {
  value = azurerm_resource_group.main.name
}

output "virtual_network_name" {
  value = azurerm_virtual_network.main.name
}

output "load_balancer_public_ip" {
  value = azurerm_public_ip.load_balancer.ip_address
}

output "bastion_public_ip" {
  value = azurerm_public_ip.bastion.ip_address
}

output "app_vm_private_ip" {
  value = azurerm_network_interface.app.private_ip_address
}

output "db_vm_private_ip" {
  value = azurerm_network_interface.db.private_ip_address
}
```

**Deployment Commands:**
```bash
# Initialize Terraform
terraform init

# Preview changes
terraform plan

# Apply changes
terraform apply -auto-approve

# Destroy infrastructure
terraform destroy
```

**Key Features in this Configuration:**
1. **VNet with CIDR 10.0.0.0/16**
2. **Four subnets:** Bastion (10.0.1.0/24), Application (10.0.2.0/24), Database (10.0.3.0/24), AzureBastionSubnet (10.0.4.0/27)
3. **Network Security Groups** with proper rules
4. **Azure Bastion Host** for secure RDP/SSH access
5. **Load Balancer** with HTTP probe and rule
6. **VMs** with password authentication
7. **Public IPs** for Bastion and Load Balancer
8. **Proper tagging** for resource management

This complete setup follows Azure best practices for security and high availability.
