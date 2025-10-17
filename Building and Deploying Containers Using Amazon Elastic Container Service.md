# Deploying Flask Microservices to AWS Using Free Tier Services

**Guide Version: 1.0.0**

***

## Table of Contents

- [Lab Overview](#lab-overview)
- [Architecture Overview](#architecture-overview)
- [Objectives](#objectives)
- [AWS Free Tier Services Used](#aws-free-tier-services-used)
- [Technical Prerequisites](#technical-prerequisites)
- [Icon Key](#icon-key)
- [Prerequisites Setup](#prerequisites-setup)
- [Task 1: Setting Up Your AWS Account and IAM](#task-1-setting-up-your-aws-account-and-iam)
- [Task 2: Creating an EC2 Command Host](#task-2-creating-an-ec2-command-host)
- [Task 3: Installing Docker and AWS CLI](#task-3-installing-docker-and-aws-cli)
- [Task 4: Exploring the Application Structure](#task-4-exploring-the-application-structure)
- [Task 5: Building the Inventory System (invsys) Container](#task-5-building-the-inventory-system-invsys-container)
- [Task 6: Building the Gateway Container](#task-6-building-the-gateway-container)
- [Task 7: Setting Up Amazon ECR Repositories](#task-7-setting-up-amazon-ecr-repositories)
- [Task 8: Pushing Images to ECR](#task-8-pushing-images-to-ecr)
- [Task 9: Creating an ECS Cluster](#task-9-creating-an-ecs-cluster)
- [Task 10: Configuring Application Load Balancer](#task-10-configuring-application-load-balancer)
- [Task 11: Creating ECS Task Definitions](#task-11-creating-ecs-task-definitions)
- [Task 12: Deploying Services to ECS](#task-12-deploying-services-to-ecs)
- [Task 13: Testing Your Deployment](#task-13-testing-your-deployment)
- [Task 14: Monitoring and Troubleshooting](#task-14-monitoring-and-troubleshooting)
- [Cleanup Instructions](#cleanup-instructions)
- [Cost Optimization Tips](#cost-optimization-tips)
- [Additional Resources](#additional-resources)

***

## Lab Overview

This comprehensive guide demonstrates how to deploy a **multi-service Flask microservices application** to AWS using **free tier eligible services**. You'll deploy an inventory management system consisting of two microservices:

- **Inventory System (invsys)**: REST API backend that manages device inventory data with CRUD operations
- **Gateway Service**: API gateway that routes requests to the inventory system and provides a unified entry point

By the end of this guide, you'll have a production-ready, scalable deployment running on AWS ECS with load balancing and container orchestration.

***

## Architecture Overview

```
Internet
    ↓
Application Load Balancer (ALB)
    ↓
┌─────────────────────────────────────┐
│   ECS Cluster                       │
│                                     │
│  ┌──────────────┐  ┌──────────────┐│
│  │ Gateway      │  │ Inventory    ││
│  │ Service      │→ │ System       ││
│  │ (Port 5001)  │  │ (Port 5000)  ││
│  └──────────────┘  └──────────────┘│
└─────────────────────────────────────┘
           ↓
    Amazon ECR (Docker Images)
```

**Key Components:**
- **Amazon ECS**: Container orchestration service (free tier: no additional charge for ECS control plane)
- **Amazon ECR**: Private Docker image registry (free tier: 500 MB storage/month)
- **Application Load Balancer**: Distributes traffic to containers (free tier: 750 hours/month for 12 months)
- **EC2 t2.micro/t3.micro**: Hosts for ECS tasks (free tier: 750 hours/month for 12 months)
- **Amazon VPC**: Network isolation (free tier: always free)

***

## Objectives

By completing this guide, you will be able to:

- Set up and configure AWS services within free tier limits
- Build Docker images for Flask microservices applications
- Push Docker images to Amazon Elastic Container Registry (ECR)
- Create and configure an ECS cluster with EC2 launch type
- Deploy multi-container applications with service discovery
- Configure Application Load Balancer for traffic distribution
- Implement health checks and auto-scaling policies
- Monitor and troubleshoot containerized applications

***

## AWS Free Tier Services Used

This deployment is designed to stay within AWS Free Tier limits:

| Service | Free Tier Allowance | Our Usage |
|---------|---------------------|-----------|
| Amazon ECS | No charge for control plane | 1 cluster |
| Amazon ECR | 500 MB storage/month | ~300 MB (2 images) |
| EC2 t2.micro | 750 hours/month for 12 months | 1-2 instances |
| Application Load Balancer | 750 hours/month for 12 months | 1 ALB |
| VPC | Always free | 1 VPC |
| Data Transfer | 100 GB outbound/month | Minimal (<5 GB) |

**Note:** Free tier benefits are only available for the first 12 months after AWS account creation (except VPC which is always free). Monitor your usage in AWS Billing Dashboard.

***

## Technical Prerequisites

### Required Knowledge
- Basic understanding of Docker and containerization
- Familiarity with Flask/Python web applications
- Command-line interface (CLI) experience
- Basic networking concepts (ports, HTTP, load balancing)

### Required Tools
- AWS Account (with free tier eligibility)
- SSH client (built-in on macOS/Linux, PuTTY for Windows)
- Modern web browser (Chrome, Firefox, Safari)
- Credit card (required for AWS account verification, but won't be charged for free tier usage)

### System Requirements
- Computer with internet connection
- Minimum 4 GB RAM recommended for development
- 10 GB free disk space for Docker images

***

## Icon Key

| Icon | Description |
|------|-------------|
| 💻 Command | Run this command in terminal |
| 📋 Expected output | Sample command output you should see |
| 💡 Note | Helpful hints, tips, or important guidance |
| ⚠️ Caution | Important warnings about costs or potential issues |
| 📚 Learn more | Reference links for additional information |
| ✅ Task complete | Checkpoint confirming task completion |
| 🔧 Troubleshooting | Common issues and solutions |
| 💰 Cost optimization | Tips to minimize AWS charges |

***

## Prerequisites Setup

Before starting, ensure you have:

1. **AWS Account**
   - Sign up at https://aws.amazon.com/free/
   - Verify your account with credit card
   - Enable MFA for security (highly recommended)

2. **Application Code**
   - Complete the Building a multicomponent Flask app course
   - Verify Dockerfiles are present in `gateway/` and `invsys/` directories

3. **Local Testing** (optional but recommended)
   - Test the application locally using docker-compose
   - Verify all endpoints work correctly

***

## Task 1: Setting Up Your AWS Account and IAM

### Step 1.1: Create IAM User with Required Permissions

⚠️ **Caution:** Never use your root AWS account for daily operations. Always create IAM users.

💻 **Command (AWS Console):**

1. Navigate to **IAM Console** → **Users** → **Add users**
2. User name: `ecs-deployment-user`
3. Select **AWS credential type**:
   - ✅ Access key - Programmatic access
   - ✅ Password - AWS Management Console access
4. Click **Next: Permissions**

### Step 1.2: Attach Required Policies

Attach the following AWS managed policies:

- ✅ `AmazonECS_FullAccess`
- ✅ `AmazonEC2ContainerRegistryFullAccess`
- ✅ `AmazonEC2FullAccess`
- ✅ `ElasticLoadBalancingFullAccess`
- ✅ `IAMReadOnlyAccess`

💡 **Note:** For production environments, create custom policies with least privilege access. These broad permissions are used for learning purposes.

### Step 1.3: Save Credentials

📋 **Expected output:**

```
Access key ID: AKIAIOSFODNN7EXAMPLE
Secret access key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

⚠️ **Caution:** Save these credentials securely. You cannot retrieve the secret access key later.

### Step 1.4: Create Service Roles

💻 **Command (AWS Console):**

1. Navigate to **IAM** → **Roles** → **Create role**
2. Select **AWS service** → **Elastic Container Service**
3. Choose **Elastic Container Service Task** as use case
4. Attach policies:
   - `AmazonECSTaskExecutionRolePolicy`
   - `CloudWatchLogsFullAccess`
5. Role name: `ecsTaskExecutionRole`
6. Create role

Repeat for ECS service role:
- Use case: **Elastic Container Service** → **Elastic Container Service**
- Policy: `AmazonEC2ContainerServiceRole`
- Role name: `ecsServiceRole`

✅ **Task complete!** IAM setup is ready.

***

## Task 2: Creating an EC2 Command Host

We'll create an EC2 instance to build and push Docker images to ECR.

### Step 2.1: Launch EC2 Instance

💻 **Command (AWS Console):**

1. Navigate to **EC2 Console** → **Launch Instance**
2. **Name**: `Docker-Build-Host`
3. **AMI**: Amazon Linux 2023 AMI (Free tier eligible)
4. **Instance type**: `t2.micro` (or `t3.micro` if available in your region)
5. **Key pair**: Create new key pair
   - Name: `docker-build-key`
   - Type: RSA
   - Format: `.pem` (macOS/Linux) or `.ppk` (Windows/PuTTY)
   - Download and save securely
6. **Network settings**:
   - ✅ Allow SSH traffic from: My IP
   - ✅ Allow HTTP traffic from the internet
   - ✅ Allow HTTPS traffic from the internet
7. **Storage**: 8 GB gp3 (free tier eligible)
8. Click **Launch instance**

💡 **Note:** The instance will take 2-3 minutes to initialize.

### Step 2.2: Connect to EC2 Instance

💻 **Command (Local Terminal):**

```bash
# Change permissions on your key file
chmod 400 docker-build-key.pem

# Get the public IP from EC2 console
export EC2_IP="<your-ec2-public-ip>"

# Connect via SSH
ssh -i docker-build-key.pem ec2-user@$EC2_IP
```

📋 **Expected output:**

```
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'

[ec2-user@ip-172-31-x-x ~]$
```

✅ **Task complete!** You're now connected to your build host.

***

## Task 3: Installing Docker and AWS CLI

### Step 3.1: Install Docker

💻 **Command:**

```bash
# Update system packages
sudo yum update -y

# Install Docker
sudo yum install docker -y

# Start Docker service
sudo systemctl start docker

# Enable Docker to start on boot
sudo systemctl enable docker

# Add ec2-user to docker group (avoid sudo)
sudo usermod -aG docker ec2-user

# Apply group changes
newgrp docker

# Verify Docker installation
docker --version
```

📋 **Expected output:**

```
Docker version 24.0.5, build ced0996
```

### Step 3.2: Install AWS CLI v2

💻 **Command:**

```bash
# Download AWS CLI installer
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Install unzip if needed
sudo yum install unzip -y

# Unzip and install
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version
```

📋 **Expected output:**

```
aws-cli/2.15.10 Python/3.11.6 Linux/6.1.72-96.166.amzn2023.x86_64 exe/x86_64.amzn.2023
```

### Step 3.3: Configure AWS CLI

💻 **Command:**

```bash
aws configure
```

When prompted, enter your IAM user credentials from Task 1:

```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-east-1
Default output format [None]: json
```

💡 **Note:** Choose a region close to you. Popular free tier regions: `us-east-1`, `us-west-2`, `eu-west-1`

### Step 3.4: Verify AWS Configuration

💻 **Command:**

```bash
# Test AWS CLI access
aws sts get-caller-identity
```

📋 **Expected output:**

```json
{
    "UserId": "AIDAI23HXX2LO3EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/ecs-deployment-user"
}
```

### Step 3.5: Set Environment Variables

💻 **Command:**

```bash
# Get account ID
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Get region
export AWS_REGION=$(aws configure get region)

# Persist to profile
echo "export ACCOUNT_ID=$ACCOUNT_ID" >> ~/.bashrc
echo "export AWS_REGION=$AWS_REGION" >> ~/.bashrc

# Verify
echo "Account ID: $ACCOUNT_ID"
echo "Region: $AWS_REGION"
```

✅ **Task complete!** Your build environment is ready.

***

## Task 4: Exploring the Application Structure

### Step 4.1: Transfer Application Code to EC2

💻 **Command (Local Terminal - new tab/window):**

```bash
# Assuming code is in ~/aws-flask-microservices/AWS_deployment/Deploy-to-buillder-lab/

# Create archive
cd ~/IdeaProjects/aws-flask-microservices/AWS_deployment/Deploy-to-buillder-lab/
tar -czf flask-app.tar.gz gateway/ invsys/

# Transfer to EC2
scp -i docker-build-key.pem flask-app.tar.gz ec2-user@$EC2_IP:~/

# Or use git clone if code is in repository
```

💻 **Command (EC2 Terminal):**

```bash
# Extract code
tar -xzf flask-app.tar.gz

# Verify structure
tree -L 2
```

📋 **Expected output:**

```
.
├── gateway
│   ├── Dockerfile
│   ├── application.py
│   └── requirements.txt
└── invsys
    ├── Dockerfile
    ├── api.py
    ├── dal.py
    └── requirements.txt
```

### Step 4.2: Review Inventory System (invsys) Dockerfile

💻 **Command:**

```bash
cd ~/invsys
cat Dockerfile
```

📋 **Expected output:**

```dockerfile
FROM python:3.10
COPY requirements.txt /
RUN pip install -r /requirements.txt
COPY . /app
WORKDIR /app
EXPOSE 5000
CMD [ "python", "api.py" ]
```

💡 **Note:** This Dockerfile:
- Uses Python 3.10 base image
- Installs dependencies from requirements.txt
- Exposes port 5000 for the inventory API
- Runs api.py as the main application

### Step 4.3: Review Gateway Dockerfile

💻 **Command:**

```bash
cd ~/gateway
cat Dockerfile
```

📋 **Expected output:**

```dockerfile
FROM python:3.10
COPY requirements.txt /
RUN pip install -r /requirements.txt
COPY . /app
WORKDIR /app
EXPOSE 5001
CMD [ "python", "application.py" ]
```

💡 **Note:** Gateway uses port 5001 and forwards requests to invsys on port 5000.

### Step 4.4: Understanding Service Communication

The gateway service communicates with inventory system using internal service discovery:

```python
# From gateway/application.py
response = requests.get('http://invsys:5000/items')
```

In ECS, we'll configure this using:
- **Bridge networking** with port mappings, OR
- **Service discovery** with AWS Cloud Map

✅ **Task complete!** You understand the application architecture.

***

## Task 5: Building the Inventory System (invsys) Container

### Step 5.1: Build the Inventory System Image

💻 **Command:**

```bash
cd ~/invsys

# Build the Docker image
docker build -t flask-inventory/invsys:latest .
```

📋 **Expected output:**

```
[+] Building 45.2s (10/10) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [internal] load metadata for docker.io/library/python:3.10
 => [1/5] FROM docker.io/library/python:3.10@sha256:abc123...
 => [internal] load build context
 => [2/5] COPY requirements.txt /
 => [3/5] RUN pip install -r /requirements.txt
 => [4/5] COPY . /app
 => [5/5] WORKDIR /app
 => exporting to image
 => => naming to docker.io/flask-inventory/invsys:latest
```

### Step 5.2: Verify Image Creation

💻 **Command:**

```bash
docker images | grep invsys
```

📋 **Expected output:**

```
flask-inventory/invsys   latest   a1b2c3d4e5f6   2 minutes ago   1.02GB
```

### Step 5.3: Test the Container Locally

💻 **Command:**

```bash
# Run container in background
docker run -d -p 5000:5000 --name invsys-test flask-inventory/invsys:latest

# Check if container is running
docker ps

# Test the API
curl http://localhost:5000/items
```

📋 **Expected output:**

```json
{
  "devices": []
}
```

💻 **Command (Add test data):**

```bash
# POST a new device
curl -X POST http://localhost:5000/items \
  -H "Content-Type: application/json" \
  -d '{
    "id": "device-001",
    "name": "Test Device",
    "type": "sensor",
    "location": "Office-A"
  }'

# Verify
curl http://localhost:5000/items
```

📋 **Expected output:**

```json
{
  "devices": [
    {
      "id": "device-001",
      "name": "Test Device",
      "type": "sensor",
      "location": "Office-A"
    }
  ]
}
```

### Step 5.4: Stop Test Container

💻 **Command:**

```bash
docker stop invsys-test
docker rm invsys-test
```

✅ **Task complete!** Inventory system container is built and tested.

***

## Task 6: Building the Gateway Container

### Step 6.1: Build Gateway Image

💻 **Command:**

```bash
cd ~/gateway

# Build the Docker image
docker build -t flask-inventory/gateway:latest .
```

📋 **Expected output:**

```
[+] Building 42.8s (10/10) FINISHED
 => exporting to image
 => => naming to docker.io/flask-inventory/gateway:latest
```

### Step 6.2: Test Gateway with Docker Network

For local testing, we need both containers on the same network:

💻 **Command:**

```bash
# Create a custom bridge network
docker network create flask-network

# Run invsys on the network
docker run -d --network flask-network --name invsys flask-inventory/invsys:latest

# Run gateway on the network
docker run -d --network flask-network --name gateway -p 5001:5001 flask-inventory/gateway:latest

# Check both containers
docker ps
```

📋 **Expected output:**

```
CONTAINER ID   IMAGE                       PORTS                    NAMES
abc123def456   flask-inventory/gateway     0.0.0.0:5001->5001/tcp   gateway
def789ghi012   flask-inventory/invsys                               invsys
```

### Step 6.3: Test Gateway Routing

💻 **Command:**

```bash
# Test through gateway
curl http://localhost:5001/

# Test items endpoint through gateway
curl http://localhost:5001/items

# Add item through gateway
curl -X POST http://localhost:5001/items \
  -H "Content-Type: application/json" \
  -d '{
    "id": "device-002",
    "name": "Gateway Test Device",
    "type": "actuator",
    "location": "Lab-B"
  }'

# Verify
curl http://localhost:5001/items
```

📋 **Expected output:**

```
Hello from Gateway!

{"devices":[]}

{"message":"Device added successfully"}

{"devices":[{"id":"device-002","name":"Gateway Test Device","type":"actuator","location":"Lab-B"}]}
```

### Step 6.4: Clean Up Test Containers

💻 **Command:**

```bash
docker stop gateway invsys
docker rm gateway invsys
docker network rm flask-network
```

✅ **Task complete!** Both containers are built and verified.

***

## Task 7: Setting Up Amazon ECR Repositories

Amazon ECR (Elastic Container Registry) is AWS's managed Docker registry.

### Step 7.1: Create ECR Repository for Inventory System

💻 **Command:**

```bash
# Create repository
aws ecr create-repository \
  --repository-name flask-inventory/invsys \
  --image-tag-mutability IMMUTABLE \
  --region $AWS_REGION

# Save the repository URI
export INVSYS_REPO_URI=$(aws ecr describe-repositories \
  --repository-names flask-inventory/invsys \
  --query 'repositories[0].repositoryUri' \
  --output text)

echo "Inventory System Repository: $INVSYS_REPO_URI"
```

📋 **Expected output:**

```json
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-east-1:123456789012:repository/flask-inventory/invsys",
        "registryId": "123456789012",
        "repositoryName": "flask-inventory/invsys",
        "repositoryUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/flask-inventory/invsys",
        "createdAt": "2025-01-15T10:30:00+00:00",
        "imageTagMutability": "IMMUTABLE"
    }
}

Inventory System Repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/flask-inventory/invsys
```

### Step 7.2: Create ECR Repository for Gateway

💻 **Command:**

```bash
# Create repository
aws ecr create-repository \
  --repository-name flask-inventory/gateway \
  --image-tag-mutability IMMUTABLE \
  --region $AWS_REGION

# Save the repository URI
export GATEWAY_REPO_URI=$(aws ecr describe-repositories \
  --repository-names flask-inventory/gateway \
  --query 'repositories[0].repositoryUri' \
  --output text)

echo "Gateway Repository: $GATEWAY_REPO_URI"
```

### Step 7.3: Persist Environment Variables

💻 **Command:**

```bash
# Add to bash profile
echo "export INVSYS_REPO_URI=$INVSYS_REPO_URI" >> ~/.bashrc
echo "export GATEWAY_REPO_URI=$GATEWAY_REPO_URI" >> ~/.bashrc

# Verify
echo "Repositories created:"
echo "  - invsys: $INVSYS_REPO_URI"
echo "  - gateway: $GATEWAY_REPO_URI"
```

💡 **Note:** ECR charges $0.10 per GB-month for storage. Our images (~2 GB total) stay well within the 500 MB free tier for the first year if we keep only latest versions.

✅ **Task complete!** ECR repositories are ready.

***

## Task 8: Pushing Images to ECR

### Step 8.1: Authenticate Docker with ECR

💻 **Command:**

```bash
# Get ECR login token and authenticate
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin \
  $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

📋 **Expected output:**

```
Login Succeeded
```

💡 **Note:** This login token is valid for 12 hours. Re-run this command if you get authentication errors.

### Step 8.2: Tag Images for ECR

💻 **Command:**

```bash
# Tag inventory system image
docker tag flask-inventory/invsys:latest $INVSYS_REPO_URI:latest
docker tag flask-inventory/invsys:latest $INVSYS_REPO_URI:v1.0.0

# Tag gateway image
docker tag flask-inventory/gateway:latest $GATEWAY_REPO_URI:latest
docker tag flask-inventory/gateway:latest $GATEWAY_REPO_URI:v1.0.0

# Verify tags
docker images | grep flask-inventory
```

📋 **Expected output:**

```
flask-inventory/invsys    latest    a1b2c3d4e5f6   30 minutes ago   1.02GB
123...ecr.../invsys       latest    a1b2c3d4e5f6   30 minutes ago   1.02GB
123...ecr.../invsys       v1.0.0    a1b2c3d4e5f6   30 minutes ago   1.02GB
flask-inventory/gateway   latest    f6e5d4c3b2a1   25 minutes ago   1.01GB
123...ecr.../gateway      latest    f6e5d4c3b2a1   25 minutes ago   1.01GB
123...ecr.../gateway      v1.0.0    f6e5d4c3b2a1   25 minutes ago   1.01GB
```

### Step 8.3: Push Inventory System Image

💻 **Command:**

```bash
# Push both tags
docker push $INVSYS_REPO_URI:latest
docker push $INVSYS_REPO_URI:v1.0.0
```

📋 **Expected output:**

```
The push refers to repository [123456789012.dkr.ecr.us-east-1.amazonaws.com/flask-inventory/invsys]
5f70bf18a086: Pushed
d7e2a3f1c8b9: Pushed
e8f9a1b2c3d4: Pushed
...
latest: digest: sha256:abc123... size: 2841
v1.0.0: digest: sha256:abc123... size: 2841
```

💡 **Note:** First push will take 5-10 minutes depending on internet speed. Subsequent pushes are faster due to layer caching.

### Step 8.4: Push Gateway Image

💻 **Command:**

```bash
# Push both tags
docker push $GATEWAY_REPO_URI:latest
docker push $GATEWAY_REPO_URI:v1.0.0
```

### Step 8.5: Verify Images in ECR

💻 **Command:**

```bash
# List images in invsys repository
aws ecr describe-images \
  --repository-name flask-inventory/invsys \
  --query 'imageDetails[*].[imageTags[0],imagePushedAt,imageSizeInBytes]' \
  --output table

# List images in gateway repository
aws ecr describe-images \
  --repository-name flask-inventory/gateway \
  --query 'imageDetails[*].[imageTags[0],imagePushedAt,imageSizeInBytes]' \
  --output table
```

📋 **Expected output:**

```
------------------------------------------------------------------------------------------------
|                                        DescribeImages                                        |
+------------+-------------------------------------------------------+-------------------------+
|  latest    |  2025-01-15T11:00:00.123000+00:00                    |  1020000000             |
|  v1.0.0    |  2025-01-15T11:00:00.123000+00:00                    |  1020000000             |
+------------+-------------------------------------------------------+-------------------------+
```

✅ **Task complete!** Images are pushed to ECR.

***

## Task 9: Creating an ECS Cluster

### Step 9.1: Create ECS Cluster with EC2 Launch Type

💻 **Command:**

```bash
# Create cluster
aws ecs create-cluster \
  --cluster-name flask-inventory-cluster \
  --region $AWS_REGION

# Save cluster ARN
export CLUSTER_ARN=$(aws ecs describe-clusters \
  --clusters flask-inventory-cluster \
  --query 'clusters[0].clusterArn' \
  --output text)

echo "Cluster ARN: $CLUSTER_ARN"
```

📋 **Expected output:**

```json
{
    "cluster": {
        "clusterArn": "arn:aws:ecs:us-east-1:123456789012:cluster/flask-inventory-cluster",
        "clusterName": "flask-inventory-cluster",
        "status": "ACTIVE",
        "registeredContainerInstancesCount": 0,
        "runningTasksCount": 0,
        "pendingTasksCount": 0,
        "activeServicesCount": 0
    }
}
```

### Step 9.2: Create Launch Template for ECS Instances

First, we need to create a security group:

💻 **Command:**

```bash
# Get default VPC ID
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

echo "VPC ID: $VPC_ID"

# Create security group for ECS instances
aws ec2 create-security-group \
  --group-name ecs-instance-sg \
  --description "Security group for ECS container instances" \
  --vpc-id $VPC_ID

# Save security group ID
export ECS_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=ecs-instance-sg" \
  --query 'SecurityGroups[0].GroupId' \
  --output text)

echo "Security Group ID: $ECS_SG_ID"
```

### Step 9.3: Configure Security Group Rules

💻 **Command:**

```bash
# Allow HTTP from anywhere (for ALB to reach containers)
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow port 5000 (invsys)
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol tcp \
  --port 5000 \
  --cidr 0.0.0.0/0

# Allow port 5001 (gateway)
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol tcp \
  --port 5001 \
  --cidr 0.0.0.0/0

# Allow all traffic within security group (for service-to-service communication)
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol -1 \
  --source-group $ECS_SG_ID

# Allow SSH (optional, for debugging)
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

### Step 9.4: Create ECS-Optimized Launch Template

💻 **Command:**

```bash
# Get latest ECS-optimized AMI
export ECS_AMI=$(aws ssm get-parameters \
  --names /aws/service/ecs/optimized-ami/amazon-linux-2023/recommended/image_id \
  --query 'Parameters[0].Value' \
  --output text)

echo "ECS AMI: $ECS_AMI"

# Create user data script for ECS agent
cat > ecs-userdata.txt << 'EOF'
#!/bin/bash
echo ECS_CLUSTER=flask-inventory-cluster >> /etc/ecs/ecs.config
echo ECS_ENABLE_TASK_IAM_ROLE=true >> /etc/ecs/ecs.config
echo ECS_ENABLE_TASK_IAM_ROLE_NETWORK_HOST=true >> /etc/ecs/ecs.config
EOF

# Base64 encode user data
export USER_DATA=$(base64 -w 0 ecs-userdata.txt)

# Create IAM instance profile for ECS instances (if not exists)
aws iam create-role \
  --role-name ecsInstanceRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }' 2>/dev/null || true

# Attach required policies
aws iam attach-role-policy \
  --role-name ecsInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role

# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name ecsInstanceRole 2>/dev/null || true

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name ecsInstanceRole \
  --role-name ecsInstanceRole 2>/dev/null || true

# Wait for profile to be ready
sleep 10
```

### Step 9.5: Launch EC2 Instances for ECS

💻 **Command:**

```bash
# Launch 2 t2.micro instances for HA
aws ec2 run-instances \
  --image-id $ECS_AMI \
  --count 2 \
  --instance-type t2.micro \
  --key-name docker-build-key \
  --security-group-ids $ECS_SG_ID \
  --iam-instance-profile Name=ecsInstanceRole \
  --user-data file://ecs-userdata.txt \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ECS-Instance},{Key=Cluster,Value=flask-inventory-cluster}]' \
  --region $AWS_REGION
```

📋 **Expected output:**

```json
{
    "Instances": [
        {
            "InstanceId": "i-0123456789abcdef0",
            "InstanceType": "t2.micro",
            "State": {"Name": "pending"}
        }
    ]
}
```

### Step 9.6: Verify ECS Cluster Instances

Wait 2-3 minutes for instances to register with the cluster:

💻 **Command:**

```bash
# Check cluster status
aws ecs describe-clusters \
  --clusters flask-inventory-cluster \
  --query 'clusters[0].[registeredContainerInstancesCount,status]' \
  --output table

# List container instances
aws ecs list-container-instances \
  --cluster flask-inventory-cluster
```

📋 **Expected output (after instances register):**

```
-----------------------------
|    DescribeClusters       |
+-----+--------------------+
|  2  |  ACTIVE            |
+-----+--------------------+

{
    "containerInstanceArns": [
        "arn:aws:ecs:us-east-1:123456789012:container-instance/flask-inventory-cluster/abc123...",
        "arn:aws:ecs:us-east-1:123456789012:container-instance/flask-inventory-cluster/def456..."
    ]
}
```

💡 **Note:** If instances don't register within 5 minutes, check:
- ECS agent is running on instances
- IAM instance profile is attached
- Security group allows outbound traffic

✅ **Task complete!** ECS cluster with EC2 instances is ready.

***

## Task 10: Configuring Application Load Balancer

### Step 10.1: Create ALB

💻 **Command:**

```bash
# Get default subnets (need at least 2 for ALB)
export SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[0:2].SubnetId' \
  --output text | tr '\t' ' ')

echo "Subnets: $SUBNET_IDS"

# Create security group for ALB
aws ec2 create-security-group \
  --group-name alb-sg \
  --description "Security group for Application Load Balancer" \
  --vpc-id $VPC_ID

export ALB_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=alb-sg" \
  --query 'SecurityGroups[0].GroupId' \
  --output text)

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Create ALB
aws elbv2 create-load-balancer \
  --name flask-inventory-alb \
  --subnets $SUBNET_IDS \
  --security-groups $ALB_SG_ID \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4

# Get ALB ARN and DNS
export ALB_ARN=$(aws elbv2 describe-load-balancers \
  --names flask-inventory-alb \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

export ALB_DNS=$(aws elbv2 describe-load-balancers \
  --names flask-inventory-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

echo "ALB ARN: $ALB_ARN"
echo "ALB DNS: $ALB_DNS"
```

### Step 10.2: Create Target Group for Gateway Service

💻 **Command:**

```bash
# Create target group for gateway (port 5001)
aws elbv2 create-target-group \
  --name gateway-tg \
  --protocol HTTP \
  --port 5001 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-enabled \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Get target group ARN
export GATEWAY_TG_ARN=$(aws elbv2 describe-target-groups \
  --names gateway-tg \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

echo "Gateway Target Group ARN: $GATEWAY_TG_ARN"
```

### Step 10.3: Create Listener Rules

💻 **Command:**

```bash
# Create listener on port 80 (forwards all traffic to gateway)
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$GATEWAY_TG_ARN

# Get listener ARN
export LISTENER_ARN=$(aws elbv2 describe-listeners \
  --load-balancer-arn $ALB_ARN \
  --query 'Listeners[0].ListenerArn' \
  --output text)

echo "Listener ARN: $LISTENER_ARN"
```

### Step 10.4: Update ECS Security Group for ALB

💻 **Command:**

```bash
# Allow traffic from ALB to ECS instances
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol tcp \
  --port 5001 \
  --source-group $ALB_SG_ID

aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol tcp \
  --port 5000 \
  --source-group $ALB_SG_ID
```

### Step 10.5: Save Environment Variables

💻 **Command:**

```bash
# Persist to profile
cat >> ~/.bashrc << EOF
export VPC_ID=$VPC_ID
export ECS_SG_ID=$ECS_SG_ID
export ALB_SG_ID=$ALB_SG_ID
export ALB_ARN=$ALB_ARN
export ALB_DNS=$ALB_DNS
export GATEWAY_TG_ARN=$GATEWAY_TG_ARN
export LISTENER_ARN=$LISTENER_ARN
EOF

echo "✅ ALB configuration complete!"
echo "🌐 Access your application at: http://$ALB_DNS"
```

✅ **Task complete!** Load balancer is configured.

***

## Task 11: Creating ECS Task Definitions

Task definitions are blueprints for running containers in ECS.

### Step 11.1: Create Task Definition for Inventory System

💻 **Command:**

```bash
# Create task definition JSON
cat > ~/invsys-task-def.json << EOF
{
  "family": "flask-inventory-invsys",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "invsys",
      "image": "${INVSYS_REPO_URI}:latest",
      "cpu": 256,
      "memory": 512,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 5000,
          "hostPort": 5000,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/flask-inventory-invsys",
          "awslogs-region": "${AWS_REGION}",
          "awslogs-stream-prefix": "ecs",
          "awslogs-create-group": "true"
        }
      }
    }
  ]
}
EOF

# Register task definition
aws ecs register-task-definition \
  --cli-input-json file://~/invsys-task-def.json

# Get task definition ARN
export INVSYS_TASK_DEF=$(aws ecs describe-task-definition \
  --task-definition flask-inventory-invsys \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text)

echo "Inventory System Task Definition: $INVSYS_TASK_DEF"
```

📋 **Expected output:**

```json
{
    "taskDefinition": {
        "taskDefinitionArn": "arn:aws:ecs:us-east-1:123456789012:task-definition/flask-inventory-invsys:1",
        "family": "flask-inventory-invsys",
        "revision": 1,
        "status": "ACTIVE"
    }
}
```

### Step 11.2: Create Task Definition for Gateway

For the gateway, we need to configure it to communicate with invsys using ECS service discovery or by using the private IP.

💻 **Command:**

```bash
# Create task definition JSON
cat > ~/gateway-task-def.json << EOF
{
  "family": "flask-inventory-gateway",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "gateway",
      "image": "${GATEWAY_REPO_URI}:latest",
      "cpu": 256,
      "memory": 512,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 5001,
          "hostPort": 5001,
          "protocol": "tcp"
        }
      ],
      "links": ["invsys"],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/flask-inventory-gateway",
          "awslogs-region": "${AWS_REGION}",
          "awslogs-stream-prefix": "ecs",
          "awslogs-create-group": "true"
        }
      }
    }
  ]
}
EOF

# Register task definition
aws ecs register-task-definition \
  --cli-input-json file://~/gateway-task-def.json

# Get task definition ARN
export GATEWAY_TASK_DEF=$(aws ecs describe-task-definition \
  --task-definition flask-inventory-gateway \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text)

echo "Gateway Task Definition: $GATEWAY_TASK_DEF"
```

💡 **Note:** The `links` parameter creates DNS resolution between containers on the same ECS instance. For production, consider using AWS Cloud Map for service discovery.

✅ **Task complete!** Task definitions are registered.

***

## Task 12: Deploying Services to ECS

### Step 12.1: Deploy Inventory System Service

💻 **Command:**

```bash
# Create service
aws ecs create-service \
  --cluster flask-inventory-cluster \
  --service-name invsys-service \
  --task-definition flask-inventory-invsys \
  --desired-count 2 \
  --launch-type EC2 \
  --scheduling-strategy REPLICA \
  --deployment-configuration maximumPercent=200,minimumHealthyPercent=50

# Wait for service to stabilize
echo "⏳ Waiting for invsys service to be stable (this may take 2-3 minutes)..."
aws ecs wait services-stable \
  --cluster flask-inventory-cluster \
  --services invsys-service

echo "✅ Inventory system service is running!"
```

📋 **Expected output:**

```json
{
    "service": {
        "serviceArn": "arn:aws:ecs:us-east-1:123456789012:service/flask-inventory-cluster/invsys-service",
        "serviceName": "invsys-service",
        "clusterArn": "arn:aws:ecs:us-east-1:123456789012:cluster/flask-inventory-cluster",
        "status": "ACTIVE",
        "desiredCount": 2,
        "runningCount": 0,
        "pendingCount": 2
    }
}
```

### Step 12.2: Verify Inventory System Deployment

💻 **Command:**

```bash
# Check service status
aws ecs describe-services \
  --cluster flask-inventory-cluster \
  --services invsys-service \
  --query 'services[0].[serviceName,status,runningCount,desiredCount]' \
  --output table

# List running tasks
aws ecs list-tasks \
  --cluster flask-inventory-cluster \
  --service-name invsys-service
```

📋 **Expected output:**

```
------------------------------------------------------
|              DescribeServices                      |
+-------------------+----------+-------+-------------+
|  invsys-service   |  ACTIVE  |  2    |  2          |
+-------------------+----------+-------+-------------+
```

### Step 12.3: Deploy Gateway Service with Load Balancer

💻 **Command:**

```bash
# Create service with ALB integration
aws ecs create-service \
  --cluster flask-inventory-cluster \
  --service-name gateway-service \
  --task-definition flask-inventory-gateway \
  --desired-count 2 \
  --launch-type EC2 \
  --scheduling-strategy REPLICA \
  --load-balancers targetGroupArn=$GATEWAY_TG_ARN,containerName=gateway,containerPort=5001 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/ecsServiceRole \
  --deployment-configuration maximumPercent=200,minimumHealthyPercent=50

# Wait for service to stabilize
echo "⏳ Waiting for gateway service to be stable (this may take 3-5 minutes)..."
aws ecs wait services-stable \
  --cluster flask-inventory-cluster \
  --services gateway-service

echo "✅ Gateway service is running!"
```

### Step 12.4: Verify Gateway Deployment

💻 **Command:**

```bash
# Check service status
aws ecs describe-services \
  --cluster flask-inventory-cluster \
  --services gateway-service \
  --query 'services[0].[serviceName,status,runningCount,desiredCount]' \
  --output table

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn $GATEWAY_TG_ARN \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]' \
  --output table
```

📋 **Expected output:**

```
------------------------------------------------------
|              DescribeServices                      |
+-------------------+----------+-------+-------------+
|  gateway-service  |  ACTIVE  |  2    |  2          |
+-------------------+----------+-------+-------------+

-------------------------------------------------
|          DescribeTargetHealth                 |
+----------------------+------------------------+
|  i-0123456789...    |  healthy               |
|  i-abcdef0123...    |  healthy               |
+----------------------+------------------------+
```

🔧 **Troubleshooting:** If targets are unhealthy:
- Check CloudWatch logs: `aws logs tail /ecs/flask-inventory-gateway --follow`
- Verify security group rules allow ALB → ECS communication
- Check container is listening on correct port

✅ **Task complete!** Services are deployed and running.

***

## Task 13: Testing Your Deployment

### Step 13.1: Test Gateway Health Check

💻 **Command:**

```bash
# Test ALB endpoint
curl http://$ALB_DNS/

# Should see: "Hello from Gateway!"
```

📋 **Expected output:**

```
Hello from Gateway!
```

### Step 13.2: Test Inventory API - GET Items

💻 **Command:**

```bash
# Get all devices (should be empty initially)
curl http://$ALB_DNS/items

# Pretty print JSON
curl -s http://$ALB_DNS/items | jq .
```

📋 **Expected output:**

```json
{
  "devices": []
}
```

### Step 13.3: Test Inventory API - POST Device

💻 **Command:**

```bash
# Add a new device
curl -X POST http://$ALB_DNS/items \
  -H "Content-Type: application/json" \
  -d '{
    "id": "aws-device-001",
    "name": "Production Sensor",
    "type": "temperature",
    "location": "DataCenter-US-East"
  }'

# Add another device
curl -X POST http://$ALB_DNS/items \
  -H "Content-Type: application/json" \
  -d '{
    "id": "aws-device-002",
    "name": "Humidity Monitor",
    "type": "humidity",
    "location": "DataCenter-US-West"
  }'
```

📋 **Expected output:**

```json
{
  "message": "Device added successfully"
}
```

### Step 13.4: Test Inventory API - GET Specific Device

💻 **Command:**

```bash
# Get specific device
curl http://$ALB_DNS/items/aws-device-001 | jq .

# Get all devices
curl http://$ALB_DNS/items | jq .
```

📋 **Expected output:**

```json
{
  "id": "aws-device-001",
  "name": "Production Sensor",
  "type": "temperature",
  "location": "DataCenter-US-East"
}

{
  "devices": [
    {
      "id": "aws-device-001",
      "name": "Production Sensor",
      "type": "temperature",
      "location": "DataCenter-US-East"
    },
    {
      "id": "aws-device-002",
      "name": "Humidity Monitor",
      "type": "humidity",
      "location": "DataCenter-US-West"
    }
  ]
}
```

### Step 13.5: Test Inventory API - UPDATE Device

💻 **Command:**

```bash
# Update device
curl -X PUT http://$ALB_DNS/items/aws-device-001 \
  -H "Content-Type: application/json" \
  -d '{
    "id": "aws-device-001",
    "name": "Production Sensor - Updated",
    "type": "temperature",
    "location": "DataCenter-US-East-Updated"
  }'

# Verify update
curl http://$ALB_DNS/items/aws-device-001 | jq .
```

📋 **Expected output:**

```json
{
  "message": "Device updated successfully"
}

{
  "id": "aws-device-001",
  "name": "Production Sensor - Updated",
  "type": "temperature",
  "location": "DataCenter-US-East-Updated"
}
```

### Step 13.6: Test Inventory API - DELETE Device

💻 **Command:**

```bash
# Delete device
curl -X DELETE http://$ALB_DNS/items/aws-device-002

# Verify deletion
curl http://$ALB_DNS/items | jq .
```

📋 **Expected output:**

```json
{
  "message": "Device deleted successfully"
}

{
  "devices": [
    {
      "id": "aws-device-001",
      "name": "Production Sensor - Updated",
      "type": "temperature",
      "location": "DataCenter-US-East-Updated"
    }
  ]
}
```

### Step 13.7: Load Testing (Optional)

💻 **Command:**

```bash
# Simple load test with curl
for i in {1..50}; do
  curl -s http://$ALB_DNS/items > /dev/null &
done
wait

echo "✅ Load test complete"

# Check ECS task count (should still be 2)
aws ecs describe-services \
  --cluster flask-inventory-cluster \
  --services gateway-service \
  --query 'services[0].runningCount'
```

✅ **Task complete!** Application is fully functional on AWS!

***

## Task 14: Monitoring and Troubleshooting

### Step 14.1: View CloudWatch Logs

💻 **Command:**

```bash
# View gateway logs
aws logs tail /ecs/flask-inventory-gateway --follow

# In another terminal, view invsys logs
aws logs tail /ecs/flask-inventory-invsys --follow
```

📋 **Expected output (sample):**

```
2025-01-15T12:30:45.123Z [ecs-task-123] * Running on http://0.0.0.0:5001
2025-01-15T12:31:10.456Z [ecs-task-123] 10.0.1.5 - - [15/Jan/2025 12:31:10] "GET /items HTTP/1.1" 200 -
```

### Step 14.2: Check ECS Service Events

💻 **Command:**

```bash
# View gateway service events
aws ecs describe-services \
  --cluster flask-inventory-cluster \
  --services gateway-service \
  --query 'services[0].events[0:10]' \
  --output table

# View invsys service events
aws ecs describe-services \
  --cluster flask-inventory-cluster \
  --services invsys-service \
  --query 'services[0].events[0:10]' \
  --output table
```

### Step 14.3: Monitor ECS Cluster Metrics

💻 **Command (AWS Console):**

1. Navigate to **CloudWatch Console** → **Dashboards**
2. Create a new dashboard: `Flask-Inventory-Monitoring`
3. Add widgets:
   - **CPUUtilization** for ECS services
   - **MemoryUtilization** for ECS services
   - **RequestCount** for ALB
   - **TargetResponseTime** for ALB

### Step 14.4: Set Up CloudWatch Alarms (Optional)

💻 **Command:**

```bash
# Create alarm for high CPU usage
aws cloudwatch put-metric-alarm \
  --alarm-name ecs-high-cpu-gateway \
  --alarm-description "Alert when gateway CPU exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=ServiceName,Value=gateway-service Name=ClusterName,Value=flask-inventory-cluster

# Create alarm for unhealthy targets
aws cloudwatch put-metric-alarm \
  --alarm-name alb-unhealthy-targets \
  --alarm-description "Alert when targets are unhealthy" \
  --metric-name UnHealthyHostCount \
  --namespace AWS/ApplicationELB \
  --statistic Average \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 2 \
  --dimensions Name=TargetGroup,Value=$(echo $GATEWAY_TG_ARN | cut -d: -f6) Name=LoadBalancer,Value=$(echo $ALB_ARN | cut -d: -f6)
```

### Step 14.5: Common Issues and Solutions

🔧 **Issue: Tasks keep stopping and restarting**

Solution:
```bash
# Check task stopped reason
aws ecs describe-tasks \
  --cluster flask-inventory-cluster \
  --tasks $(aws ecs list-tasks --cluster flask-inventory-cluster --service-name gateway-service --query 'taskArns[0]' --output text) \
  --query 'tasks[0].stoppedReason'

# Common causes:
# - Out of memory → Increase memory in task definition
# - Application crash → Check CloudWatch logs
# - Health check failure → Adjust health check settings
```

🔧 **Issue: Targets are unhealthy in ALB**

Solution:
```bash
# Check health check configuration
aws elbv2 describe-target-health --target-group-arn $GATEWAY_TG_ARN

# Verify security group allows ALB → ECS
aws ec2 describe-security-groups --group-ids $ECS_SG_ID

# Check if container is listening on correct port
aws ecs describe-tasks --cluster flask-inventory-cluster --tasks <task-arn>
```

🔧 **Issue: Gateway cannot communicate with invsys**

Solution:
```bash
# Check that invsys service is running
aws ecs describe-services --cluster flask-inventory-cluster --services invsys-service

# Verify container links in task definition
aws ecs describe-task-definition --task-definition flask-inventory-gateway

# For bridge networking, both containers must be on same instance
# Consider using awsvpc network mode with service discovery for production
```

✅ **Task complete!** You can now monitor and troubleshoot your deployment.

***

## Cleanup Instructions

⚠️ **Important:** Follow these steps to avoid ongoing charges after testing.

### Step 1: Delete ECS Services

💻 **Command:**

```bash
# Update services to 0 desired count
aws ecs update-service \
  --cluster flask-inventory-cluster \
  --service gateway-service \
  --desired-count 0

aws ecs update-service \
  --cluster flask-inventory-cluster \
  --service invsys-service \
  --desired-count 0

# Wait for tasks to stop
sleep 30

# Delete services
aws ecs delete-service \
  --cluster flask-inventory-cluster \
  --service gateway-service \
  --force

aws ecs delete-service \
  --cluster flask-inventory-cluster \
  --service invsys-service \
  --force
```

### Step 2: Deregister ECS Container Instances

💻 **Command:**

```bash
# List container instances
CONTAINER_INSTANCES=$(aws ecs list-container-instances \
  --cluster flask-inventory-cluster \
  --query 'containerInstanceArns' \
  --output text)

# Deregister each instance
for instance in $CONTAINER_INSTANCES; do
  aws ecs deregister-container-instance \
    --cluster flask-inventory-cluster \
    --container-instance $instance \
    --force
done
```

### Step 3: Delete ECS Cluster

💻 **Command:**

```bash
aws ecs delete-cluster --cluster flask-inventory-cluster
```

### Step 4: Terminate EC2 Instances

💻 **Command:**

```bash
# Get instance IDs
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:Cluster,Values=flask-inventory-cluster" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text)

# Terminate instances
aws ec2 terminate-instances --instance-ids $INSTANCE_IDS

# Also terminate the build host
aws ec2 terminate-instances --instance-ids <build-host-instance-id>
```

### Step 5: Delete Load Balancer and Target Groups

💻 **Command:**

```bash
# Delete listener
aws elbv2 delete-listener --listener-arn $LISTENER_ARN

# Delete load balancer
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN

# Wait for ALB to be deleted (2-3 minutes)
sleep 180

# Delete target group
aws elbv2 delete-target-group --target-group-arn $GATEWAY_TG_ARN
```

### Step 6: Delete ECR Repositories

💻 **Command:**

```bash
# Delete images and repositories
aws ecr delete-repository \
  --repository-name flask-inventory/gateway \
  --force

aws ecr delete-repository \
  --repository-name flask-inventory/invsys \
  --force
```

### Step 7: Delete Security Groups

💻 **Command:**

```bash
# Wait for ENIs to be released (5 minutes)
sleep 300

# Delete security groups
aws ec2 delete-security-group --group-id $ALB_SG_ID
aws ec2 delete-security-group --group-id $ECS_SG_ID
```

### Step 8: Delete IAM Roles (Optional)

💻 **Command:**

```bash
# Remove instance profile
aws iam remove-role-from-instance-profile \
  --instance-profile-name ecsInstanceRole \
  --role-name ecsInstanceRole

aws iam delete-instance-profile --instance-profile-name ecsInstanceRole

# Detach policies
aws iam detach-role-policy \
  --role-name ecsInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role

aws iam detach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Delete roles
aws iam delete-role --role-name ecsInstanceRole
aws iam delete-role --role-name ecsTaskExecutionRole
aws iam delete-role --role-name ecsServiceRole
```

### Step 9: Delete CloudWatch Log Groups

💻 **Command:**

```bash
aws logs delete-log-group --log-group-name /ecs/flask-inventory-gateway
aws logs delete-log-group --log-group-name /ecs/flask-inventory-invsys
```

### Step 10: Verify Cleanup

💻 **Command:**

```bash
# Verify no running resources
aws ecs list-clusters
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"
aws elbv2 describe-load-balancers
aws ecr describe-repositories

# Check AWS Billing Dashboard for any charges
```

✅ **Cleanup complete!** All resources have been deleted.

***

## Cost Optimization Tips

💰 **Keep Costs Within Free Tier:**

1. **Monitor Usage**
   - Set up AWS Budgets with $1 threshold
   - Enable billing alerts
   - Check Free Tier usage dashboard weekly

2. **EC2 Optimization**
   - Use only t2.micro or t3.micro instances (free tier eligible)
   - Stop instances when not testing (ECS tasks will stop automatically)
   - Maximum 750 hours/month per instance = ~1 instance running 24/7

3. **ECR Optimization**
   - Delete old image versions (keep only latest and v1.0.0)
   - 500 MB free tier = enough for 2-3 optimized images
   - Use multi-stage builds to reduce image size

4. **Data Transfer**
   - Stay under 100 GB outbound/month (easily achievable for testing)
   - Use CloudFront CDN if serving static content (50 GB free tier)

5. **CloudWatch Logs**
   - Logs: 5 GB ingestion free tier
   - Set retention to 7 days for test environments
   - Delete log groups when done testing

6. **ALB Costs**
   - 750 hours/month free tier (1 ALB running 24/7)
   - After free tier: $0.0225/hour (~$16/month)
   - 15 LCUs free/month for processing

**Estimated Monthly Cost After Free Tier Expires:**
- 2x t2.micro EC2: ~$18
- ALB: ~$16
- ECR storage (500 MB): $0.05
- Data transfer: ~$1
- **Total: ~$35/month**

💡 **Best Practice:** Set up auto-stop schedules to run services only during business hours.

***

## Additional Resources

### AWS Documentation
- [Amazon ECS Developer Guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/)
- [Amazon ECR User Guide](https://docs.aws.amazon.com/AmazonECR/latest/userguide/)
- [Application Load Balancer Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)
- [AWS Free Tier](https://aws.amazon.com/free/)

### Docker Resources
- [Docker Documentation](https://docs.docker.com/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Security](https://docs.docker.com/engine/security/)

### Flask Resources
- [Flask Documentation](https://flask.palletsprojects.com/)
- [Flask Microservices Patterns](https://flask.palletsprojects.com/en/stable/patterns/)
- [RESTful API Design](https://restfulapi.net/)

### AWS CLI Reference
- [ECS CLI Commands](https://docs.aws.amazon.com/cli/latest/reference/ecs/)
- [ECR CLI Commands](https://docs.aws.amazon.com/cli/latest/reference/ecr/)
- [ELBv2 CLI Commands](https://docs.aws.amazon.com/cli/latest/reference/elbv2/)

### AWS Training
- [AWS Training and Certification](https://aws.amazon.com/training/)
- [AWS Skill Builder](https://skillbuilder.aws/)
- [ECS Workshop](https://ecsworkshop.com/)

### Community Support
- [AWS Forums](https://forums.aws.amazon.com/)
- [Stack Overflow - AWS Tags](https://stackoverflow.com/questions/tagged/amazon-web-services)
- [AWS Reddit Community](https://www.reddit.com/r/aws/)

***

## Conclusion

🎉 **Congratulations!** You have successfully:

- ✅ Built Docker images for a multi-service Flask application
- ✅ Pushed images to Amazon ECR
- ✅ Created and configured an ECS cluster with EC2 instances
- ✅ Deployed microservices with container orchestration
- ✅ Set up Application Load Balancer for traffic distribution
- ✅ Implemented monitoring and logging with CloudWatch
- ✅ Learned cost optimization strategies for AWS free tier

This deployment pattern is production-ready and can scale to handle thousands of requests. You've gained hands-on experience with:

- **Container Orchestration**: ECS task definitions, services, and clusters
- **Infrastructure as Code**: AWS CLI automation and configuration
- **Microservices Architecture**: Service-to-service communication
- **Load Balancing**: Traffic distribution and health checks
- **DevOps Practices**: CI/CD pipeline foundations


***

_Great work! Your feedback helps improve this guide. Please share your experience and suggestions._


