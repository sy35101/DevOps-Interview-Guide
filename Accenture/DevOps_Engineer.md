# Accenture

**EXP- 6yrs**


- You have created an IAM user in AWS and configured role-based access in EKS. How do you bind the IAM user to the EKS role?
- Assume you have 10 AWS accounts. How will you securely log in to them, considering access keys are not used for security reasons?
- What are the ways to log in to an AWS account?
- Does Amazon S3 require a VPC?
- What happen when we run terraform init ?
- write a terraform script to create an ec2 instance in multiple region.
- You have defined a multi-region Terraform configuration (region1, region2, region3). If you create an EC2 instance, in which region will it be deployed?
- If the frontend, backend, and database are all deployed in private subnets, how can an end user access the application?
- If secrets are created in AWS Secrets Manager, how can Amazon EKS access those secrets?
- How do you set up RBAC in Amazon EKS?


--------------------------------------


Here are direct, interview-ready answers with accurate technical detail.

---

### 1. IAM user + EKS role binding

EKS uses AWS IAM for **authentication** and Kubernetes RBAC for **authorization**. To bind an IAM user to a Kubernetes role:

**Option A – `aws-auth` ConfigMap (legacy)**  
Edit the `aws-auth` ConfigMap in `kube-system`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/john
      username: john
      groups:
        - dev-team
```

Then create a Kubernetes `Role` or `ClusterRole`, and bind it to the group `dev-team` using a `RoleBinding`/`ClusterRoleBinding`.

**Option B – EKS Access Entries (recommended for new clusters)**  
Use the EKS API/console:

```bash
aws eks create-access-entry --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/john \
  --kubernetes-groups dev-team
```

Then bind the Kubernetes group `dev-team` to a role with RBAC.

---

### 2. Secure login to 10 AWS accounts without access keys

Use **AWS IAM Identity Center (formerly AWS SSO)** with **AWS Organizations**:

- All 10 accounts are member accounts of the same AWS Organization.
- Enable IAM Identity Center in a central account (usually management or delegated admin).
- Integrate with your corporate identity provider (Okta, Azure AD, or AWS Managed Microsoft AD) via SAML/OIDC.
- Users log in to the IAM Identity Center portal with corporate credentials and MFA.
- Assign users/groups to permission sets and accounts.  
- Users get **short-lived temporary credentials** (via STS) when they assume a role in a target account.
- No long-lived access keys are created.

Alternative: use **cross-account IAM roles** from a central identity account, with federation via SAML/OIDC.

---

### 3. Ways to log in to an AWS account

- **AWS Management Console** with root user email/password or IAM user username/password.
- **IAM Identity Center (SSO) portal** – federated or AWS-managed users, temporary credentials.
- **AWS CLI/SDK** using IAM access keys (long-term, not recommended).
- **AWS CLI/SDK** using temporary credentials from `sts:AssumeRole` or `aws sso login`.
- **IAM Roles for AWS services** – EC2/ECS/Lambda automatically provide temporary credentials via instance metadata.
- **Federated login** via SAML 2.0 or OIDC to the console or CLI.
- **AWS Access Portal** / `aws sso login` for CLI with Identity Center.

---

### 4. Does Amazon S3 require a VPC?

No. Amazon S3 is a public service and does not reside in a VPC. It is accessed over HTTPS using public AWS endpoints.  
If you want private access from a VPC without internet, you can use an **S3 Gateway VPC Endpoint** or **Interface VPC Endpoint**. But S3 itself is not inside a VPC.

---

### 5. What happens when you run `terraform init`?

`terraform init` initializes the working directory:

- Reads the configuration files (`*.tf`) in the current directory.
- Determines required providers and modules.
- Downloads provider plugins from the Terraform Registry (or a configured mirror) into `.terraform/providers/`.
- Downloads referenced modules into `.terraform/modules/`.
- Initializes the configured **backend** (local or remote, e.g., S3 + DynamoDB). If migrating state, it prompts for confirmation.
- Creates/updates the **dependency lock file** (`.terraform.lock.hcl`) with provider versions and checksums.
- Verifies provider package authenticity.

It does **not** create or modify any infrastructure resources.

---

### 6. Terraform script to create EC2 instance in multiple regions

```hcl
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "us_west_2"
  region = "us-west-2"
}

