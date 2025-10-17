# Deploying Node.js Full-Stack Application to AWS EC2 with CodeDeploy


## Lab Overview

In this guide, you will learn how to deploy a full-stack Node.js/React application to Amazon EC2 instances using AWS CodeDeploy. The application consists of an Express.js backend with WebSocket support and a React frontend built with Vite. You will practice deploying to both development/testing and production environments using AWS free tier services.

### Application Architecture

The application being deployed consists of:
- **Backend**: Express.js server (Node.js 20) with REST API and Socket.IO for real-time communication
- **Frontend**: React application built with Vite
- **Database**: SQLite (embedded database)
- **Port Configuration**: Backend runs on port 8000, Frontend on port 3000

### Objectives

By completing this guide, you will be able to:

- Set up EC2 instances for web application hosting using AWS free tier
- Configure AWS CodeDeploy for automated deployments
- Create deployment packages with proper application structure
- Deploy Node.js applications to development and production environments
- Configure Amazon SNS for deployment notifications
- Implement in-place deployment strategies
- Monitor and troubleshoot deployments
- Set up environment variables and application configuration
- Configure NGINX as a reverse proxy for production deployments

### Prerequisites

- AWS Account with free tier eligibility
- Basic understanding of Node.js and React
- Familiarity with Linux command line
- Knowledge of Git version control
- Understanding of SDLC (Software Development Life Cycle)
- Basic AWS Console navigation skills

### Icon Key

- **Consider:** Pause and reflect on the concept
- **Note:** Important guidance or hint
- **Learn more:** Additional information source
- **Task complete:** Lab milestone reached
- **Command:** Shell command to execute
- **Expected output:** Verification of command/file
- **Warning:** Critical information to prevent errors
- **Caution:** Wait or verify before proceeding

***

## AWS Services Used in This Guide

### AWS CodeDeploy
Automates code deployments to EC2 instances, on-premises servers, Lambda functions, or ECS services. Handles deployment orchestration, rollback capabilities, and health monitoring.

### Amazon EC2 (Elastic Compute Cloud)
Provides scalable virtual servers in the AWS cloud. Free tier includes:
- 750 hours per month of t2.micro or t3.micro instances (Linux)
- Enough for running 1 instance 24/7 or multiple instances for testing

### Amazon S3 (Simple Storage Service)
Scalable object storage for deployment packages. Free tier includes:
- 5 GB of standard storage
- 20,000 GET requests
- 2,000 PUT requests per month

### Amazon SNS (Simple Notification Service)
Managed pub-sub messaging service for deployment notifications. Free tier includes:
- 1,000 email notifications per month

### IAM (Identity and Access Management)
Manages access to AWS services securely through roles and policies.

***

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Cloud                                │
│                                                                   │
│  ┌──────────────┐         ┌─────────────────────────────────┐  │
│  │   S3 Bucket   │────────>│      AWS CodeDeploy             │  │
│  │ (Deployments) │         │                                 │  │
│  └──────────────┘         └────────┬────────────────────────┘  │
│                                     │                            │
│                    ┌────────────────┼──────────────────┐        │
│                    │                │                  │        │
│                    ▼                ▼                  ▼        │
│          ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│          │   DevTest    │  │  Production  │  │  Production  │ │
│          │  EC2 t2.micro│  │  EC2 t2.micro│  │  EC2 t2.micro│ │
│          │              │  │  (Instance 1)│  │  (Instance 2)│ │
│          │  NGINX       │  │              │  │              │ │
│          │  Node.js     │  │  NGINX       │  │  NGINX       │ │
│          │  Backend     │  │  Node.js     │  │  Node.js     │ │
│          │  Frontend    │  │  Backend     │  │  Backend     │ │
│          │              │  │  Frontend    │  │  Frontend    │ │
│          └──────────────┘  └──────────────┘  └──────────────┘ │
│                    │                │                  │        │
│                    └────────────────┼──────────────────┘        │
│                                     │                            │
│                                     ▼                            │
│                            ┌──────────────┐                     │
│                            │   SNS Topic  │                     │
│                            │(Notifications)│                    │
│                            └──────────────┘                     │
│                                     │                            │
└─────────────────────────────────────┼────────────────────────────┘
                                      │
                                      ▼
                              Your Email Address
