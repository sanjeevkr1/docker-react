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
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:

    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: 'us-east-1'

      - name: Deploy to EC2
        run: |
          INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=YOUR_INSTANCE_NAME" --query "Reservations[].Instances[].InstanceId" --output text)
          PUBLIC_IP=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=YOUR_INSTANCE_NAME" --query "Reservations[].Instances[].PublicIpAddress" --output text)
          DOMAIN_NAME="yourdomain.com"

          # SSH into the instance and pull the latest Docker image
          ssh -o StrictHostKeyChecking=no -i "YOUR_KEY_PAIR.pem" ec2-user@$PUBLIC_IP << EOF
            docker pull your_dockerhub_username/your_image_name:latest
            docker stop frontend || true
            docker rm frontend || true
            docker run -d --name frontend -p 80:80 your_dockerhub_username/your_image_name:latest
          EOF

          # Update Route 53 record
          HOSTED_ZONE_ID="YOUR_HOSTED_ZONE_ID"
          CHANGE_BATCH="{\"Changes\":[{\"Action\":\"UPSERT\",\"ResourceRecordSet\":{\"Name\":\"$DOMAIN_NAME\",\"Type\":\"A\",\"TTL\":300,\"ResourceRecords\":[{\"Value\":\"$PUBLIC_IP\"}]}}]}"
          aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch "$CHANGE_BATCH"
```

## Deployment Process

1. **Push Code to GitHub**:
   - Commit and push your code to the `main` branch of your GitHub repository.

2. **GitHub Actions Build**:
   - On push, the build workflow triggers:
     - Installs dependencies
     - Builds the React app
     - Builds and pushes the Docker image to Docker Hub

3. **GitHub Actions Deploy**:
   - The deploy workflow triggers:
     - SSH into the EC2 instance
     - Pulls the latest Docker image
     - Stops and removes the old container
     - Runs the new container
     - Updates the Route 53 DNS record

4. **Access the Application**:
   - Once the deployment is complete, access your application using the EC2 public IP or your custom domain.

## Troubleshooting

- **Common Issues**:
  - If the app doesn't load, check the EC2 instance security group and ensure ports 80, 443, and 22 are open.
  - For Docker issues, SSH into the EC2 instance and check the Docker logs: `docker logs container_id`.

- **GitHub Actions Failures**:
  - Check the Actions tab in your GitHub repository for workflow run details.
  - Ensure all secrets (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, DOCKER_HUB_USERNAME, DOCKER_HUB_ACCESS_TOKEN) are correctly set in the repository settings.

- **Further Debugging**:
  - Use `aws ssm send-command` to run commands on the EC2 instance for debugging.
  - Check the Nginx configuration and logs if there are issues with serving the React app.

---

By following this tutorial, you should have a React application fully containerized with Docker, and automatically deployed to AWS EC2 using GitHub Actions. This setup ensures a smooth and efficient development to production workflow, leveraging modern tools and best practices.

#### Key Points:

* Builder stage: Node image to build the React app
* Final stage: Lightweight nginx image to serve the built app
* Port 80: Application is exposed on HTTP port 80

**nginx.conf**

Configuration for serving the React static files:
```
### GitHub Repository Setup
Step 1: Initialize Git Repository
If not already a git repo:
````
Step 2: Create GitHub Repository
1. Go to GitHub
2. Create a new repository named docker-react
3. Do NOT initialize with README, .gitignore, or license

Step 3: Push to GitHub

### AWS Configuration
Step 1: Create EC2 Instance
1. Go to AWS Console → EC2 Dashboard → Instances
2. Click Launch instances
3. Name: react-app
4. AMI: Select Amazon Linux 2 (or Amazon Linux AMI)
5. Instance Type: t2.micro (eligible for free tier)
6. Key Pair:
    * Create new key pair named react-app-key
    * Download the .pem file and store safely
7. Network settings:
    * Check "Allow HTTP traffic"
    * Check "Allow HTTPS traffic"
    * Do NOT allow SSH (we'll use Systems Manager)
8. Storage: Keep default (8 GB)
9. Click Launch instance

Step 2: Tag Your EC2 Instance
1. Go to EC2 Dashboard → Instances
2. Select your instance
3. Click Tags tab
4. Click Manage Tags
5. Add new tag:
    * Key: instance-name
    * Value: react-app
6. Click Save

Step 3: Create IAM Role for EC2
1. Go to IAM → Roles → Create Role
2. Select AWS Service → EC2
3. Search and attach: AmazonSSMManagedInstanceCore
4. Role name: EC2-SSM-Deploy-Role
5. Click Create role

Step 4: Attach Role to EC2 Instance
1. Go to EC2 Dashboard → Instances
2. Select your instance → Instance Details
3. Click Security tab
4.Click the IAM instance profile → Modify IAM Instance Profile
5. Select EC2-SSM-Deploy-Role
6. Click Update
7. Reboot the instance (right-click → Reboot instance)

Step 5: Verify SSM Agent
1. Go to AWS Systems Manager → Session Manager
2. Click Start session
3. Select your react-app instance
4. If successful, you're in a shell on your instance

Verify SSM agent:
```bash
sudo systemctl status amazon-ssm-agent
```
If not running:

```bash
sudo systemctl start amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
```
Step 6: Setup OIDC Provider for GitHub Actions
1. Go to IAM → Identity providers → Add provider
2. Provider type: OpenID Connect
3. Provider URL: https://token.actions.githubusercontent.com
4. Audience: sts.amazonaws.com
5. Click Add provider

Step 7: Create IAM Role for GitHub Actions
1. Go to IAM → Roles → Create Role
2. Trusted entity type: Web identity
3. Identity provider: token.actions.githubusercontent.com
4. Audience: sts.amazonaws.com
5. Click Next
6. Role name: GitHubActionsRole
7. Click Create role (skip adding permissions for now)

Step 8: Add Inline Policy to GitHub Actions Role
1. Go to IAM → Roles → Select GitHubActionsRole
2. Click Add permissions → Create inline policy
3. Select JSON tab
4. Paste this policy (replace ACCOUNT_ID):
````
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeTags"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:SendCommand"
      ],
      "Resource": [
        "arn:aws:ssm:*::document/*",
        "arn:aws:ec2:*:ACCOUNT_ID:instance/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetCommandInvocation",
        "ssm:ListCommandInvocations",
        "ssm:ListCommands"
      ],
      "Resource": "*"
    }
  ]
}
````
5. Click Review policy → Create policy
6. Policy name: GitHubActionsPolicy

Step 9: Update Trust Relationship
1. In GitHubActionsRole, click Trust relationships
2. Click Edit trust policy
3. Replace with this (replace ACCOUNT_ID, YOUR_USERNAME, YOUR_REPO):
````
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_USERNAME/docker-react:ref:refs/heads/*"
        }
      }
    }
  ]
}
````
4. Click Update policy
---------------------------------------------------
### GitHub Actions Workflows
Step 1: Create Workflow Files Structure
Create the following directory structure:
````bash
mkdir -p .github/workflows/ssm-commands
````