data "aws_ami" "amzn2_us_east_1" {
  provider    = aws.us_east_1
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

data "aws_ami" "amzn2_us_west_2" {
  provider    = aws.us_west_2
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "example_us_east_1" {
  provider      = aws.us_east_1
  ami           = data.aws_ami.amzn2_us_east_1.id
  instance_type = "t3.micro"

  tags = {
    Name = "ec2-us-east-1"
  }
}

resource "aws_instance" "example_us_west_2" {
  provider      = aws.us_west_2
  ami           = data.aws_ami.amzn2_us_west_2.id
  instance_type = "t3.micro"

  tags = {
    Name = "ec2-us-west-2"
  }
}
```

---

### 7. Multi-region Terraform config: where will EC2 be deployed?

It will be deployed in **the region of the provider that is associated with the `aws_instance` resource**.

- If the resource does **not** have a `provider` argument, Terraform uses the **default (unaliased) `aws` provider**.
- The default provider’s `region` determines the region.
- Defining three provider aliases does **not** automatically deploy to all three regions; you need separate `aws_instance` resources or a module invoked with each provider.

So: **it goes to the default provider’s region unless you explicitly set the `provider` argument on the resource.**

---

### 8. Frontend, backend, database all in private subnets – how do end users access the app?

End users cannot reach private subnets directly. You need a **public-facing entry point** that forwards traffic to private resources:

- **Internet-facing Application Load Balancer (ALB)** in public subnets that routes to frontend/backend targets in private subnets. Users hit the ALB DNS name; the ALB sends traffic to private IPs.
- **Amazon CloudFront** distribution with an origin that can access the private ALB or, using **VPC Origins**, directly access private resources.
- **Amazon API Gateway** with a **VPC Link** to a private Network Load Balancer, which then routes to private services.
- Optionally protect with AWS WAF, security groups, and TLS.

The key: **public load balancer / CloudFront / API Gateway is in a public subnet or edge location, while the app tiers stay private.**

---

### 9. EKS accessing secrets from AWS Secrets Manager

Recommended methods using **IAM Roles for Service Accounts (IRSA)**:

**Method 1: IRSA + AWS SDK**
- Create an IAM OIDC provider for the EKS cluster.
- Create an IAM policy allowing `secretsmanager:GetSecretValue` on the secret ARN.
- Create an IAM role with a trust policy for the Kubernetes service account.
- Annotate the Kubernetes `ServiceAccount`:
  ```yaml
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-secret-role
  ```
- Application code uses AWS SDK to fetch the secret from Secrets Manager.

**Method 2: External Secrets Operator**
- Install External Secrets Operator.
- Create a `SecretStore` that references AWS Secrets Manager using IRSA.
- Create an `ExternalSecret` resource that syncs a Secrets Manager secret into a Kubernetes `Secret`.

**Method 3: Secrets Store CSI Driver + AWS Provider**
- Install the CSI driver and AWS provider.
- Use IRSA to grant access.
- Define a `SecretProviderClass` to mount the secret as a file or environment variable, or sync to a Kubernetes Secret.

**Never use long-lived IAM access keys inside pods.**

---

### 10. Setting up RBAC in Amazon EKS

**Legacy method (`aws-auth` ConfigMap):**
1. Create a Kubernetes `Role` or `ClusterRole` with desired permissions.
2. Edit the `aws-auth` ConfigMap in `kube-system` to map an IAM user/role to a Kubernetes username and group.
3. Create a `RoleBinding`/`ClusterRoleBinding` that binds that username/group to the role.

**Modern method (EKS Access Entries):**
1. Use AWS CLI/console to create an access entry for the IAM principal and assign Kubernetes groups:
   ```bash
   aws eks create-access-entry --cluster-name my-cluster \
     --principal-arn arn:aws:iam::123456789012:user/john \
     --kubernetes-groups dev-team
   ```
2. In Kubernetes, create `Role`/`ClusterRole` and bind the group `dev-team` using `RoleBinding`/`ClusterRoleBinding`.
3. Alternatively, associate a managed access policy such as `AmazonEKSAdminPolicy` or `AmazonEKSViewPolicy` to the access entry.

EKS authenticates via IAM; Kubernetes RBAC authorizes the actual API actions.