```

***

## Cost Considerations (AWS Free Tier)

This guide is designed to work within AWS free tier limits:

| Service | Free Tier Allowance | Estimated Usage |
|---------|---------------------|-----------------|
| EC2 t2.micro | 750 hours/month | 720 hours (1 instance 24/7) |
| S3 Storage | 5 GB | < 100 MB |
| S3 Requests | 20K GET, 2K PUT | < 100 requests |
| SNS Email | 1,000 notifications | < 50 notifications |
| CodeDeploy | Free for EC2 | Unlimited |

**Note:** Stay within free tier by:
- Using t2.micro or t3.micro instances only
- Stopping instances when not in use for learning
- Cleaning up resources after completing the guide
- Monitoring usage in AWS Billing Dashboard

***

# Tasks

## Task 0: Initial AWS Setup and Prerequisites

Before starting deployments, you need to set up the foundational AWS infrastructure.

### Task 0.1: Create an IAM Role for EC2 Instances

EC2 instances need permission to communicate with CodeDeploy and S3.

1. Open the **AWS Console** and navigate to **IAM**.
2. In the left navigation, choose **Roles**.
3. Click **Create role**.
4. Configure the role:
   - **Trusted entity type**: AWS service
   - **Use case**: EC2
   - Click **Next**.
5. Attach permissions policies:
   - Search for and select: `AmazonEC2RoleforAWSCodeDeploy`
   - Search for and select: `AmazonS3ReadOnlyAccess`
   - Click **Next**.
6. Name the role: `EC2CodeDeployRole`
7. Click **Create role**.

### Task 0.2: Create an IAM Role for CodeDeploy

CodeDeploy needs permission to manage deployments to EC2 instances.

1. In **IAM** → **Roles**, click **Create role**.
2. Configure the role:
   - **Trusted entity type**: AWS service
   - **Use case**: CodeDeploy
   - Select **CodeDeploy** (not CodeDeploy ECS)
   - Click **Next**.
3. The policy `AWSCodeDeployRole` is automatically attached.
4. Name the role: `CodeDeployServiceRole`
5. Click **Create role**.

### Task 0.3: Create an S3 Bucket for Deployment Packages

1. Navigate to **S3** in the AWS Console.
2. Click **Create bucket**.
3. Configure bucket:
   - **Bucket name**: `codedeploy-app-deployments-<your-unique-id>` (e.g., `codedeploy-app-deployments-12345`)
   - **Region**: Choose your preferred region (e.g., us-east-1)
   - **Block Public Access**: Keep all options checked (recommended)
   - Leave other settings as default
4. Click **Create bucket**.
5. **Note:** Save your bucket name for later use.

**Task complete:** AWS infrastructure foundation is ready.

***

## Task 1: Launch and Configure EC2 Instances

### Task 1.1: Launch DevTest EC2 Instance

1. Navigate to **EC2** in the AWS Console.
2. Click **Launch Instance**.
3. Configure the instance:

   **Name and tags:**
   - Name: `DevTest-Instance`

   **Application and OS Images (Amazon Machine Image):**
   - **Quick Start**: Amazon Linux
   - **Amazon Machine Image (AMI)**: Amazon Linux 2023 AMI (Free tier eligible)

   **Instance type:**
   - Type: `t2.micro` (Free tier eligible)

   **Key pair (login):**
   - Click **Create new key pair**
   - Key pair name: `nodejs-app-key`
   - Key pair type: RSA
   - Private key file format: `.pem` (for Mac/Linux) or `.ppk` (for Windows/PuTTY)
   - Click **Create key pair** (the file will download automatically)
   - **Important:** Save this file securely; you cannot download it again

   **Network settings:**
   - Click **Edit**
   - Auto-assign public IP: **Enable**
   - Firewall (security groups): **Create security group**
   - Security group name: `nodejs-app-sg-devtest`
   - Description: `Security group for Node.js application DevTest`
   - Add the following inbound rules:

   | Type | Protocol | Port Range | Source | Description |
   |------|----------|------------|--------|-------------|
   | SSH | TCP | 22 | My IP | SSH access |
   | HTTP | TCP | 80 | 0.0.0.0/0 | HTTP access |
   | Custom TCP | TCP | 3000 | 0.0.0.0/0 | Frontend application |
   | Custom TCP | TCP | 8000 | 0.0.0.0/0 | Backend API (temporary) |

   **Configure storage:**
   - Size: 8 GiB (default)
   - Volume type: gp3 (default)

   **Advanced details:**
   - IAM instance profile: Select `EC2CodeDeployRole`
   - User data: Paste the following script:

```bash
#!/bin/bash
# Update system packages
dnf update -y

# Install Node.js 20
dnf install -y nodejs npm

# Install CodeDeploy agent
dnf install -y ruby wget
cd /home/ec2-user
wget https://aws-codedeploy-us-east-1.s3.us-east-1.amazonaws.com/latest/install
chmod +x ./install
./install auto

# Install and configure NGINX
dnf install -y nginx
systemctl enable nginx

# Create application directory
mkdir -p /var/www/nodejs-app
chown -R ec2-user:ec2-user /var/www/nodejs-app

