# Docker & GitHub Actions Deployment - Complete Tutorial

This tutorial walks through the entire process of setting up a React application with Docker containerization and automated deployment to AWS EC2 using GitHub Actions.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Local Development Setup](#local-development-setup)
4. [Docker Configuration](#docker-configuration)
5. [GitHub Repository Setup](#github-repository-setup)
6. [AWS Configuration](#aws-configuration)
7. [GitHub Actions Workflows](#github-actions-workflows)
8. [Deployment Process](#deployment-process)
9. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before starting, ensure you have the following installed:

- **Node.js** (v18.x or later) - [Download](https://nodejs.org/)
- **npm** (comes with Node.js)
- **Docker Desktop** - [Download](https://www.docker.com/products/docker-desktop)
- **Git** - [Download](https://git-scm.com/)
- **AWS Account** - [Create Free Account](https://aws.amazon.com/free/)
- **Docker Hub Account** - [Create Account](https://hub.docker.com/)
- **GitHub Account** - [Create Account](https://github.com/)

### Verify Installation

```bash
node --version
npm --version
docker --version
git --version
```

### Project Structure
Your project should have the following structure:
````markdown
frontend/
├── .github/
│   └── workflows/
│       ├── build.yaml                    # Build workflow
│       ├── deploy.yaml                   # Deploy workflow
│       └── ssm-commands/
│           ├── check-docker.json         # Docker install/check commands
│           ├── pull-image.json           # Image pull commands
│           ├── deploy-app.json           # App deployment commands
│           └── sanity-check.json         # Health check commands
├── public/
│   ├── index.html
│   ├── manifest.json
│   └── robots.txt
├── src/
│   ├── App.js
│   ├── App.css
│   ├── App.test.js
│   ├── index.js
│   ├── index.css
│   └── ...
├── Dockerfile                           # Production Docker image
├── Dockerfile.dev                       # Development Docker image
├── docker-compose.yml                   # Local Docker Compose
├── nginx.conf                           # Nginx configuration
├── package.json                         # npm package configuration
├── package-lock.json
├── .gitignore
├── .dockerignore
└── README.md
````

### Local Development Setup
Step 1: Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/docker-react.git
cd frontend
```
Step 2: Install Dependencies

```bash
npm install
```
Step 3: Start Development Server

```bash
npm start
```
The app will open at http://localhost:3000

Step 4: Using Docker Compose for Local Development
Build and run the application using Docker Compose:

```bash
docker-compose up --build
```
This will:

* Build the dev image from Dockerfile.dev
* Mount your source code as a volume
* Enable live reloading when you edit files
* Expose the app on http://localhost:3001

Stop the containers:

```bash
docker-compose down
```
### Docker Configuration
Dockerfile.dev (Development)
Used for local development with live reloading:

```dockerfile
FROM node:lts-alpine

WORKDIR '/app'

COPY package.json .
RUN npm install

COPY . .

CMD ["npm", "run", "start"]
```

### Dockerfile (Production)
Multi-stage build for production deployment:

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm install

COPY . .
RUN npm run build

FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### docker-compose.yml
Configuration for local development with Docker Compose:

```yaml
version: '3.8'

services:
  frontend:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - '3001:3000'
    volumes:
      - .:/app
    environment:
      - CHOKIDAR_USEPOLLING=true
```

### nginx.conf
Basic Nginx configuration for serving the React app:

```nginx
server {
  listen 80;

  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html;
  }
}
```

## GitHub Repository Setup

Create a new repository on GitHub and push your local project to this repository.

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/docker-react.git
git push -u origin main
```

## AWS Configuration

1. **Create an EC2 Instance**:
   - Use the Amazon Linux 2 AMI.
   - Choose an instance type (e.g., t2.micro for free tier).
   - Configure security group to allow HTTP (80), HTTPS (443), and SSH (22) access.

2. **Install Docker on EC2**:

```bash
sudo yum update -y
sudo amazon-linux-extras install docker
sudo service docker start
sudo usermod -a -G docker ec2-user
```

3. **Configure Docker Hub Credentials** (optional, if using private repos):

```bash
aws ssm put-parameter --name "DOCKER_HUB_USERNAME" --value "your_docker_hub_username" --type "String" --overwrite
aws ssm put-parameter --name "DOCKER_HUB_ACCESS_TOKEN" --value "your_docker_hub_access_token" --type "String" --overwrite
```

## GitHub Actions Workflows

### Build Workflow (`.github/workflows/build.yaml`)

```yaml
name: Build

on:
  push:
    branches:
      - main

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm install

      - name: Build
        run: npm run build

      - name: Docker Build
        run: |
          docker build . -t your_dockerhub_username/your_image_name:latest
          echo "${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}" | docker login -u "${{ secrets.DOCKER_HUB_USERNAME }}" --password-stdin
          docker push your_dockerhub_username/your_image_name:latest
```

### Deploy Workflow (`.github/workflows/deploy.yaml`)

```yaml
name: Deploy to AWS EC2
on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}
      
      - name: Get EC2 Instance ID by tag
        id: get-instance
        run: |
          INSTANCE_ID=$(aws ec2 describe-instances \
            --filters "Name=tag:instance-name,Values=react-app" "Name=instance-state-name,Values=running" \
            --query 'Reservations[0].Instances[0].InstanceId' \
            --output text)
          echo "instance_id=$INSTANCE_ID" >> $GITHUB_OUTPUT
          echo "Instance ID: $INSTANCE_ID"
      
      - name: Verify AWS credentials
        run: aws sts get-caller-identity

      - name: Check and Install Docker
        run: |
          ID=${{ steps.get-instance.outputs.instance_id }}
          echo "Checking Docker installation on instance: $ID"

          DOCKER_CMD_ID=$(aws ssm send-command \
            --instance-ids "$ID" \
            --document-name "AWS-RunShellScript" \
            --parameters file://.github/workflows/ssm-commands/check-docker.json \
            --query "Command.CommandId" --output text)
          echo "Docker check/install Command ID: $DOCKER_CMD_ID"

          aws ssm wait command-executed --command-id "$DOCKER_CMD_ID" --instance-id "$ID"
          
          DOCKER_STATUS=$(aws ssm get-command-invocation \
            --command-id "$DOCKER_CMD_ID" \
            --instance-id "$ID" \
            --query 'Status' \
            --output text)
          
          echo "Docker command status: $DOCKER_STATUS"
          aws ssm get-command-invocation --command-id "$DOCKER_CMD_ID" --instance-id "$ID" --output text
          
          if [ "$DOCKER_STATUS" != "Success" ]; then
            echo "ERROR: Docker installation/check failed"
            exit 1
          fi

      - name: Pull Docker Image from Docker Hub
        run: |
          ID=${{ steps.get-instance.outputs.instance_id }}
          DOCKER_USER="${{ secrets.DOCKER_USERNAME }}"
          DEPLOY_PATH="${{ secrets.EC2_DEPLOY_PATH }}"

          echo "Pulling Docker image: $DOCKER_USER/react-app:latest"

          sed "s|\${DOCKER_USER}|$DOCKER_USER|g; s|\${DEPLOY_PATH}|$DEPLOY_PATH|g" \
            .github/workflows/ssm-commands/pull-image.json > /tmp/pull-image-resolved.json

          PULL_CMD_ID=$(aws ssm send-command \
            --instance-ids "$ID" \
            --document-name "AWS-RunShellScript" \
            --parameters file:///tmp/pull-image-resolved.json \
            --query "Command.CommandId" --output text)
          echo "Pull image Command ID: $PULL_CMD_ID"

          aws ssm wait command-executed --command-id "$PULL_CMD_ID" --instance-id "$ID"
          
          PULL_STATUS=$(aws ssm get-command-invocation \
            --command-id "$PULL_CMD_ID" \
            --instance-id "$ID" \
            --query 'Status' \
            --output text)
          
          echo "Pull image command status: $PULL_STATUS"
          aws ssm get-command-invocation --command-id "$PULL_CMD_ID" --instance-id "$ID" --output text
          
          if [ "$PULL_STATUS" != "Success" ]; then
            echo "ERROR: Docker image pull failed"
            exit 1
          fi

      - name: Deploy Application Container
        run: |
          ID=${{ steps.get-instance.outputs.instance_id }}
          DOCKER_USER="${{ secrets.DOCKER_USERNAME }}"
          DEPLOY_PATH="${{ secrets.EC2_DEPLOY_PATH }}"

          echo "Deploying application container"

          sed "s|\${DOCKER_USER}|$DOCKER_USER|g; s|\${DEPLOY_PATH}|$DEPLOY_PATH|g" \
            .github/workflows/ssm-commands/deploy-app.json > /tmp/deploy-app-resolved.json

          DEPLOY_CMD_ID=$(aws ssm send-command \
            --instance-ids "$ID" \
            --document-name "AWS-RunShellScript" \
            --parameters file:///tmp/deploy-app-resolved.json \
            --query "Command.CommandId" --output text)
          echo "Deploy app Command ID: $DEPLOY_CMD_ID"

          aws ssm wait command-executed --command-id "$DEPLOY_CMD_ID" --instance-id "$ID"
          
          DEPLOY_STATUS=$(aws ssm get-command-invocation \
            --command-id "$DEPLOY_CMD_ID" \
            --instance-id "$ID" \
            --query 'Status' \
            --output text)
          
          echo "Deploy command status: $DEPLOY_STATUS"
          aws ssm get-command-invocation --command-id "$DEPLOY_CMD_ID" --instance-id "$ID" --output text
          
          if [ "$DEPLOY_STATUS" != "Success" ]; then
            echo "ERROR: Application deployment failed"
            exit 1
          fi

      - name: Sanity Check - Verify App is Running
        run: |
          ID=${{ steps.get-instance.outputs.instance_id }}

          echo "Running sanity checks on deployed application"

          SANITY_CMD_ID=$(aws ssm send-command \
            --instance-ids "$ID" \
            --document-name "AWS-RunShellScript" \
            --parameters file://.github/workflows/ssm-commands/sanity-check.json \
            --query "Command.CommandId" --output text)
          echo "Sanity check Command ID: $SANITY_CMD_ID"

          aws ssm wait command-executed --command-id "$SANITY_CMD_ID" --instance-id "$ID"
          
          SANITY_STATUS=$(aws ssm get-command-invocation \
            --command-id "$SANITY_CMD_ID" \
            --instance-id "$ID" \
            --query 'Status' \
            --output text)
          
          echo "Sanity check status: $SANITY_STATUS"
          aws ssm get-command-invocation --command-id "$SANITY_CMD_ID" --instance-id "$ID" --output text
          
          if [ "$SANITY_STATUS" != "Success" ]; then
            echo "ERROR: Sanity check failed - application not responding with HTTP 200"
            exit 1
          fi
          
          echo "✓ All sanity checks passed - application is running and healthy"
```
-----------------------------------
### Deployment Process
Manual Deployment Steps
1. **Push code changes to GitHub**
```python
    git add .
    git commit -m "Your commit message"
    git push origin main
```
2. **Trigger Build Workflow**
* Go to your GitHub repository → Actions
* Select Build and Push to Docker Hub
* Click Run workflow → Run workflow
* Wait for the workflow to complete
* Verify image pushed to Docker Hub

3. **Trigger Deploy Workflow**
* Go to your GitHub repository → Actions
* Select Deploy to AWS EC2
* Click Run workflow → Run workflow
* Monitor the workflow execution:
    * ✅ Configure AWS credentials
    * ✅ Get EC2 Instance ID
    * ✅ Check and Install Docker
    * ✅ Pull Docker Image
    * ✅ Deploy Application Container
    * ✅ Sanity Check - Verify App is Running

4. **Access Your Application**
* Once deployment is complete, you can access your application at:  http://YOUR_EC2_PUBLIC_IP/

**To find your EC2 public IP:**

* Go to AWS EC2 Dashboard → Instances
* Select your react-app instance
* Copy the Public IPv4 address

**Full Deployment Workflow**

    Developer pushes code
            ↓
        GitHub Actions
            ↓
        Build Workflow
            ↓
    Run Tests in Docker
            ↓
    Build Production Image
            ↓
    Push to Docker Hub
            ↓
        Deploy Workflow
            ↓
    Get EC2 Instance ID (by tag)
            ↓
    Check/Install Docker on EC2
            ↓
    Pull Image from Docker Hub
            ↓
    Stop Old Container & Start New
            ↓
    Run Health Checks
            ↓
    App is Live! 🚀


-------------------------------------------------------
### Troubleshooting
* Issue: GitHub Actions - Credentials could not be loaded
    * **Cause**: Trust relationship issue with OIDC provider

    * Solution:

        1. Verify OIDC provider exists: IAM → Identity providers
        2. Check trust relationship includes correct sub pattern

            ` repo:YOUR_USERNAME/docker-react:ref:refs/heads/*`
        3. Verify AWS_ROLE_ARN secret is correct

    * Issue: Docker command not found on EC2
        * Cause: Docker not installed or SSM agent not running
        * Solution:
            1. Verify SSM agent is running:

                `sudo systemctl status amazon-ssm-agent`
            2. Check EC2 instance role has AmazonSSMManagedInstanceCore policy
            3. Verify EC2 instance is tagged correctly: instance-name=react-app

    * Issue: Application returns HTTP 500
        * Cause: Container exited or app failed to start

        * Solution:
            1. Check container logs via Systems Manager Session Manager
                `sudo docker logs react-app`
            2. Verify nginx.conf is correct
            3. Check application build completed successfully

    * Issue: Permission Denied - docker: command not found
        * Cause: Commands running as non-root without sudo

        * Solution: All docker commands use sudo in deployment scripts

    * Issue: EC2 Instance Not Found
        * Cause: Wrong tag name or instance not running

        * Solution:

            1. Verify instance tag: Name = instance-name, Value = react-app
            2. Verify instance is in Running state
            3. Check instance is in same region as workflow

    * Issue: Cannot Push to Docker Hub
        * Cause: Invalid Docker Hub credentials

        * Solution:

            1. Create Docker Hub personal access token:
                * Go to Docker Hub → Account Settings → Security
                * Generate new access token
            2. Update GitHub secret DOCKER_PASSWORD with token
            3. Verify username is correct

--------------------------------------------
### Useful Commands
**Local Development**
```python
    # Install dependencies
    npm install

    # Start development server
    npm start

    # Build for production
    npm run build

    # Run tests
    npm test
```
**Docker Commands**
```python
    # Connect to EC2 via Session Manager
    aws ssm start-session --target i-INSTANCE_ID

    # Get EC2 instances
    aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`instance-name`].Value]'

    # View SSM command results
    aws ssm get-command-invocation --command-id CMD_ID --instance-id i-INSTANCE_ID
```
---------------------------------------------
### Next Steps
1. **Set up CI/CD pipeline** - Add automatic builds on push
2. **Use Amazon ECR** - Store images in private ECR registry
3. **Add monitoring** - CloudWatch logs and alerts
4. **SSL/TLS** - Use AWS Certificate Manager for HTTPS
5. **Auto-scaling** - Use Auto Scaling Groups for multiple instances
6. **Load Balancing** - Use Application Load Balancer
7. **Database** - Add RDS for persistent data
