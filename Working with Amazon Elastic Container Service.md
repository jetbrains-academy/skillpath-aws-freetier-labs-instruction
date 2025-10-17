# Working with Amazon Elastic Container Service - Flask Microservices Edition

***

## Table of Contents

- [Lab Overview](#lab-overview)
- [Amazon ECS Architecture Overview](#amazon-ecs-architecture-overview)
- [Objectives](#objectives)
- [Technical Prerequisites](#technical-prerequisites)
- [Icon Key](#icon-key)
- [Lab Environment Setup](#lab-environment-setup)
- [Task 1: Understanding ECS Core Concepts](#task-1-understanding-ecs-core-concepts)
- [Task 2: Creating Task Definitions for Flask Services](#task-2-creating-task-definitions-for-flask-services)
- [Task 3: Deploying Inventory System Service](#task-3-deploying-inventory-system-service)
- [Task 4: Deploying Gateway Service with Service Discovery](#task-4-deploying-gateway-service-with-service-discovery)
- [Task 5: Updating Application - Rolling Deployments](#task-5-updating-application-rolling-deployments)
- [Task 6: Scaling Your Services](#task-6-scaling-your-services)
- [Task 7: Implementing Health Checks](#task-7-implementing-health-checks)
- [Task 8: Working with Service Auto Scaling](#task-8-working-with-service-auto-scaling)
- [Task 9: Monitoring with CloudWatch](#task-9-monitoring-with-cloudwatch)
- [Task 10: Understanding CloudFormation Templates (Optional)](#task-10-understanding-cloudformation-templates-optional)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)
- [Cleanup Instructions](#cleanup-instructions)
- [Knowledge Check Questions](#knowledge-check-questions)
- [Additional Resources](#additional-resources)

***

## Lab Overview

This lab guides you through working with **Amazon Elastic Container Service (ECS)** to deploy, manage, and scale a Flask-based microservices application. You'll learn production-ready container orchestration patterns using AWS free tier services.

### What You'll Build

A complete inventory management system consisting of:

- **Inventory System (invsys)**: RESTful API backend for device management
  - CRUD operations for devices
  - Data validation with Marshmallow
  - SQLite/in-memory storage

- **Gateway Service**: API gateway for routing and service aggregation
  - Request forwarding
  - Unified API endpoint
  - Service-to-service communication

### Why This Matters

Amazon ECS simplifies:
- **Container orchestration** without managing Kubernetes complexity
- **Automatic scaling** based on demand
- **Rolling deployments** with zero downtime
- **Service discovery** for microservices communication
- **Integration** with AWS services (ALB, CloudWatch, ECR)

***

## Amazon ECS Architecture Overview

### ECS Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Amazon ECS Service                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Provisioning Layer                      │  │
│  │  (AWS Console, CLI, CloudFormation, Terraform)       │  │
│  └─────────────────────────────────────────────────────┘  │
│                            ↓                                │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Controller Layer                        │  │
│  │  (Deploy/Manage applications, Services, Tasks)       │  │
│  └─────────────────────────────────────────────────────┘  │
│                            ↓                                │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Capacity Layer                          │  │
│  │  (EC2 Instances / AWS Fargate)                       │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Key ECS Terminology

| Term | Definition | Example |
|------|------------|---------|
| **Cluster** | Logical grouping of tasks/services | `flask-inventory-cluster` |
| **Task Definition** | Blueprint describing containers | JSON defining invsys container |
| **Task** | Running instance of task definition | Single invsys container instance |
| **Service** | Maintains desired number of tasks | Keep 2 gateway tasks running |
| **Container Instance** | EC2 instance running ECS agent | t2.micro with ECS agent |
| **Container** | Docker container from task definition | Flask app in httpd container |

### ECS Launch Types

| Launch Type | Description | Free Tier | Best For |
|-------------|-------------|-----------|----------|
| **EC2** | You manage EC2 instances | ✅ Yes (750 hrs/month t2.micro) | Learning, cost optimization |
| **Fargate** | Serverless, AWS manages infrastructure | ❌ No | Production, simplicity |

💡 **This guide uses EC2 launch type** to stay within free tier.

***

## Objectives

By completing this lab, you will be able to:

- ✅ **Create** ECS Task Definitions for Flask microservices
- ✅ **Deploy** multi-container applications to ECS Services
- ✅ **Configure** service discovery for inter-service communication
- ✅ **Update** applications with zero-downtime rolling deployments
- ✅ **Scale** services horizontally (add/remove containers)
- ✅ **Implement** health checks and monitoring
- ✅ **Troubleshoot** common ECS deployment issues
- ✅ **Understand** Infrastructure as Code with CloudFormation

***

## Technical Prerequisites

### Required Knowledge
- Docker containers and Dockerfile syntax
- Basic Flask/Python web applications
- REST API concepts (GET, POST, PUT, DELETE)
- Networking fundamentals (ports, load balancing)
- Linux command line basics

### Required Setup
- Completed AWS account setup from previous guide
- Flask application code (gateway + invsys)
- Docker images pushed to ECR
- Basic ECS infrastructure (can reuse from deployment guide)

### Recommended Background
- Completed: "Building and Deploying Containers Using Amazon ECS" guide
- Familiar with AWS Console navigation
- Understanding of microservices architecture patterns

***

## Icon Key

| Icon | Description |
|------|-------------|
| 💻 Command | Run this command in terminal |
| 📋 Expected output | Sample output you should see |
| 💡 Note | Important tips and guidance |
| ⚠️ Caution | Warnings about costs or critical steps |
| 📚 Learn more | Links to AWS documentation |
| ✅ Task complete | Checkpoint - you've finished this section |
| 🔧 Troubleshooting | Common issues and solutions |
| 🔄 Refresh | Refresh browser or re-run command |
| 🎯 Best Practice | Production-ready recommendations |

***

## Lab Environment Setup

### Prerequisites Check

Before starting, verify you have:

💻 **Command:**

```bash
# Verify AWS CLI is configured
aws sts get-caller-identity

# Verify Docker is installed
docker --version

# Check ECR repositories exist
aws ecr describe-repositories --query 'repositories[*].repositoryName'

# Verify ECS cluster exists (or create new one)
aws ecs describe-clusters --clusters flask-inventory-cluster
```

📋 **Expected output:**

```json
{
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/ecs-user"
}

Docker version 24.0.5, build ced0996

[
    "flask-inventory/gateway",
    "flask-inventory/invsys"
]

{
    "clusters": [{
        "clusterName": "flask-inventory-cluster",
        "status": "ACTIVE"
    }]
}
```

### Set Environment Variables

💻 **Command:**

```bash
# Set your variables (adjust as needed)
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION=$(aws configure get region)
export CLUSTER_NAME="flask-inventory-cluster"

# Get ECR repository URIs
export INVSYS_REPO_URI=$(aws ecr describe-repositories \
  --repository-names flask-inventory/invsys \
  --query 'repositories[0].repositoryUri' \
  --output text)

export GATEWAY_REPO_URI=$(aws ecr describe-repositories \
  --repository-names flask-inventory/gateway \
  --query 'repositories[0].repositoryUri' \
  --output text)

# Verify
echo "Account ID: $ACCOUNT_ID"
echo "Region: $AWS_REGION"
echo "Cluster: $CLUSTER_NAME"
echo "Invsys Image: $INVSYS_REPO_URI:latest"
echo "Gateway Image: $GATEWAY_REPO_URI:latest"
```

✅ **Setup complete!** Environment is ready.

***

## Task 1: Understanding ECS Core Concepts

Before diving into deployments, let's understand how ECS components work together.

### 1.1: Task Definitions - The Blueprint

A **Task Definition** is a JSON document describing containers that form your application.

**Key parameters:**
- `family`: Name grouping related task definition versions
- `containerDefinitions`: Array of container configurations
- `cpu` & `memory`: Resource allocation
- `networkMode`: How containers communicate
- `requiresCompatibilities`: Launch type (EC2 or Fargate)

💡 **Analogy:** Task Definition is like a Dockerfile for ECS - it defines what to run.

### 1.2: Tasks vs Services

| Aspect | Task | Service |
|--------|------|---------|
| **Definition** | Single running instance | Maintains desired count of tasks |
| **Lifecycle** | Runs once, then stops | Continuously running |
| **Use Case** | Batch jobs, one-time operations | Web servers, APIs |
| **Auto-recovery** | No | Yes - replaces failed tasks |
| **Scaling** | Manual | Automatic with auto-scaling |

### 1.3: Networking Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `bridge` | Docker bridge network | Simple deployments, port mapping |
| `awsvpc` | Each task gets ENI with private IP | Service discovery, Fargate |
| `host` | Uses host's network | High network performance |
| `none` | No external networking | Custom networking |

💡 **For this lab:** We'll use `bridge` mode (simpler) and `awsvpc` (production-ready).

### 1.4: Service Discovery

**Problem:** How does Gateway find Inventory System?

**Solutions:**

| Method | How It Works | Pros | Cons |
|--------|--------------|------|------|
| **Container Links** | Docker `--link` in bridge mode | Simple | Same host only |
| **AWS Cloud Map** | DNS-based service discovery | Dynamic, any host | More setup |
| **Environment Variables** | Hardcode service URLs | Quick for testing | Not dynamic |

📚 **Learn more:** [Amazon ECS Task Networking](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html)

✅ **Concepts understood!** Let's build task definitions.

***

## Task 2: Creating Task Definitions for Flask Services

### 2.1: Create Task Definition for Inventory System

We'll create a task definition that runs our invsys Flask API.

💻 **Command:**

```bash
# Create task definition JSON
cat > ~/invsys-task-definition.json << EOF
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
          "hostPort": 0,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "FLASK_ENV",
          "value": "production"
        },
        {
          "name": "PYTHONUNBUFFERED",
          "value": "1"
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
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:5000/items || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
EOF

cat ~/invsys-task-definition.json
```

💡 **Key Configuration Explained:**

- **`hostPort: 0`**: Dynamic port mapping - ECS assigns random host port
  - Allows multiple tasks on same EC2 instance
  - ALB automatically discovers the port
- **`essential: true`**: If this container stops, entire task stops
- **`PYTHONUNBUFFERED=1`**: See Python logs in real-time
- **`healthCheck`**: Container-level health monitoring
  - ECS stops/restarts unhealthy containers
  - ALB uses target group health checks separately

### 2.2: Register Inventory System Task Definition

💻 **Command:**

```bash
# Register task definition
aws ecs register-task-definition \
  --cli-input-json file://~/invsys-task-definition.json

# Verify registration
aws ecs describe-task-definition \
  --task-definition flask-inventory-invsys \
  --query 'taskDefinition.[family,revision,status]' \
  --output table
```

📋 **Expected output:**

```
------------------------------------------
|      DescribeTaskDefinition           |
+-------------------------+--------------+
|  flask-inventory-invsys |  1           |
|                         |  ACTIVE      |
+-------------------------+--------------+
```

### 2.3: Create Task Definition for Gateway Service

The gateway needs to communicate with invsys. We'll use **AWS Cloud Map** for service discovery.

💻 **Command:**

```bash
# Create task definition JSON
cat > ~/gateway-task-definition.json << EOF
{
  "family": "flask-inventory-gateway",
  "networkMode": "awsvpc",
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
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "FLASK_ENV",
          "value": "production"
        },
        {
          "name": "PYTHONUNBUFFERED",
          "value": "1"
        },
        {
          "name": "INVSYS_URL",
          "value": "http://invsys.local:5000"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/flask-inventory-gateway",
          "awslogs-region": "${AWS_REGION}",
          "awslogs-stream-prefix": "ecs",
          "awslogs-create-group": "true"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:5001/ || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
EOF

cat ~/gateway-task-definition.json
```

💡 **Key Differences from Invsys:**

- **`networkMode: awsvpc`**: Each task gets its own network interface
  - Required for service discovery
  - Better security isolation
  - No port conflicts
- **`INVSYS_URL`**: Will use Cloud Map DNS for service discovery
  - `invsys.local` resolves to invsys service tasks

⚠️ **Important:** We need to update the gateway application code to use the `INVSYS_URL` environment variable instead of hardcoded `http://invsys:5000`.

### 2.4: Update Gateway Code for Service Discovery

💻 **Command:**

```bash
# Create updated gateway application
cat > ~/gateway-updated/application.py << 'EOF'
from flask import Flask, request, Response
import requests
import os

app = Flask(__name__)

# Get invsys URL from environment variable with fallback
INVSYS_URL = os.getenv('INVSYS_URL', 'http://invsys:5000')

@app.route("/")
def index():
    return 'Hello from Gateway!'

@app.route('/items', methods=['GET'])
@app.route('/items/<string:item_id>', methods=['GET'])
def get_devices(item_id=None):
    # Forward the request to the relevant endpoint in invsys
    if item_id:
        response = requests.get(f'{INVSYS_URL}/items/{item_id}')
    else:
        response = requests.get(f'{INVSYS_URL}/items')

    return Response(response.content, response.status_code)

@app.route('/items/<string:item_id>', methods=['DELETE'])
def delete_device(item_id):
    response = requests.delete(f'{INVSYS_URL}/items/{item_id}')
    return Response(response.content, response.status_code)

@app.route('/items', methods=['POST'])
def post_device():
    payload = request.get_json(force=True)
    response = requests.post(f'{INVSYS_URL}/items', json=payload)
    return Response(response.content, response.status_code)

@app.route('/items/<string:item_id>', methods=['PUT'])
def put_device(item_id):
    payload = request.get_json(force=True)
    response = requests.put(f'{INVSYS_URL}/items/{item_id}', json=payload)
    return Response(response.content, response.status_code)

@app.route('/health')
def health():
    return {'status': 'healthy'}, 200

if __name__ == "__main__":
    app.run("0.0.0.0", port=5001, debug=True)
EOF
```

💡 **Note:** For production use, rebuild the gateway Docker image with this updated code and push to ECR.

### 2.5: Register Gateway Task Definition

💻 **Command:**

```bash
# Register task definition
aws ecs register-task-definition \
  --cli-input-json file://~/gateway-task-definition.json

# Verify both task definitions
aws ecs list-task-definitions \
  --family-prefix flask-inventory \
  --query 'taskDefinitionArns' \
  --output table
```

📋 **Expected output:**

```
---------------------------------------------------------
|              ListTaskDefinitions                      |
+-------------------------------------------------------+
|  arn:aws:ecs:...:task-definition/flask-inventory-invsys:1    |
|  arn:aws:ecs:...:task-definition/flask-inventory-gateway:1   |
+-------------------------------------------------------+
```

✅ **Task complete!** Task definitions are registered.

***

## Task 3: Deploying Inventory System Service

### 3.1: Create Service Discovery Namespace

AWS Cloud Map provides DNS-based service discovery.

💻 **Command:**

```bash
# Create private DNS namespace
aws servicediscovery create-private-dns-namespace \
  --name local \
  --vpc $VPC_ID \
  --description "Service discovery for Flask inventory app"

# Wait for operation to complete
sleep 30

# Get namespace ID
export NAMESPACE_ID=$(aws servicediscovery list-namespaces \
  --filters Name=NAME,Values=local,Condition=EQ \
  --query 'Namespaces[0].Id' \
  --output text)

echo "Namespace ID: $NAMESPACE_ID"
```

📋 **Expected output:**

```json
{
    "OperationId": "abc123-def456-ghi789"
}

Namespace ID: ns-abcd1234efgh5678
```

### 3.2: Create Service Discovery Service for Invsys

💻 **Command:**

```bash
# Create service discovery service
aws servicediscovery create-service \
  --name invsys \
  --dns-config "NamespaceId=${NAMESPACE_ID},DnsRecords=[{Type=A,TTL=60}]" \
  --health-check-custom-config FailureThreshold=1 \
  --description "Service discovery for inventory system"

# Get service discovery service ID
export INVSYS_DISCOVERY_ARN=$(aws servicediscovery list-services \
  --filters Name=NAMESPACE_ID,Values=$NAMESPACE_ID \
  --query "Services[?Name=='invsys'].Arn" \
  --output text)

echo "Discovery Service ARN: $INVSYS_DISCOVERY_ARN"
```

💡 **What this does:**
- Creates DNS record `invsys.local` in private DNS namespace
- When gateway queries `invsys.local`, it resolves to invsys task IPs
- AWS updates DNS automatically as tasks start/stop

### 3.3: Create Target Group for Invsys (Optional - for ALB access)

💻 **Command:**

```bash
# Create target group
aws elbv2 create-target-group \
  --name invsys-tg \
  --protocol HTTP \
  --port 5000 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-enabled \
  --health-check-protocol HTTP \
  --health-check-path "/items" \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Get target group ARN
export INVSYS_TG_ARN=$(aws elbv2 describe-target-groups \
  --names invsys-tg \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

echo "Invsys Target Group ARN: $INVSYS_TG_ARN"
```

💡 **Note:** Using `target-type ip` because we're using `awsvpc` network mode.

### 3.4: Deploy Invsys Service with Service Discovery

💻 **Command:**

```bash
# Get subnet IDs for awsvpc mode
export SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].SubnetId' \
  --output text | tr '\t' ',')

echo "Subnets: $SUBNET_IDS"

# Create ECS service
aws ecs create-service \
  --cluster $CLUSTER_NAME \
  --service-name invsys-service \
  --task-definition flask-inventory-invsys:1 \
  --desired-count 2 \
  --launch-type EC2 \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[$ECS_SG_ID],assignPublicIp=DISABLED}" \
  --service-registries "registryArn=$INVSYS_DISCOVERY_ARN" \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=50" \
  --health-check-grace-period-seconds 60

# Wait for service to stabilize
echo "⏳ Waiting for invsys service to become stable (2-3 minutes)..."
aws ecs wait services-stable \
  --cluster $CLUSTER_NAME \
  --services invsys-service

echo "✅ Invsys service is running!"
```

📋 **Expected output:**

```json
{
    "service": {
        "serviceArn": "arn:aws:ecs:us-east-1:123456789012:service/flask-inventory-cluster/invsys-service",
        "serviceName": "invsys-service",
        "status": "ACTIVE",
        "desiredCount": 2,
        "runningCount": 0,
        "pendingCount": 2,
        "launchType": "EC2",
        "networkConfiguration": {
            "awsvpcConfiguration": {
                "subnets": ["subnet-abc123", "subnet-def456"],
                "securityGroups": ["sg-0123456789"],
                "assignPublicIp": "DISABLED"
            }
        }
    }
}
```

### 3.5: Verify Service Deployment

💻 **Command:**

```bash
# Check service status
aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services invsys-service \
  --query 'services[0].[serviceName,status,runningCount,desiredCount]' \
  --output table

# List running tasks
aws ecs list-tasks \
  --cluster $CLUSTER_NAME \
  --service-name invsys-service

# Check CloudWatch logs
aws logs tail /ecs/flask-inventory-invsys --follow
```

📋 **Expected output:**

```
---------------------------------------------------------
|              DescribeServices                         |
+------------------+----------+-------------+-----------+
|  invsys-service  |  ACTIVE  |  2          |  2        |
+------------------+----------+-------------+-----------+

{
    "taskArns": [
        "arn:aws:ecs:us-east-1:123456789012:task/flask-inventory-cluster/abc123...",
        "arn:aws:ecs:us-east-1:123456789012:task/flask-inventory-cluster/def456..."
    ]
}

2025-01-15T14:00:00.000Z * Running on http://0.0.0.0:5000
2025-01-15T14:00:01.000Z * Debug mode: off
```

### 3.6: Test Service Discovery

💻 **Command:**

```bash
# Query DNS from within VPC (from EC2 instance)
dig invsys.local

# Or use AWS CLI to check service instances
aws servicediscovery list-instances \
  --service-id $(echo $INVSYS_DISCOVERY_ARN | rev | cut -d'/' -f1 | rev)
```

📋 **Expected output:**

```
; <<>> DiG 9.16.1 <<>> invsys.local
;; ANSWER SECTION:
invsys.local.           60      IN      A       10.0.1.45
invsys.local.           60      IN      A       10.0.2.67

{
    "Instances": [
        {
            "Id": "abc123",
            "Attributes": {
                "AWS_INSTANCE_IPV4": "10.0.1.45"
            }
        },
        {
            "Id": "def456",
            "Attributes": {
                "AWS_INSTANCE_IPV4": "10.0.2.67"
            }
        }
    ]
}
```

✅ **Task complete!** Inventory service is deployed with service discovery.

***

## Task 4: Deploying Gateway Service with Service Discovery

### 4.1: Create Listener Rule for Gateway

💻 **Command:**

```bash
# Add listener rule to forward traffic to gateway
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 10 \
  --conditions Field=path-pattern,Values='/*' \
  --actions Type=forward,TargetGroupArn=$GATEWAY_TG_ARN
```

### 4.2: Deploy Gateway Service

💻 **Command:**

```bash
# Create service with ALB integration
aws ecs create-service \
  --cluster $CLUSTER_NAME \
  --service-name gateway-service \
  --task-definition flask-inventory-gateway:1 \
  --desired-count 2 \
  --launch-type EC2 \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[$ECS_SG_ID],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=$GATEWAY_TG_ARN,containerName=gateway,containerPort=5001" \
  --health-check-grace-period-seconds 60 \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=50"

# Wait for service to stabilize
echo "⏳ Waiting for gateway service to become stable (3-5 minutes)..."
aws ecs wait services-stable \
  --cluster $CLUSTER_NAME \
  --services gateway-service

echo "✅ Gateway service is running!"
```

### 4.3: Verify End-to-End Communication

💻 **Command:**

```bash
# Test gateway endpoint
curl http://$ALB_DNS/

# Test gateway → invsys communication
curl http://$ALB_DNS/items

# Add a device through gateway
curl -X POST http://$ALB_DNS/items \
  -H "Content-Type: application/json" \
  -d '{
    "id": "ecs-device-001",
    "name": "ECS Test Device",
    "location": "AWS Cloud",
    "status": "active"
  }'

# Verify
curl http://$ALB_DNS/items | jq .
```

📋 **Expected output:**

```
Hello from Gateway!

{"items":[]}

{"Posted a device":{"id":"ecs-device-001","name":"ECS Test Device","location":"AWS Cloud","status":"active"}}

{
  "items": [
    {
      "id": "ecs-device-001",
      "name": "ECS Test Device",
      "location": "AWS Cloud",
      "status": "active"
    }
  ]
}
```

✅ **Task complete!** Full stack is deployed and communicating!

***

## Task 5: Updating Application - Rolling Deployments

Learn how to deploy application updates with zero downtime.

### 5.1: Simulate Application Update

Let's add a version endpoint to the inventory system.

💻 **Command:**

```bash
# Create updated API code with version endpoint
cat > ~/invsys-v2/api.py << 'EOF'
from flask import Flask, request, jsonify
from marshmallow import Schema, fields, ValidationError, validates_schema
import dal

app = Flask(__name__)

# Version endpoint
@app.route('/version')
def version():
    return jsonify({"version": "2.0.0", "service": "inventory-system"}), 200

# ... rest of the code remains the same ...
# (Include all existing endpoints from original api.py)
EOF
```

### 5.2: Build and Push Updated Image

💻 **Command:**

```bash
# Build new version
cd ~/invsys-v2
docker build -t flask-inventory/invsys:v2.0.0 .

# Tag for ECR
docker tag flask-inventory/invsys:v2.0.0 $INVSYS_REPO_URI:v2.0.0
docker tag flask-inventory/invsys:v2.0.0 $INVSYS_REPO_URI:latest

# Push to ECR
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin \
  $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

docker push $INVSYS_REPO_URI:v2.0.0
docker push $INVSYS_REPO_URI:latest
```

### 5.3: Create New Task Definition Revision

💻 **Command:**

```bash
# Update task definition with new image version
cat > ~/invsys-task-definition-v2.json << EOF
{
  "family": "flask-inventory-invsys",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["EC2"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "invsys",
      "image": "${INVSYS_REPO_URI}:v2.0.0",
      "cpu": 256,
      "memory": 512,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 5000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "FLASK_ENV",
          "value": "production"
        },
        {
          "name": "APP_VERSION",
          "value": "2.0.0"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/flask-inventory-invsys",
          "awslogs-region": "${AWS_REGION}",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:5000/version || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
EOF

# Register new revision
aws ecs register-task-definition \
  --cli-input-json file://~/invsys-task-definition-v2.json

# Verify new revision
aws ecs describe-task-definition \
  --task-definition flask-inventory-invsys \
  --query 'taskDefinition.[family,revision,containerDefinitions[0].image]' \
  --output table
```

📋 **Expected output:**

```
-----------------------------------------------------------------------
|                     DescribeTaskDefinition                          |
+-------------------------+----------+----------------------------------+
|  flask-inventory-invsys |  2       |  123...ecr.../invsys:v2.0.0     |
+-------------------------+----------+----------------------------------+
```

### 5.4: Update Service with Rolling Deployment

💻 **Command:**

```bash
# Update service to use new task definition revision
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service invsys-service \
  --task-definition flask-inventory-invsys:2 \
  --force-new-deployment

# Monitor deployment progress
watch -n 5 "aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services invsys-service \
  --query 'services[0].deployments' \
  --output table"
```

📋 **Expected output (during deployment):**

```
-------------------------------------------------------------------
|                          Deployments                            |
+--------+----------+-----------+--------------+------------------+
| Status | Revision | Running   | Pending      | Desired          |
+--------+----------+-----------+--------------+------------------+
| PRIMARY| 2        | 1         | 1            | 2                |
| ACTIVE | 1        | 1         | 0            | 2                |
+--------+----------+-----------+--------------+------------------+
```

💡 **Rolling Deployment Behavior:**

1. **ECS starts new tasks** (revision 2) while old tasks (revision 1) still run
2. **Health checks pass** on new tasks
3. **ALB drains connections** from old tasks
4. **ECS stops old tasks** once drained
5. **Zero downtime** - users don't experience interruptions

### 5.5: Verify Deployment

💻 **Command:**

```bash
# Wait for deployment to complete
aws ecs wait services-stable \
  --cluster $CLUSTER_NAME \
  --services invsys-service

# Check deployment status
aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services invsys-service \
  --query 'services[0].deployments[0].[status,taskDefinition,desiredCount,runningCount]' \
  --output table

# Test new version endpoint (through gateway if exposed)
# Or check logs
aws logs tail /ecs/flask-inventory-invsys --since 5m
```

📋 **Expected output:**

```
---------------------------------------------------------
|                  DescribeServices                     |
+---------+---------------------+-----------+-----------+
| PRIMARY | ...invsys:2         | 2         | 2         |
+---------+---------------------+-----------+-----------+

2025-01-15T15:30:00.000Z Starting Flask app version 2.0.0
2025-01-15T15:30:01.000Z * Running on http://0.0.0.0:5000
```

### 5.6: Rollback if Needed

💻 **Command:**

```bash
# Rollback to previous revision if there's an issue
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service invsys-service \
  --task-definition flask-inventory-invsys:1 \
  --force-new-deployment

echo "Rolling back to revision 1..."
```

🎯 **Best Practice:** Always test in staging before production deployments.

✅ **Task complete!** You've performed a zero-downtime rolling deployment.

***

## Task 6: Scaling Your Services

### 6.1: Manual Scaling - Horizontal

Scale the number of running tasks.

💻 **Command:**

```bash
# Scale gateway service from 2 to 4 tasks
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service gateway-service \
  --desired-count 4

# Monitor scaling
aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services gateway-service \
  --query 'services[0].[serviceName,desiredCount,runningCount,pendingCount]' \
  --output table
```

📋 **Expected output:**

```
---------------------------------------------------------
|                  DescribeServices                     |
+------------------+-----------+-----------+------------+
|  gateway-service |  4        |  2        |  2         |
+------------------+-----------+-----------+------------+

# After a minute:
---------------------------------------------------------
|                  DescribeServices                     |
+------------------+-----------+-----------+------------+
|  gateway-service |  4        |  4        |  0         |
+------------------+-----------+-----------+------------+
```

### 6.2: Verify Load Distribution

💻 **Command:**

```bash
# Make multiple requests and check which task handles them
for i in {1..10}; do
  curl -s http://$ALB_DNS/
  sleep 1
done

# Check ALB metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=$(echo $ALB_ARN | cut -d: -f6) \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum
```

### 6.3: Scale Down

💻 **Command:**

```bash
# Scale back down to 2 tasks
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service gateway-service \
  --desired-count 2

echo "⏳ Scaling down to 2 tasks..."
```

💡 **What happens during scale-down:**
1. ECS marks 2 tasks for termination
2. ALB stops sending new requests to those tasks
3. ECS waits for existing connections to drain (default: 300 seconds)
4. ECS stops the tasks

✅ **Task complete!** You've manually scaled services.

***

## Task 7: Implementing Health Checks

### 7.1: Understanding Health Check Layers

ECS and ALB have different health check mechanisms:

| Layer | Type | Purpose | Failure Action |
|-------|------|---------|----------------|
| **Container** | Docker HEALTHCHECK | Monitor container process | Restart container |
| **ECS Task** | Task health check | Monitor task readiness | Stop/restart task |
| **ALB Target** | Target group health check | Route traffic decisions | Remove from load balancer |

### 7.2: Add Health Check Endpoint to Application

💻 **Command:**

```bash
# Update invsys with detailed health endpoint
cat >> ~/invsys/api.py << 'EOF'

@app.route('/health')
def health():
    """Detailed health check endpoint"""
    try:
        # Check if we can access storage
        devices = dal.get()

        return jsonify({
            "status": "healthy",
            "service": "inventory-system",
            "version": "2.0.0",
            "checks": {
                "database": "ok",
                "storage": "ok"
            }
        }), 200
    except Exception as e:
        return jsonify({
            "status": "unhealthy",
            "error": str(e)
        }), 503

@app.route('/health/ready')
def readiness():
    """Readiness probe - is service ready to accept traffic?"""
    return jsonify({"ready": True}), 200

@app.route('/health/live')
def liveness():
    """Liveness probe - is service alive?"""
    return jsonify({"alive": True}), 200
EOF
```

### 7.3: Update Task Definition with Health Checks

💻 **Command:**

```bash
# Create task definition with comprehensive health checks
cat > ~/invsys-task-def-health.json << EOF
{
  "family": "flask-inventory-invsys",
  "networkMode": "awsvpc",
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
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "FLASK_ENV",
          "value": "production"
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
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:5000/health/live || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
EOF

# Register and update service
aws ecs register-task-definition --cli-input-json file://~/invsys-task-def-health.json
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service invsys-service \
  --task-definition flask-inventory-invsys \
  --force-new-deployment
```

### 7.4: Update ALB Target Group Health Checks

💻 **Command:**

```bash
# Update target group health check
aws elbv2 modify-target-group \
  --target-group-arn $GATEWAY_TG_ARN \
  --health-check-path "/health/ready" \
  --health-check-interval-seconds 15 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn $GATEWAY_TG_ARN \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Reason]' \
  --output table
```

📋 **Expected output:**

```
---------------------------------------------------------
|              DescribeTargetHealth                     |
+---------------------+---------+----------------------+
|  10.0.1.50          | healthy | -                    |
|  10.0.2.75          | healthy | -                    |
+---------------------+---------+----------------------+
```

### 7.5: Test Health Check Failure Scenario

💻 **Command:**

```bash
# Simulate unhealthy task by stopping Flask process
# (In production, this would be an actual failure)

# Get task ARN
TASK_ARN=$(aws ecs list-tasks \
  --cluster $CLUSTER_NAME \
  --service-name invsys-service \
  --query 'taskArns[0]' \
  --output text)

# Stop the task to simulate failure
aws ecs stop-task \
  --cluster $CLUSTER_NAME \
  --task $TASK_ARN \
  --reason "Testing health check recovery"

# ECS will automatically start a new task
echo "⏳ Watching ECS replace failed task..."
watch -n 5 "aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services invsys-service \
  --query 'services[0].[runningCount,desiredCount]' \
  --output table"
```

📋 **Expected behavior:**

```
1. Task fails health check
2. ECS marks task as unhealthy
3. ALB removes task from load balancer rotation
4. ECS starts replacement task
5. New task passes health checks
6. ALB adds new task to rotation
7. System returns to desired state
```

✅ **Task complete!** Health checks are configured and tested.

***

## Task 8: Working with Service Auto Scaling

Configure automatic scaling based on metrics (within free tier limits).

### 8.1: Register Scalable Target

💻 **Command:**

```bash
# Register gateway service as scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/gateway-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 4 \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/ecsAutoscaleRole

# Verify registration
aws application-autoscaling describe-scalable-targets \
  --service-namespace ecs \
  --resource-ids service/${CLUSTER_NAME}/gateway-service
```

⚠️ **Free Tier Note:** Limit max-capacity to stay within EC2 free tier hours (750 hrs/month ≈ 1 instance 24/7).

### 8.2: Create Scaling Policy - Target Tracking

💻 **Command:**

```bash
# Scale based on CPU utilization
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/gateway-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name gateway-cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'

# Verify policy
aws application-autoscaling describe-scaling-policies \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/gateway-service
```

💡 **Policy Explained:**
- **Target: 70% CPU** - ECS maintains average CPU at this level
- **Scale out (add tasks)** when CPU > 70% for 60 seconds
- **Scale in (remove tasks)** when CPU < 70% for 300 seconds
- **Cooldown** prevents rapid scaling oscillations

### 8.3: Create Scaling Policy - Request Count

💻 **Command:**

```bash
# Scale based on ALB request count per target
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/gateway-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name gateway-request-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 1000.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "'"$(echo $ALB_ARN | cut -d: -f6)/$(echo $GATEWAY_TG_ARN | cut -d: -f6)"'"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

💡 **This policy:** Scales out when request count exceeds 1000 requests/minute per task.

### 8.4: Test Auto Scaling with Load

💻 **Command:**

```bash
# Generate load (simple bash loop)
echo "🔥 Generating load for 5 minutes..."
for i in {1..300}; do
  for j in {1..10}; do
    curl -s http://$ALB_DNS/items > /dev/null &
  done
  sleep 1
done

# Monitor scaling activity
watch -n 10 "aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services gateway-service \
  --query 'services[0].[desiredCount,runningCount]' \
  --output table"

# Check scaling activities
aws application-autoscaling describe-scaling-activities \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/gateway-service \
  --max-results 10
```

📋 **Expected output (after load test):**

```
Scaling activity initiated at 2025-01-15T16:00:00Z
Cause: alarm triggered due to high CPU utilization
Status: Successful
From: 2 tasks → To: 3 tasks

Scaling activity initiated at 2025-01-15T16:03:00Z
Cause: alarm triggered due to high request count
Status: Successful
From: 3 tasks → To: 4 tasks
```

### 8.5: View Scaling History

💻 **Command:**

```bash
# Get detailed scaling events
aws application-autoscaling describe-scaling-activities \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/gateway-service \
  --query 'ScalingActivities[*].[StartTime,Cause,StatusCode,Description]' \
  --output table
```

🎯 **Best Practice:** Set conservative scaling policies in free tier to avoid unexpected costs.

✅ **Task complete!** Auto scaling is configured and tested.

***

## Task 9: Monitoring with CloudWatch

### 9.1: View ECS CloudWatch Metrics

💻 **Command:**

```bash
# List available metrics
aws cloudwatch list-metrics \
  --namespace AWS/ECS \
  --dimensions Name=ServiceName,Value=gateway-service Name=ClusterName,Value=$CLUSTER_NAME

# Get CPU utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=gateway-service Name=ClusterName,Value=$CLUSTER_NAME \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum \
  --output table

# Get memory utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name MemoryUtilization \
  --dimensions Name=ServiceName,Value=gateway-service Name=ClusterName,Value=$CLUSTER_NAME \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum \
  --output table
```

### 9.2: Create CloudWatch Dashboard

💻 **Command:**

```bash
# Create dashboard JSON
cat > ~/dashboard-config.json << EOF
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/ECS", "CPUUtilization", {"stat": "Average"}],
          [".", "MemoryUtilization", {"stat": "Average"}]
        ],
        "view": "timeSeries",
        "region": "${AWS_REGION}",
        "title": "ECS Service Metrics",
        "period": 300
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/ApplicationELB", "RequestCount", {"stat": "Sum"}],
          [".", "TargetResponseTime", {"stat": "Average"}]
        ],
        "view": "timeSeries",
        "region": "${AWS_REGION}",
        "title": "ALB Metrics",
        "period": 60
      }
    }
  ]
}
EOF

# Create dashboard
aws cloudwatch put-dashboard \
  --dashboard-name FlaskInventoryMonitoring \
  --dashboard-body file://~/dashboard-config.json

echo "📊 Dashboard created: https://console.aws.amazon.com/cloudwatch/home?region=${AWS_REGION}#dashboards:name=FlaskInventoryMonitoring"
```

### 9.3: Set Up CloudWatch Alarms

💻 **Command:**

```bash
# Alarm for high CPU
aws cloudwatch put-metric-alarm \
  --alarm-name ecs-gateway-high-cpu \
  --alarm-description "Alert when gateway CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=ServiceName,Value=gateway-service Name=ClusterName,Value=$CLUSTER_NAME

# Alarm for service failures
aws cloudwatch put-metric-alarm \
  --alarm-name ecs-gateway-task-failures \
  --alarm-description "Alert when tasks are failing" \
  --metric-name RunningTaskCount \
  --namespace AWS/ECS \
  --statistic Average \
  --period 60 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=ServiceName,Value=gateway-service Name=ClusterName,Value=$CLUSTER_NAME

# List alarms
aws cloudwatch describe-alarms \
  --alarm-names ecs-gateway-high-cpu ecs-gateway-task-failures \
  --query 'MetricAlarms[*].[AlarmName,StateValue,MetricName]' \
  --output table
```

### 9.4: View Container Logs

💻 **Command:**

```bash
# Tail logs from gateway service
aws logs tail /ecs/flask-inventory-gateway --follow --since 10m

# Filter logs for errors
aws logs filter-log-events \
  --log-group-name /ecs/flask-inventory-gateway \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s)000

# Get logs from specific task
TASK_ID=$(aws ecs list-tasks --cluster $CLUSTER_NAME --service-name gateway-service --query 'taskArns[0]' --output text | rev | cut -d'/' -f1 | rev)

aws logs tail /ecs/flask-inventory-gateway --follow --filter-pattern "$TASK_ID"
```

### 9.5: Enable Container Insights (Optional - Costs Apply)

⚠️ **Note:** Container Insights has costs outside free tier (~$0.30/GB ingested).

💻 **Command:**

```bash
# Enable Container Insights for cluster
aws ecs update-cluster-settings \
  --cluster $CLUSTER_NAME \
  --settings name=containerInsights,value=enabled

# View in CloudWatch Console
echo "View Container Insights: https://console.aws.amazon.com/cloudwatch/home?region=${AWS_REGION}#container-insights:performance/ecs"
```

✅ **Task complete!** Monitoring is configured.

***

## Task 10: Understanding CloudFormation Templates (Optional)

Learn how the infrastructure was provisioned.

### 10.1: Generate CloudFormation Template from Existing Resources

💻 **Command:**

```bash
# Create CloudFormation template for ECS service
cat > ~/ecs-cloudformation-template.yaml << 'EOF'
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Flask Inventory Microservices ECS Deployment'

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC for ECS cluster

  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Subnets for ECS tasks

  InvsysImageUri:
    Type: String
    Description: ECR URI for inventory system image

  GatewayImageUri:
    Type: String
    Description: ECR URI for gateway image

Resources:
  # ECS Cluster
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: flask-inventory-cluster
      CapacityProviders:
        - FARGATE
        - FARGATE_SPOT
      DefaultCapacityProviderStrategy:
        - CapacityProvider: FARGATE
          Weight: 1
          Base: 1
      Configuration:
        ExecuteCommandConfiguration:
          Logging: DEFAULT
      ClusterSettings:
        - Name: containerInsights
          Value: disabled

  # Service Discovery Namespace
  ServiceDiscoveryNamespace:
    Type: AWS::ServiceDiscovery::PrivateDnsNamespace
    Properties:
      Name: local
      Vpc: !Ref VpcId
      Description: Service discovery for microservices

  # Inventory System Service Discovery
  InvsysServiceDiscovery:
    Type: AWS::ServiceDiscovery::Service
    Properties:
      Name: invsys
      DnsConfig:
        DnsRecords:
          - Type: A
            TTL: 60
      HealthCheckCustomConfig:
        FailureThreshold: 1
      NamespaceId: !Ref ServiceDiscoveryNamespace

  # Task Definition for Invsys
  InvsysTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: flask-inventory-invsys
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - EC2
      Cpu: '256'
      Memory: '512'
      ExecutionRoleArn: !Sub 'arn:aws:iam::${AWS::AccountId}:role/ecsTaskExecutionRole'
      ContainerDefinitions:
        - Name: invsys
          Image: !Ref InvsysImageUri
          Cpu: 256
          Memory: 512
          Essential: true
          PortMappings:
            - ContainerPort: 5000
              Protocol: tcp
          Environment:
            - Name: FLASK_ENV
              Value: production
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: /ecs/flask-inventory-invsys
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: ecs
              awslogs-create-group: 'true'
          HealthCheck:
            Command:
              - CMD-SHELL
              - curl -f http://localhost:5000/items || exit 1
            Interval: 30
            Timeout: 5
            Retries: 3
            StartPeriod: 60

  # ECS Service for Invsys
  InvsysService:
    Type: AWS::ECS::Service
    Properties:
      ServiceName: invsys-service
      Cluster: !Ref ECSCluster
      TaskDefinition: !Ref InvsysTaskDefinition
      DesiredCount: 2
      LaunchType: EC2
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets: !Ref SubnetIds
          SecurityGroups:
            - !Ref ECSSecurityGroup
          AssignPublicIp: DISABLED
      ServiceRegistries:
        - RegistryArn: !GetAtt InvsysServiceDiscovery.Arn
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 50

  # Security Group
  ECSSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for ECS tasks
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5000
          ToPort: 5001
          CidrIp: 10.0.0.0/16

Outputs:
  ClusterName:
    Description: ECS Cluster Name
    Value: !Ref ECSCluster
    Export:
      Name: !Sub '${AWS::StackName}-ClusterName'

  ServiceDiscoveryNamespace:
    Description: Service Discovery Namespace ID
    Value: !Ref ServiceDiscoveryNamespace
    Export:
      Name: !Sub '${AWS::StackName}-NamespaceId'
EOF

echo "📄 CloudFormation template created at ~/ecs-cloudformation-template.yaml"
```

### 10.2: Deploy Stack Using CloudFormation (Optional)

💻 **Command:**

```bash
# Validate template
aws cloudformation validate-template \
  --template-body file://~/ecs-cloudformation-template.yaml

# Create stack (dry run)
aws cloudformation create-stack \
  --stack-name flask-inventory-ecs \
  --template-body file://~/ecs-cloudformation-template.yaml \
  --parameters \
    ParameterKey=VpcId,ParameterValue=$VPC_ID \
    ParameterKey=SubnetIds,ParameterValue=\"$SUBNET_IDS\" \
    ParameterKey=InvsysImageUri,ParameterValue=$INVSYS_REPO_URI:latest \
    ParameterKey=GatewayImageUri,ParameterValue=$GATEWAY_REPO_URI:latest \
  --capabilities CAPABILITY_IAM \
  --dry-run

echo "💡 Template is valid and ready for deployment"
```

📚 **Learn more:** [AWS CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)

✅ **Task complete!** You understand Infrastructure as Code.

***

## Troubleshooting Common Issues

### Issue 1: Tasks Won't Start

🔧 **Symptoms:**
- Tasks stuck in PENDING state
- Events show "unable to place task"

**Diagnosis:**

💻 **Command:**

```bash
# Check cluster capacity
aws ecs describe-clusters \
  --clusters $CLUSTER_NAME \
  --query 'clusters[0].[registeredContainerInstancesCount,runningTasksCount,pendingTasksCount]'

# Check container instance resources
aws ecs describe-container-instances \
  --cluster $CLUSTER_NAME \
  --container-instances $(aws ecs list-container-instances --cluster $CLUSTER_NAME --query 'containerInstanceArns[0]' --output text) \
  --query 'containerInstances[0].remainingResources'
```

**Solutions:**

1. **Add more EC2 instances** to cluster
2. **Reduce task CPU/memory** requirements
3. **Check security group** allows ECS agent traffic

***

### Issue 2: Service Discovery Not Working

🔧 **Symptoms:**
- Gateway can't reach invsys
- "Name or service not known" errors

**Diagnosis:**

💻 **Command:**

```bash
# Check service discovery configuration
aws servicediscovery list-instances \
  --service-id $(aws servicediscovery list-services --query "Services[?Name=='invsys'].Id" --output text)

# From EC2 instance, test DNS resolution
dig invsys.local
nslookup invsys.local
```

**Solutions:**

1. **Verify awsvpc network mode** - required for service discovery
2. **Check VPC DNS settings** - enableDnsHostnames=true, enableDnsSupport=true
3. **Confirm tasks are registered** with service discovery

***

### Issue 3: Health Checks Failing

🔧 **Symptoms:**
- Targets show "unhealthy" in target group
- Tasks repeatedly stop and start

**Diagnosis:**

💻 **Command:**

```bash
# Check target health
aws elbv2 describe-target-health \
  --target-group-arn $GATEWAY_TG_ARN

# Get detailed failure reason
aws ecs describe-tasks \
  --cluster $CLUSTER_NAME \
  --tasks $(aws ecs list-tasks --cluster $CLUSTER_NAME --service-name gateway-service --query 'taskArns[0]' --output text) \
  --query 'tasks[0].containers[0].healthStatus'

# Check application logs
aws logs tail /ecs/flask-inventory-gateway --since 5m
```

**Solutions:**

1. **Increase health check grace period** (default: 0, try 60 seconds)
2. **Verify health endpoint** returns 200 status
3. **Check security groups** allow ALB → container traffic
4. **Adjust health check path** to correct endpoint

***

### Issue 4: High Memory Usage / OOM Kills

🔧 **Symptoms:**
- Tasks stopped with "OutOfMemory" reason
- Frequent task restarts

**Diagnosis:**

💻 **Command:**

```bash
# Check memory metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name MemoryUtilization \
  --dimensions Name=ServiceName,Value=gateway-service Name=ClusterName,Value=$CLUSTER_NAME \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Maximum

# Check task stopped reason
aws ecs describe-tasks \
  --cluster $CLUSTER_NAME \
  --tasks $(aws ecs list-tasks --cluster $CLUSTER_NAME --service-name gateway-service --desired-status STOPPED --query 'taskArns[0]' --output text) \
  --query 'tasks[0].stoppedReason'
```

**Solutions:**

1. **Increase memory reservation** in task definition
2. **Profile application** for memory leaks
3. **Add memory limits** to prevent OOM crashes
4. **Use Fargate** for automatic resource management

***

### Issue 5: Deployments Stuck or Rolling Back

🔧 **Symptoms:**
- New task definition won't deploy
- Service shows "deployment in progress" for long time

**Diagnosis:**

💻 **Command:**

```bash
# Check deployment status
aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services gateway-service \
  --query 'services[0].deployments'

# Check service events
aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services gateway-service \
  --query 'services[0].events[0:10]'
```

**Solutions:**

1. **Check new task health** - might be failing health checks
2. **Verify ECR image** exists and is accessible
3. **Review task execution role** permissions
4. **Check container logs** for startup errors

***

## Cleanup Instructions

### Step 1: Delete ECS Services

💻 **Command:**

```bash
# Scale down services
aws ecs update-service --cluster $CLUSTER_NAME --service gateway-service --desired-count 0
aws ecs update-service --cluster $CLUSTER_NAME --service invsys-service --desired-count 0

# Wait for tasks to stop
sleep 30

# Delete services
aws ecs delete-service --cluster $CLUSTER_NAME --service gateway-service --force
aws ecs delete-service --cluster $CLUSTER_NAME --service invsys-service --force
```

### Step 2: Delete Service Discovery

💻 **Command:**

```bash
# Delete service discovery services
aws servicediscovery delete-service \
  --id $(aws servicediscovery list-services --query "Services[?Name=='invsys'].Id" --output text)

# Delete namespace
aws servicediscovery delete-namespace \
  --id $NAMESPACE_ID
```

### Step 3: Delete Auto Scaling Configuration

💻 **Command:**

```bash
# Delete scaling policies
aws application-autoscaling delete-scaling-policy \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/gateway-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name gateway-cpu-scaling

# Deregister scalable target
aws application-autoscaling deregister-scalable-target \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/gateway-service \
  --scalable-dimension ecs:service:DesiredCount
```

### Step 4: Delete CloudWatch Resources

💻 **Command:**

```bash
# Delete alarms
aws cloudwatch delete-alarms --alarm-names ecs-gateway-high-cpu ecs-gateway-task-failures

# Delete dashboard
aws cloudwatch delete-dashboards --dashboard-names FlaskInventoryMonitoring

# Delete log groups (optional - logs will expire)
aws logs delete-log-group --log-group-name /ecs/flask-inventory-gateway
aws logs delete-log-group --log-group-name /ecs/flask-inventory-invsys
```

### Step 5: Deregister Task Definitions

💻 **Command:**

```bash
# List all revisions
aws ecs list-task-definitions --family-prefix flask-inventory

# Deregister each revision (cannot delete, only deregister)
for revision in $(seq 1 10); do
  aws ecs deregister-task-definition --task-definition flask-inventory-invsys:$revision 2>/dev/null
  aws ecs deregister-task-definition --task-definition flask-inventory-gateway:$revision 2>/dev/null
done
```

### Step 6: Delete ECS Cluster

💻 **Command:**

```bash
aws ecs delete-cluster --cluster $CLUSTER_NAME
```

### Step 7: Delete Load Balancer Resources (from previous guide)

Follow cleanup instructions from "AWS Free Tier Deployment Guide".

✅ **Cleanup complete!** All ECS resources removed.

***

## Knowledge Check Questions

Test your understanding:

### Question 1: ECS Core Concepts

**Q:** What is the difference between an ECS Task and an ECS Service?

<details>
<summary>Click to reveal answer</summary>

**A:**
- **Task:** A single running instance of a task definition. Runs once and stops. Used for batch jobs or one-time operations.
- **Service:** Manages multiple task instances to maintain a desired count. Continuously running. Automatically replaces failed tasks. Used for long-running applications like web servers.

</details>

### Question 2: Networking Modes

**Q:** Which network mode is required for AWS Cloud Map service discovery?

<details>
<summary>Click to reveal answer</summary>

**A:** `awsvpc` network mode. This mode gives each task its own elastic network interface (ENI) with a private IP address, which is required for service discovery to register and resolve tasks.

</details>

### Question 3: Health Checks

**Q:** Name the three layers of health checks in an ECS deployment with ALB.

<details>
<summary>Click to reveal answer</summary>

**A:**
1. **Container health check** (Docker HEALTHCHECK) - monitors container process
2. **ECS task health check** - monitors task readiness
3. **ALB target health check** - determines traffic routing

</details>

### Question 4: Deployment Strategy

**Q:** During a rolling deployment, why does ECS start new tasks before stopping old ones?

<details>
<summary>Click to reveal answer</summary>

**A:** To achieve zero-downtime deployments. The `maximumPercent` parameter allows ECS to temporarily run more than the desired count, ensuring traffic is always served while new tasks become healthy and old tasks drain connections.

</details>

### Question 5: Auto Scaling

**Q:** What's the purpose of the cooldown period in auto scaling policies?

<details>
<summary>Click to reveal answer</summary>

**A:** Cooldown periods prevent rapid scaling oscillations. After a scaling activity, the cooldown period ensures the system stabilizes before triggering another scaling action. This avoids the "thrashing" effect where tasks are constantly being added and removed.

</details>

***

## Additional Resources

### AWS Documentation

- 📚 [Amazon ECS Developer Guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/)
- 📚 [ECS Task Definitions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)
- 📚 [ECS Service Scheduler](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html#service_scheduler)
- 📚 [AWS Cloud Map Service Discovery](https://docs.aws.amazon.com/cloud-map/latest/dg/)
- 📚 [Application Auto Scaling](https://docs.aws.amazon.com/autoscaling/application/userguide/)
- 📚 [CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)

### Best Practices

- 🎯 [ECS Best Practices Guide](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/intro.html)
- 🎯 [Microservices on AWS](https://docs.aws.amazon.com/whitepapers/latest/microservices-on-aws/introduction.html)
- 🎯 [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- 🎯 [12-Factor App Methodology](https://12factor.net/)

### Tools & Utilities

- 🔧 [AWS Copilot CLI](https://aws.github.io/copilot-cli/) - Simplified ECS deployments
- 🔧 [ECS CLI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI.html) - Command-line tool for ECS
- 🔧 [eksctl](https://eksctl.io/) - For migrating to EKS later
- 🔧 [Terraform ECS Modules](https://registry.terraform.io/modules/terraform-aws-modules/ecs/aws/latest) - Infrastructure as Code

### Community Resources

- 💬 [AWS Forums - ECS](https://forums.aws.amazon.com/forum.jspa?forumID=187)
- 💬 [Stack Overflow - amazon-ecs tag](https://stackoverflow.com/questions/tagged/amazon-ecs)
- 💬 [AWS Reddit Community](https://www.reddit.com/r/aws/)
- 💬 [ECS Workshop](https://ecsworkshop.com/) - Hands-on tutorials


## Conclusion

🎉 **Congratulations!** You've completed the "Working with Amazon ECS" guide!

### What You've Accomplished

✅ **Mastered ECS Core Concepts**
- Task definitions, tasks, services, and clusters
- Network modes and service discovery
- Health checks at multiple layers

✅ **Deployed Production-Ready Microservices**
- Multi-container Flask application
- Service-to-service communication with Cloud Map
- Load balancing with Application Load Balancer

✅ **Implemented Operations Best Practices**
- Zero-downtime rolling deployments
- Horizontal scaling (manual and automatic)
- Comprehensive monitoring and logging
- Infrastructure as Code with CloudFormation

✅ **Gained Troubleshooting Skills**
- Diagnosed common ECS issues
- Used CloudWatch for debugging
- Resolved deployment failures

### Skills You've Developed

- **Container Orchestration:** Managing multi-container applications at scale
- **Cloud Architecture:** Designing resilient, scalable systems
- **DevOps Practices:** CI/CD pipeline concepts, automated deployments
- **Monitoring & Observability:** CloudWatch metrics, logs, and alarms
- **Cost Optimization:** Staying within AWS free tier limits


**Thank you for completing this guide!**

Your feedback helps improve this content. Please share your experience, questions, or suggestions.