# Create NGINX configuration for Node.js app
cat > /etc/nginx/conf.d/nodejs-app.conf << 'EOF'
server {
    listen 80;
    server_name _;

    # Frontend - serve static files
    location / {
        root /var/www/nodejs-app/frontend/dist;
        try_files $uri $uri/ /index.html;
    }

    # Backend API
    location /api {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # WebSocket support
    location /socket.io {
        proxy_pass http://localhost:8000/socket.io;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
EOF

# Restart NGINX
systemctl restart nginx

# Start CodeDeploy agent
systemctl start codedeploy-agent
systemctl enable codedeploy-agent
```

4. Review and click **Launch instance**.
5. Wait for the instance state to show **Running** (this may take 2-3 minutes).
6. Select your instance and note the **Public IPv4 address** for later use.

### Task 1.2: Launch Production EC2 Instance (Optional)

For production environment, repeat Task 1.1 with these changes:
- **Name**: `Production-Instance`
- **Security group name**: `nodejs-app-sg-prod`
- Use the same key pair: `nodejs-app-key`
- Use the same IAM role: `EC2CodeDeployRole`
- Use the same user data script

**Note:** For this guide, we'll focus on DevTest deployment first. Production setup follows the same pattern.

### Task 1.3: Verify EC2 Instance Setup

1. Wait 5 minutes after instance launch for the user data script to complete.

2. Connect to your instance using SSH:

```bash
chmod 400 nodejs-app-key.pem
ssh -i nodejs-app-key.pem ec2-user@<your-instance-public-ip>
```

3. Verify installations:

```bash
# Check Node.js version
node --version
# Expected output: v20.x.x

# Check npm version
npm --version
# Expected output: 10.x.x

# Check CodeDeploy agent status
sudo systemctl status codedeploy-agent
# Expected output: active (running)

# Check NGINX status
sudo systemctl status nginx
# Expected output: active (running)
```

4. If any service is not running:

```bash
# Start CodeDeploy agent if needed
sudo systemctl start codedeploy-agent

# Start NGINX if needed
sudo systemctl start nginx
```

5. Exit the SSH session:
```bash
exit
```

**Task complete:** EC2 instances are configured and ready for deployments.

***

## Task 2: Subscribe to SNS Topic for Notifications

AWS SNS will send you email notifications about deployment status.

### Task 2.1: Create SNS Topic

1. In the AWS Console, search for and open **SNS** (Simple Notification Service).
2. In the left navigation, choose **Topics**.
3. Click **Create topic**.
4. Configure the topic:
   - **Type**: Standard
   - **Name**: `CodeDeployNotifications`
   - **Display name**: `CodeDeploy`
   - Leave other settings as default
5. Click **Create topic**.
6. Copy the **ARN** (Amazon Resource Name) of the topic for later use.

### Task 2.2: Create Email Subscription

1. On the topic details page, click **Create subscription**.
2. Configure the subscription:
   - **Protocol**: Email
   - **Endpoint**: Enter your valid email address
3. Click **Create subscription**.
4. Check your email inbox for a confirmation message from AWS Notifications.
   - **Note:** The email may take up to 5 minutes to arrive.
   - Check your spam/junk folder if you don't see it in your inbox.
5. Click the **Confirm subscription** link in the email.
6. You should see a confirmation page in your browser.
7. Close the confirmation page and return to the AWS Console.
8. Refresh the subscriptions list to verify the status is **Confirmed**.

**Task complete:** You are now subscribed to receive deployment notifications via email.

***

## Task 3: Prepare Your Application for Deployment

### Task 3.1: Understanding the Application Structure

Your application has the following structure:

```
deploy_to_builder_lab/
├── backend/
│   ├── src/
│   │   ├── index.js          # Express server entry point
│   │   ├── socket.js         # WebSocket configuration
│   │   ├── routes/           # API routes
│   │   ├── middleware/       # Express middleware
│   │   └── data/             # Database models and data
│   ├── package.json          # Backend dependencies
│   └── Dockerfile            # Backend container config
├── frontend/
│   ├── src/
│   │   ├── main.jsx          # React entry point
│   │   ├── App.jsx           # Main React component
│   │   └── components/       # React components
│   ├── index.html            # HTML template
│   ├── package.json          # Frontend dependencies
│   ├── vite.config.js        # Vite configuration
│   └── Dockerfile            # Frontend container config
├── docker-compose.yaml       # Docker Compose configuration
└── .env                      # Environment variables
```

### Task 3.2: Create AppSpec File for CodeDeploy

The AppSpec file tells CodeDeploy how to deploy your application. Create this file in your project root.

1. Navigate to your project directory:
```bash
cd /path/to/deploy_to_builder_lab
```

2. Create the `appspec.yml` file:

```bash
cat > appspec.yml << 'EOF'
version: 0.0
os: linux
files:
  - source: /backend
    destination: /var/www/nodejs-app/backend
  - source: /frontend
    destination: /var/www/nodejs-app/frontend
  - source: /scripts
    destination: /var/www/nodejs-app/scripts
  - source: /.env
    destination: /var/www/nodejs-app/.env

permissions:
  - object: /var/www/nodejs-app
    owner: ec2-user
    group: ec2-user
    mode: 755
    type:
      - directory
  - object: /var/www/nodejs-app
    owner: ec2-user
    group: ec2-user
    mode: 644
    type:
      - file

hooks:
  ApplicationStop:
    - location: scripts/stop_application.sh
      timeout: 300
      runas: ec2-user

  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 300
      runas: root

  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 600
      runas: ec2-user

  ApplicationStart:
    - location: scripts/start_application.sh
      timeout: 300
      runas: ec2-user

  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 300
      runas: ec2-user
EOF
```

**Understanding the AppSpec file:**

- **version**: AppSpec version (always 0.0 for EC2/on-premises)
- **os**: Target operating system (linux or windows)
- **files**: Source and destination mapping for application files
- **permissions**: File/directory permissions after deployment
- **hooks**: Lifecycle event hooks that run scripts at specific deployment stages

**Lifecycle Event Hooks:**
1. **ApplicationStop**: Stops the currently running application
2. **BeforeInstall**: Runs before files are copied (e.g., create directories)
3. **AfterInstall**: Runs after files are copied (e.g., install dependencies)
4. **ApplicationStart**: Starts the application
5. **ValidateService**: Validates the deployment was successful

### Task 3.3: Create Deployment Scripts

Create the lifecycle hook scripts that CodeDeploy will execute.

1. Create the scripts directory:
```bash
mkdir -p scripts
```

2. Create `scripts/stop_application.sh`:

```bash
cat > scripts/stop_application.sh << 'EOF'
#!/bin/bash
set -e

echo "Stopping Node.js application..."

# Stop backend if running
if pm2 list | grep -q "backend"; then
    pm2 stop backend || true
    pm2 delete backend || true
fi

# Stop frontend if running
if pm2 list | grep -q "frontend"; then
    pm2 stop frontend || true
    pm2 delete frontend || true
fi

echo "Application stopped successfully"
EOF
chmod +x scripts/stop_application.sh
```

3. Create `scripts/before_install.sh`:

```bash
cat > scripts/before_install.sh << 'EOF'
#!/bin/bash
set -e

echo "Running before install tasks..."

# Install PM2 globally if not already installed
if ! command -v pm2 &> /dev/null; then
    echo "Installing PM2..."
    npm install -g pm2
fi

# Create application directory if it doesn't exist
mkdir -p /var/www/nodejs-app

# Clean up old deployment files
if [ -d "/var/www/nodejs-app/backend" ]; then
    echo "Cleaning up old backend files..."
    rm -rf /var/www/nodejs-app/backend
fi

if [ -d "/var/www/nodejs-app/frontend/dist" ]; then
    echo "Cleaning up old frontend build files..."
    rm -rf /var/www/nodejs-app/frontend/dist
fi

echo "Before install tasks completed"
EOF
chmod +x scripts/before_install.sh
```

4. Create `scripts/after_install.sh`:

```bash
cat > scripts/after_install.sh << 'EOF'
#!/bin/bash
set -e

echo "Running after install tasks..."

cd /var/www/nodejs-app

# Set environment variables
export NODE_ENV=production
export STAGE_VALUE=${DEPLOYMENT_GROUP_NAME:-"DevTest"}
export APP_VERSION=${DEPLOYMENT_ID:-"1.0.0"}

# Install backend dependencies
echo "Installing backend dependencies..."
cd /var/www/nodejs-app/backend
npm ci --production

# Install frontend dependencies and build
echo "Installing frontend dependencies..."
cd /var/www/nodejs-app/frontend
npm ci

# Update frontend configuration to point to backend
if [ -f "src/config.js" ]; then
    sed -i "s|http://localhost:8000|/api|g" src/config.js
fi

# Build frontend
echo "Building frontend..."
npm run build

# Ensure proper permissions
chown -R ec2-user:ec2-user /var/www/nodejs-app

echo "After install tasks completed"
EOF
chmod +x scripts/after_install.sh
```

5. Create `scripts/start_application.sh`:

```bash
cat > scripts/start_application.sh << 'EOF'
#!/bin/bash
set -e

echo "Starting Node.js application..."

cd /var/www/nodejs-app

# Load environment variables
if [ -f ".env" ]; then
    export $(cat .env | grep -v '^#' | xargs)
fi

# Set deployment-specific environment variables
export NODE_ENV=production
export STAGE_VALUE=${DEPLOYMENT_GROUP_NAME:-"DevTest"}
export APP_VERSION=${DEPLOYMENT_ID:-"1.0.0"}

# Start backend with PM2
echo "Starting backend server..."
cd /var/www/nodejs-app/backend
pm2 start src/index.js --name backend --env production

# Save PM2 process list
pm2 save

# Configure PM2 to start on system boot
pm2 startup systemd -u ec2-user --hp /home/ec2-user

# Restart NGINX to ensure configuration is loaded
sudo systemctl restart nginx

echo "Application started successfully"
EOF
chmod +x scripts/start_application.sh
```

6. Create `scripts/validate_service.sh`:

```bash
cat > scripts/validate_service.sh << 'EOF'
#!/bin/bash
set -e

echo "Validating deployment..."

# Wait for application to start
sleep 10

# Check if backend is running
if ! pm2 list | grep -q "backend.*online"; then
    echo "ERROR: Backend is not running"
    exit 1
fi

# Check if backend responds to health check
BACKEND_HEALTH=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/health || echo "000")
if [ "$BACKEND_HEALTH" != "200" ]; then
    echo "ERROR: Backend health check failed (HTTP $BACKEND_HEALTH)"
    exit 1
fi

# Check if NGINX is running
if ! sudo systemctl is-active --quiet nginx; then
    echo "ERROR: NGINX is not running"
    exit 1
fi

# Check if frontend is accessible through NGINX
FRONTEND_CHECK=$(curl -s -o /dev/null -w "%{http_code}" http://localhost/ || echo "000")
if [ "$FRONTEND_CHECK" != "200" ] && [ "$FRONTEND_CHECK" != "304" ]; then
    echo "ERROR: Frontend is not accessible (HTTP $FRONTEND_CHECK)"
    exit 1
fi

echo "Deployment validation successful!"
echo "Backend is running and healthy"
echo "Frontend is accessible"
echo "NGINX is properly configured"
EOF
chmod +x scripts/validate_service.sh
```

### Task 3.4: Update Frontend Configuration

Your frontend needs to know where to find the backend API. Update the frontend configuration to use relative paths.

1. Check if frontend has a config file:
```bash
ls frontend/src/config.js 2>/dev/null || echo "No config.js found"
```

2. If the file exists, update it. Otherwise, create it:

```bash
cat > frontend/src/config.js << 'EOF'
// API Configuration
const config = {
  // Use relative path for production (proxied through NGINX)
  // Use localhost for development
  apiUrl: import.meta.env.PROD ? '/api' : 'http://localhost:8000',
  socketUrl: import.meta.env.PROD ? '' : 'http://localhost:8000'
};

export default config;
EOF
```

3. If your frontend uses direct API calls, update them to use the config:

```javascript
import config from './config';

// Instead of: fetch('http://localhost:8000/api/endpoint')
// Use: fetch(`${config.apiUrl}/endpoint`)
```

### Task 3.5: Add Health Check Endpoint to Backend

Ensure your backend has a health check endpoint for deployment validation.

1. Check if `backend/src/index.js` has a health endpoint:

```bash
grep -n "/health" backend/src/index.js || echo "No health endpoint found"
```

2. If not found, add it. Open `backend/src/index.js` and add:

```javascript
// Health check endpoint for CodeDeploy
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    environment: process.env.NODE_ENV || 'development'
  });
});
```

**Task complete:** Your application is now prepared for CodeDeploy with all necessary scripts and configurations.

***

## Task 4: Create and Upload Deployment Package

### Task 4.1: Create Deployment Package (Version 1.0)

1. Ensure you're in the project directory:
```bash
cd /path/to/deploy_to_builder_lab
```

2. Verify all required files are present:
```bash
ls -la
# Should see: appspec.yml, backend/, frontend/, scripts/, .env
```

3. Create the deployment package:
```bash
# Create version 1.0 deployment package
zip -r ../nodejs-app-v1.0.zip . \
  -x "*.git*" \
  -x "*node_modules*" \
  -x "*dist*" \
  -x "*.log" \
  -x "*__tests__*" \
  -x "*.md" \
  -x "*docker-compose.yaml" \
  -x "*Dockerfile"

cd ..
ls -lh nodejs-app-v1.0.zip
```

**Note:** The `-x` flags exclude unnecessary files from the deployment package (Git files, node_modules, build artifacts, tests, documentation, and Docker files).

**Expected output:**
```
-rw-r--r--  1 user  staff   XXK Nov 17 10:30 nodejs-app-v1.0.zip
```

### Task 4.2: Upload Package to S3

1. Set your S3 bucket name (replace with your actual bucket name):
```bash
export CODE_BUCKET="codedeploy-app-deployments-<your-unique-id>"
echo $CODE_BUCKET
```

2. Verify the bucket exists:
```bash
aws s3 ls | grep $CODE_BUCKET
```

3. Upload the deployment package:
```bash
aws s3 cp nodejs-app-v1.0.zip s3://$CODE_BUCKET/nodejs-app-v1.0.zip
```

**Expected output:**
```
upload: ./nodejs-app-v1.0.zip to s3://codedeploy-app-deployments-xxxxx/nodejs-app-v1.0.zip
```

4. Verify the upload:
```bash
aws s3 ls s3://$CODE_BUCKET/
```

**Expected output:**
```
2025-11-17 10:32:15     XXXXX nodejs-app-v1.0.zip
```

5. Get the S3 URI for later use:
```bash
echo "s3://$CODE_BUCKET/nodejs-app-v1.0.zip"
```

**Task complete:** Deployment package is created and uploaded to S3.

***

## Task 5: Configure AWS CodeDeploy

### Task 5.1: Create CodeDeploy Application

1. In the AWS Console, navigate to **CodeDeploy**.
2. In the left navigation, choose **Applications**.
3. Click **Create application**.
4. Configure the application:
   - **Application name**: `nodejs-fullstack-app`
   - **Compute platform**: EC2/On-premises
5. Click **Create application**.

### Task 5.2: Create Deployment Group for DevTest Environment

1. On the application page, click **Create deployment group**.
2. Configure the deployment group:

   **Deployment group name:**
   - Enter: `DevTest`

   **Service role:**
   - Select: `CodeDeployServiceRole` (created in Task 0.2)

   **Deployment type:**
   - Select: **In-place**

   **Environment configuration:**
   - Select: **Amazon EC2 instances**
   - Tag group 1:
     - Key: `Name`
     - Value: `DevTest-Instance`

   **Agent configuration with AWS Systems Manager:**
   - Leave as default (not required since we installed the agent manually)

   **Deployment settings:**
   - Deployment configuration: `CodeDeployDefault.AllAtOnce`

   **Load balancer:**
   - Uncheck **Enable load balancing** (not needed for DevTest)

3. Expand **Advanced - optional** section:

   **Alarms:**
   - Leave empty for DevTest

   **Rollback:**
   - Check **Disable rollbacks** (for learning purposes)
   - **Note:** In production, you should enable automatic rollbacks

   **Deployment triggers:**
   - Click **Create trigger**
   - Trigger name: `DeploymentFailure`
   - Events: Check **Deployment fails**
   - Amazon SNS topic: Select `CodeDeployNotifications`
   - Click **Add trigger** (Note: button may appear as checkmark)

4. Click **Create deployment group**.

**Task complete:** DevTest deployment group is configured.

***

## Task 6: Deploy Version 1.0 to DevTest Environment

### Task 6.1: Create Initial Deployment

1. On the `nodejs-fullstack-app` application page, ensure **DevTest** deployment group is selected.
2. Click **Create deployment**.
3. Configure the deployment:

   **Deployment group:**
   - Select: `DevTest`

   **Revision type:**
   - Select: **My application is stored in Amazon S3**

   **Revision location:**
   - Enter your S3 URI: `s3://codedeploy-app-deployments-<your-unique-id>/nodejs-app-v1.0.zip`
   - Tip: Use the URI from Task 4.2, step 5

   **Revision file type:**
   - Select: `.zip`

   **Deployment description:**
   - Enter: `Initial deployment - Version 1.0`

   **Additional deployment behavior settings:**
   - Leave as defaults

4. Click **Create deployment**.
5. You will be redirected to the deployment details page.

### Task 6.2: Monitor Deployment Progress

1. On the deployment details page, observe the **Status** section:
   - **In progress**: Deployment is running
   - Each lifecycle event will be listed with its status

2. Watch the **Deployment lifecycle events** section:
   - ApplicationStop
   - BeforeInstall
   - AfterInstall
   - ApplicationStart
   - ValidateService

3. Each event will show:
   - **Pending**: Waiting to run
   - **In Progress**: Currently executing
   - **Succeeded**: Completed successfully
   - **Failed**: Encountered an error

4. The deployment should complete in 5-10 minutes.

### Task 6.3: Troubleshoot Deployment Failure (Expected on First Try)

**Note:** The first deployment might fail. This is intentional to practice troubleshooting.

If the deployment shows **Failed** status:

1. Click on the **View events** link for the failed instance.
2. Identify which lifecycle event failed (look for **Failed** status).
3. Click **View logs** for the failed event.
4. Common issues and solutions:

**Issue 1: Script Not Found**
```
Error: The script does not exist at the specified location
```
**Solution:** Verify the script path in `appspec.yml` matches the actual location.

**Issue 2: Permission Denied**
```
Error: Permission denied
```
**Solution:** Ensure scripts have execute permissions (`chmod +x scripts/*.sh`).

**Issue 3: npm Command Not Found**
```
Error: npm: command not found
```
**Solution:** Verify Node.js and npm are installed on EC2 instance (revisit Task 1.3).

**Issue 4: PM2 Command Not Found**
```
Error: pm2: command not found
```
**Solution:** The `before_install.sh` script should install PM2. Check if it ran successfully.

5. To fix issues and retry:

```bash
# Fix the issue in your local files
# For example, if script permissions are missing:
chmod +x scripts/*.sh

# Create a new deployment package
cd /path/to/deploy_to_builder_lab
zip -r ../nodejs-app-v1.0.zip . \
  -x "*.git*" \
  -x "*node_modules*" \
  -x "*dist*" \
  -x "*.log" \
  -x "*__tests__*" \
  -x "*.md" \
  -x "*docker-compose.yaml" \
  -x "*Dockerfile"

# Upload to S3 (overwrite existing)
cd ..
aws s3 cp nodejs-app-v1.0.zip s3://$CODE_BUCKET/nodejs-app-v1.0.zip
```

6. In the AWS Console, on the failed deployment page, click **Retry deployment**.

### Task 6.4: Verify Successful Deployment

Once the deployment shows **Succeeded**:

1. Get your instance public IP:
   - Go to **EC2** → **Instances**
   - Select **DevTest-Instance**
   - Copy the **Public IPv4 address**

2. Open your web browser and navigate to:
```
http://<your-instance-public-ip>
```

3. Verify the application is running:
   - You should see the frontend React application
   - Test functionality (login, navigation, features)

4. Check backend API:
```
http://<your-instance-public-ip>/api/health
```
**Expected output:**
```json
{
  "status": "healthy",
  "timestamp": "2025-11-17T10:45:23.456Z",
  "environment": "production"
}
```

5. SSH into the instance to verify backend:
```bash
ssh -i nodejs-app-key.pem ec2-user@<your-instance-public-ip>

# Check PM2 status
pm2 status

# Expected output:
# ┌─────┬────────┬─────────┬──────┬─────┬──────────┐
# │ id  │ name   │ status  │ cpu  │ mem │ watching │
# ├─────┼────────┼─────────┼──────┼─────┼──────────┤
# │ 0   │ backend│ online  │ 0%   │ 50M │ disabled │
# └─────┴────────┴─────────┴──────┴─────┴──────────┘

# Check application logs
pm2 logs backend --lines 20

# Check NGINX status
sudo systemctl status nginx

# Exit SSH session
exit
```

**Task complete:** Version 1.0 is successfully deployed to DevTest environment.

***

## Task 7: Deploy to Production Environment (Optional)

For production deployments, the process is similar but with additional considerations.

### Task 7.1: Create Production Deployment Group

1. In **CodeDeploy** → **Applications** → **nodejs-fullstack-app**.
2. Click **Create deployment group**.
3. Configure:
   - **Deployment group name**: `Production`
   - **Service role**: `CodeDeployServiceRole`
   - **Deployment type**: In-place
   - **Environment configuration**: Amazon EC2 instances
     - Tag: Name / Production-Instance
   - **Deployment settings**: `CodeDeployDefault.OneAtATime` (safer for production)
   - **Load balancer**: Disabled (or configure if using ELB)
   - **Deployment triggers**:
     - Trigger for deployment failures
     - Trigger for deployment successes
4. Click **Create deployment group**.

### Task 7.2: Deploy to Production

1. Click **Create deployment**.
2. Configure:
   - **Deployment group**: Production
   - **Revision location**: `s3://codedeploy-app-deployments-<your-id>/nodejs-app-v1.0.zip`
   - **Description**: `Production deployment - Version 1.0`
3. Click **Create deployment**.
4. Monitor and verify as in Task 6.

**Task complete:** Application deployed to production environment.

***

## Task 8: Deploy Updated Version (Version 2.0)

Let's practice deploying an update to demonstrate the complete CI/CD workflow.

### Task 8.1: Make Application Changes

1. Navigate to your project directory:
```bash
cd /path/to/deploy_to_builder_lab
```

2. Make a visible change to the frontend (example):

```bash
# Update the main page title or add a version indicator
cat >> frontend/src/App.jsx << 'EOF'

// Add version display in your component
<div style={{ position: 'fixed', bottom: '10px', right: '10px',
              fontSize: '12px', color: '#666' }}>
  Version 2.0
</div>
EOF
```

3. Update the backend health endpoint to reflect the new version:

```bash
# Edit backend/src/index.js
# Update the health endpoint to return version 2.0
```

**Note:** Make meaningful changes that you can verify after deployment.

### Task 8.2: Create and Upload Version 2.0 Package

1. Create the new deployment package:
```bash
cd /path/to/deploy_to_builder_lab
zip -r ../nodejs-app-v2.0.zip . \
  -x "*.git*" \
  -x "*node_modules*" \
  -x "*dist*" \
  -x "*.log" \
  -x "*__tests__*" \
  -x "*.md" \
  -x "*docker-compose.yaml" \
  -x "*Dockerfile"

cd ..
```

2. Upload to S3:
```bash
aws s3 cp nodejs-app-v2.0.zip s3://$CODE_BUCKET/nodejs-app-v2.0.zip
```

### Task 8.3: Deploy Version 2.0 to DevTest

1. In **CodeDeploy**, go to **nodejs-fullstack-app** → **DevTest**.
2. Click **Create deployment**.
3. Configure:
   - **Revision location**: `s3://codedeploy-app-deployments-<your-id>/nodejs-app-v2.0.zip`
   - **Description**: `Update deployment - Version 2.0`
   - **File overwrite behavior**: Check **Overwrite the content**
4. Click **Create deployment**.
5. Monitor the deployment progress.

### Task 8.4: Verify the Update

1. After successful deployment, refresh your browser at:
```
http://<your-devtest-instance-ip>
```

2. Verify:
   - New frontend changes are visible
   - Backend health endpoint shows version 2.0
   - All functionality still works correctly

3. Check the deployment:
```bash
ssh -i nodejs-app-key.pem ec2-user@<your-instance-ip>
pm2 logs backend --lines 50
exit
```

**Task complete:** Successfully deployed an application update.

***

## Task 9: Implement Blue/Green Deployment (Advanced - Optional)

Blue/green deployments provide zero-downtime updates by switching traffic between two identical environments.

**Note:** This requires setting up an Application Load Balancer and Auto Scaling Group, which may exceed free tier limits. This section is for reference and learning.

### Architecture for Blue/Green:

```
Application Load Balancer
        |
        ├── Target Group Blue (current production)
        │   └── EC2 Instance(s)
        │
        └── Target Group Green (new version)
            └── EC2 Instance(s)
```

### High-Level Steps:

1. Create an Application Load Balancer
2. Create two Target Groups (Blue and Green)
3. Create an Auto Scaling Group
4. Update deployment group to use blue/green deployment
5. Configure traffic rerouting

For detailed instructions, refer to the AWS CodeDeploy Blue/Green documentation.

***

## Task 10: Monitoring and Maintenance

### Task 10.1: View Deployment History

1. In **CodeDeploy** → **Applications** → **nodejs-fullstack-app**.
2. Select a deployment group.
3. View **Deployment history** tab to see all past deployments.
4. Click on any deployment to see detailed logs and events.

### Task 10.2: Set Up CloudWatch Logs (Optional)

To collect application logs in CloudWatch:

1. Install CloudWatch agent on EC2:
```bash
ssh -i nodejs-app-key.pem ec2-user@<your-instance-ip>

sudo dnf install -y amazon-cloudwatch-agent

# Configure PM2 to output JSON logs
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
```

2. Configure CloudWatch agent to collect PM2 logs.

### Task 10.3: Monitor Application Health

Create a simple monitoring script:

```bash
ssh -i nodejs-app-key.pem ec2-user@<your-instance-ip>

cat > /home/ec2-user/monitor.sh << 'EOF'
#!/bin/bash
# Simple health check script

echo "=== Application Health Check ==="
echo "Time: $(date)"
echo ""

# Check PM2 processes
echo "PM2 Status:"
pm2 status

echo ""
echo "Backend Health:"
curl -s http://localhost:8000/health | jq .

echo ""
echo "Disk Usage:"
df -h /

echo ""
echo "Memory Usage:"
free -h
EOF

chmod +x /home/ec2-user/monitor.sh
```

Run periodically:
```bash
./monitor.sh
```

**Task complete:** Monitoring and maintenance practices established.

***

## Task 11: Cleanup and Cost Management

To avoid unexpected charges, clean up resources when done learning.

### Task 11.1: Stop EC2 Instances (Preserve for Later Use)

If you want to keep the setup but stop incurring charges:

1. Go to **EC2** → **Instances**.
2. Select your instances.
3. Click **Instance state** → **Stop instance**.

**Note:** You will still be charged for EBS volumes, but not for instance runtime.

### Task 11.2: Complete Cleanup (Remove All Resources)

To completely remove all resources:

1. **Delete CodeDeploy application:**
   - **CodeDeploy** → **Applications**
   - Select `nodejs-fullstack-app`
   - **Actions** → **Delete application**

2. **Terminate EC2 instances:**
   - **EC2** → **Instances**
   - Select instances → **Instance state** → **Terminate instance**

3. **Delete S3 bucket:**
   ```bash
   # Delete all objects first
   aws s3 rm s3://$CODE_BUCKET --recursive

   # Delete bucket
   aws s3 rb s3://$CODE_BUCKET
   ```

4. **Delete SNS topic and subscription:**
   - **SNS** → **Topics**
   - Select `CodeDeployNotifications` → **Delete**

5. **Delete IAM roles:**
   - **IAM** → **Roles**
   - Delete `EC2CodeDeployRole` and `CodeDeployServiceRole`
   - **Caution:** Only delete if not used by other resources

6. **Delete Security Groups:**
   - **EC2** → **Security Groups**
   - Delete `nodejs-app-sg-devtest` and `nodejs-app-sg-prod`

7. **Delete Key Pair:**
   - **EC2** → **Key Pairs**
   - Delete `nodejs-app-key`
   - Also delete the local `.pem` file

**Task complete:** All AWS resources cleaned up.

***

## Conclusion

Congratulations! You have successfully:

- Set up EC2 instances for hosting Node.js applications
- Configured AWS CodeDeploy for automated deployments
- Created deployment packages with proper AppSpec configuration
- Deployed a full-stack JavaScript application to AWS
- Implemented lifecycle hooks for deployment automation
- Monitored and troubleshooted deployments
- Deployed application updates
- Learned cleanup procedures to manage costs

### Key Takeaways

1. **AppSpec File**: The heart of CodeDeploy configuration, defining file mappings and lifecycle hooks
2. **Lifecycle Hooks**: Enable custom scripts at each deployment stage for maximum control
3. **In-Place Deployments**: Simple and cost-effective for development and small-scale production
4. **Monitoring**: Essential for catching issues early and maintaining application health
5. **Free Tier**: AWS provides generous free tier options for learning and small projects

### Skills Acquired

- EC2 instance configuration and management
- CodeDeploy application and deployment group setup
- Shell scripting for deployment automation
- NGINX configuration as reverse proxy
- PM2 process management
- AWS CLI usage for S3 and deployments
- Troubleshooting deployment failures
- Infrastructure as Code practices

***

## Additional Resources

### AWS CodeDeploy Documentation

- [CodeDeploy User Guide](https://docs.aws.amazon.com/codedeploy/latest/userguide/)
- [AppSpec File Reference](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file.html)
- [AppSpec Hooks Section](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html)
- [Troubleshooting CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/troubleshooting.html)

### AWS Services Documentation

- [EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [S3 User Guide](https://docs.aws.amazon.com/s3/)
- [SNS Developer Guide](https://docs.aws.amazon.com/sns/)
- [IAM User Guide](https://docs.aws.amazon.com/iam/)

### Node.js and Deployment Best Practices

- [Node.js Production Best Practices](https://nodejs.org/en/docs/guides/nodejs-docker-webapp/)
- [PM2 Process Manager](https://pm2.keymetrics.io/docs/usage/quick-start/)
- [NGINX Configuration Guide](https://nginx.org/en/docs/)
- [Express.js Production Best Practices](https://expressjs.com/en/advanced/best-practice-performance.html)

### AWS Free Tier Information

- [AWS Free Tier Details](https://aws.amazon.com/free/)
- [AWS Pricing Calculator](https://calculator.aws/)
- [AWS Cost Management](https://aws.amazon.com/aws-cost-management/)

### Tutorials and Learning Resources

- [AWS CodeDeploy Tutorial](https://aws.amazon.com/codedeploy/getting-started/)
- [Deploy Node.js App to EC2](https://aws.amazon.com/getting-started/hands-on/deploy-nodejs-web-app/)
- [Using CodeDeploy Environment Variables](https://aws.amazon.com/blogs/devops/using-codedeploy-environment-variables/)

***

## Appendix A: Common Issues and Solutions

### Issue: CodeDeploy Agent Not Installed or Not Running

**Symptoms:**
- Deployment fails immediately
- Error: "The deployment failed because no instances were found for your deployment group"

**Solutions:**
```bash
# SSH to instance
ssh -i nodejs-app-key.pem ec2-user@<instance-ip>

# Check agent status
sudo systemctl status codedeploy-agent

# If not running, start it
sudo systemctl start codedeploy-agent
sudo systemctl enable codedeploy-agent

# Check logs if issues persist
sudo tail -f /var/log/aws/codedeploy-agent/codedeploy-agent.log
```

### Issue: Node.js or npm Not Found

**Symptoms:**
- AfterInstall hook fails
- Error: "npm: command not found"

**Solutions:**
```bash
# Install Node.js 20
sudo dnf install -y nodejs npm

# Verify installation
node --version
npm --version
```

### Issue: Permission Denied Errors

**Symptoms:**
- Scripts fail with "Permission denied"

**Solutions:**
```bash
# On local machine before creating deployment package
chmod +x scripts/*.sh

# Or on EC2 instance after deployment
ssh -i nodejs-app-key.pem ec2-user@<instance-ip>
chmod +x /var/www/nodejs-app/scripts/*.sh
```

### Issue: Frontend Not Accessible

**Symptoms:**
- Backend works but frontend shows 404 or doesn't load

**Solutions:**
```bash
# Check if frontend was built
ls -la /var/www/nodejs-app/frontend/dist/

# Check NGINX configuration
sudo nginx -t

# Restart NGINX
sudo systemctl restart nginx

# Check NGINX logs
sudo tail -f /var/log/nginx/error.log
```

### Issue: WebSocket Connection Fails

**Symptoms:**
- Real-time features don't work
- Browser console shows WebSocket errors

**Solutions:**
```bash
# Verify NGINX WebSocket configuration
sudo cat /etc/nginx/conf.d/nodejs-app.conf

# Ensure upgrade headers are present:
# proxy_set_header Upgrade $http_upgrade;
# proxy_set_header Connection "upgrade";

# Check security group allows WebSocket traffic
# Ensure port 80 is open for HTTP/WebSocket
```

### Issue: Deployment Succeeds but Old Version Still Showing

**Symptoms:**
- Deployment shows success
- Frontend still shows old content

**Solutions:**
```bash
# Clear browser cache (hard refresh)
# Ctrl+Shift+R (Windows/Linux) or Cmd+Shift+R (Mac)

# Verify deployment on server
ssh -i nodejs-app-key.pem ec2-user@<instance-ip>

# Check PM2 process start time
pm2 status

# Restart application
pm2 restart backend

# Verify build files
ls -lt /var/www/nodejs-app/frontend/dist/ | head
```

***

## Appendix B: Environment Variables Reference

### Backend Environment Variables

Create or update `.env` file in your project root:

```bash
# Application Configuration
NODE_ENV=production
PORT=8000

# JWT Configuration
JWT_SECRET=your-secure-secret-here-generate-with-script
JWT_EXPIRY=24h

# Database Configuration
DB_PATH=./data/database.sqlite

# CORS Configuration
CORS_ORIGIN=*

# Deployment Information (auto-populated by CodeDeploy)
STAGE_VALUE=${DEPLOYMENT_GROUP_NAME}
APP_VERSION=${DEPLOYMENT_ID}
INSTANCE_ID=${EC2_INSTANCE_ID}
```

### Frontend Environment Variables

Create `frontend/.env.production`:

```bash
VITE_API_URL=/api
VITE_SOCKET_URL=
```

### Generating Secure JWT Secret

```bash
cd /path/to/deploy_to_builder_lab/backend
npm run generate-secret
```

This runs the script defined in `backend/scripts/generateSecret.js` to create a cryptographically secure secret.

***

## Appendix C: NGINX Configuration Reference

Complete NGINX configuration for Node.js full-stack app:

```nginx
# /etc/nginx/conf.d/nodejs-app.conf

server {
    listen 80;
    server_name _;

    # Logging
    access_log /var/log/nginx/nodejs-app-access.log;
    error_log /var/log/nginx/nodejs-app-error.log;

    # Frontend - serve static files
    location / {
        root /var/www/nodejs-app/frontend/dist;
        try_files $uri $uri/ /index.html;

        # Cache static assets
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    # Backend API
    location /api {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # WebSocket support for Socket.IO
    location /socket.io {
        proxy_pass http://localhost:8000/socket.io;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # WebSocket timeouts (longer for persistent connections)
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }

    # Health check endpoint
    location /health {
        proxy_pass http://localhost:8000/health;
        access_log off;
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

***

## Appendix D: PM2 Configuration (Optional)

For more advanced PM2 configuration, create `ecosystem.config.js`:

```javascript
module.exports = {
  apps: [{
    name: 'backend',
    script: './src/index.js',
    cwd: '/var/www/nodejs-app/backend',
    instances: 1,
    exec_mode: 'fork',
    env: {
      NODE_ENV: 'production',
      PORT: 8000
    },
    error_file: '/var/log/nodejs-app/backend-error.log',
    out_file: '/var/log/nodejs-app/backend-out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    merge_logs: true,
    autorestart: true,
    watch: false,
    max_memory_restart: '500M',
    min_uptime: '10s',
    max_restarts: 10
  }]
};
```

Use with:
```bash
pm2 start ecosystem.config.js
```

***

## Appendix E: Useful AWS CLI Commands

### CodeDeploy Commands

```bash
# List applications
aws deploy list-applications

# List deployments for a deployment group
aws deploy list-deployments \
  --application-name nodejs-fullstack-app \
  --deployment-group-name DevTest

# Get deployment details
aws deploy get-deployment \
  --deployment-id d-XXXXXXXXX

# Stop a deployment
aws deploy stop-deployment \
  --deployment-id d-XXXXXXXXX \
  --auto-rollback-enabled

# Create deployment from CLI
aws deploy create-deployment \
  --application-name nodejs-fullstack-app \
  --deployment-group-name DevTest \
  --s3-location bucket=$CODE_BUCKET,key=nodejs-app-v1.0.zip,bundleType=zip \
  --description "Deployment from CLI"
```

### S3 Commands

```bash
# List buckets
aws s3 ls

# List objects in bucket
aws s3 ls s3://$CODE_BUCKET/

# Copy file to S3
aws s3 cp file.zip s3://$CODE_BUCKET/

# Sync directory to S3
aws s3 sync ./local-dir s3://$CODE_BUCKET/remote-dir/

# Delete object
aws s3 rm s3://$CODE_BUCKET/file.zip

# Get object metadata
aws s3api head-object \
  --bucket $CODE_BUCKET \
  --key nodejs-app-v1.0.zip
```

### EC2 Commands

```bash
# List instances
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=DevTest-Instance" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' \
  --output table

# Stop instance
aws ec2 stop-instances --instance-ids i-XXXXXXXXX

# Start instance
aws ec2 start-instances --instance-ids i-XXXXXXXXX

# Get instance metadata (run from instance)
curl http://169.254.169.254/latest/meta-data/
```

***

## Appendix F: Security Best Practices

### 1. Security Group Configuration

**Principle of Least Privilege:**
- Only open ports that are necessary
- Restrict SSH access to your IP address
- Use security group rules instead of iptables when possible

**Recommended Rules:**

For DevTest:
```
SSH (22): Your IP only
HTTP (80): 0.0.0.0/0
```

For Production:
```
SSH (22): Your IP or bastion host only
HTTP (80): 0.0.0.0/0 or Load Balancer security group
HTTPS (443): 0.0.0.0/0 or Load Balancer security group
```

### 2. Environment Variable Security

**Never commit sensitive data to Git:**
```bash
# Add to .gitignore
echo ".env" >> .gitignore
echo "*.pem" >> .gitignore
```

**Use AWS Systems Manager Parameter Store for secrets:**
```bash
# Store secret
aws ssm put-parameter \
  --name "/nodejs-app/jwt-secret" \
  --value "your-secret-here" \
  --type "SecureString"

# Retrieve in application
aws ssm get-parameter \
  --name "/nodejs-app/jwt-secret" \
  --with-decryption \
  --query 'Parameter.Value' \
  --output text
```

### 3. IAM Best Practices

- Use role-based access instead of access keys
- Enable MFA for AWS Console access
- Regularly rotate credentials
- Follow principle of least privilege for IAM policies

### 4. Application Security

```bash
# Update packages regularly
npm audit
npm audit fix

# Use helmet for Express.js security headers
npm install helmet

# Enable HTTPS in production (use AWS Certificate Manager + ALB)
```

### 5. Monitoring and Logging

- Enable CloudTrail for API call logging
- Use CloudWatch for application monitoring
- Set up billing alerts to prevent unexpected charges
- Regularly review AWS Trusted Advisor recommendations

***

## Feedback and Support

If you encounter issues or have suggestions for improving this guide:

- Check the **Common Issues and Solutions** (Appendix A)
- Review AWS documentation linked in Additional Resources
- Contact AWS Support (free tier includes basic support)
- Consult AWS Community Forums

**Remember:** Always clean up resources after completing the guide to avoid unexpected charges.

***

**End of Guide**

Thank you for following this comprehensive AWS deployment guide! You now have the knowledge to deploy and manage Node.js full-stack applications on AWS infrastructure.
