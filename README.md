**DevOps Cloud Assignment**

_Implementation Documentation_

**By: Malav Shah**

**Email ID:** [**shahmalav1999@gmail.com**](mailto:shahmalav1999@gmail.com)

**Contact Details: +91 8355817127**

Cloud Platform: Amazon Web Services (AWS) | Region: ap-south-1 (Mumbai)

Repository: <https://github.com/Malavshah999/devops-assignment-case1.git>

# 1\. Assignment Overview

This assignment covers two infrastructure cases, both deployed on Amazon Web Services (AWS) in the ap-south-1 (Mumbai) region. The goal was to design, implement, secure, and document a production-grade cloud infrastructure demonstrating DevOps best practices.

| **Case** | **Objective**                               | **Key Services Used**                    |
| -------- | ------------------------------------------- | ---------------------------------------- |
| Case 1   | Serverless Dockerized Service with JWT Auth | ECS Fargate, ALB, CloudFront, ECR, VPC   |
| Case 2   | Secure Public Bucket Storage via CDN only   | S3, CloudFront OAC, IAM, Bucket Policies |

# 2\. Pre-requisites

| **Tool**       | **Version** | **Purpose**                      |
| -------------- | ----------- | -------------------------------- |
| Git            | 2.44+       | Version control for all code     |
| Node.js        | v20.x LTS   | Runtime for Express application  |
| Docker Desktop | 4.29+       | Build and run containers locally |
| AWS CLI        | v2          | Interact with AWS from terminal  |

# 3\. Case 1 - Serverless Dockerized Service

## 3.1 Architecture Overview

The application is designed so that the container running the actual code is NEVER directly reachable from the internet. All traffic must flow through the CDN and Load Balancer layers.

<IMAGE HERE>

Fig (3.1 Figma chart of Architecture overview)

| **Layer** | **Component**                   | **Subnet Type** | **Purpose**                                     |
| --------- | ------------------------------- | --------------- | ----------------------------------------------- |
| Edge      | CloudFront CDN                  | Global Edge     | CDN caching, HTTPS termination, DDoS protection |
| Public    | Application Load Balancer (ALB) | Public Subnet   | Routes traffic, health checks, SSL offload      |
| Private   | ECS Fargate Container           | Private Subnet  | Runs the Express.js application                 |
|           |                                 |                 |                                                 |

## 3.2 Application Code

A Node.js Express application was built with the following features:

| **Endpoint** | **Method** | **Auth Required** | **Description**                                      |
| ------------ | ---------- | ----------------- | ---------------------------------------------------- |
| /health      | GET        | No                | Health check for ALB                                 |
| /login       | POST       | No                | Accepts username+password, returns JWT token         |
| /items       | GET        | Yes (Bearer JWT)  | Protected list - returns items only with valid token |

**Authentication Flow**

- User sends /login with { username: 'admin', password: 'password123' }
- Server validates credentials against hardcoded mock user store
- On success, server signs a JWT token with a secret key (1 hour expiry)
- Client includes token in subsequent requests: Authorization: Bearer &lt;token&gt;
- authMiddleware verifies token on every protected route
- Invalid/missing token returns 401 Unauthorized

## 3.3 Docker Configuration

The application was containerized using a multi-security-layer Dockerfile:

## 3.4 AWS Infrastructure Setup

**Step 1 - VPC and Network Configuration**

A dedicated Virtual Private Cloud (VPC) was created to isolate all resources:

| **Resource**          | **CIDR / Config**         | **Purpose**                                            |
| --------------------- | ------------------------- | ------------------------------------------------------ |
| VPC (devops-vpc)      | 10.0.0.0/16               | Isolated network for all resources                     |
| Private Subnet        | 10.0.1.0/24 - ap-south-1a | ECS Fargate tasks run here - no direct internet        |
| Public Subnet         | 10.0.2.0/24 - ap-south-1b | ALB and NAT Gateway live here                          |
| Internet Gateway      | Attached to VPC           | Allows public subnets to reach internet                |
| NAT Gateway           | In Public Subnet          | Allows private subnet to pull ECR images outbound      |
| Route Table (Public)  | 0.0.0.0/0 -> IGW          | Routes internet traffic for public subnets             |
| Route Table (Private) | 0.0.0.0/0 -> NAT GW       | Routes outbound traffic through NAT for private subnet |
|                       |                           |                                                        |

**Step 2 - Amazon ECR (Container Registry)**

Amazon Elastic Container Registry was created to store Docker images:

- Repository name: devops-case1
- Region: ap-south-1
- URI: 417492524162.dkr.ecr.ap-south-1.amazonaws.com/devops-case1
- Images tagged as :latest on every CI/CD pipeline run

**Step 3 - ECS Fargate Cluster and Service**

| **ECS Component** | **Configuration**                                          |
| ----------------- | ---------------------------------------------------------- |
| Cluster Name      | my-cluster                                                 |
| Service Name      | my-service                                                 |
| Launch Type       | FARGATE (serverless - no EC2 instances to manage)          |
| Task CPU          | 256 (.25 vCPU)                                             |
| Task Memory       | 512 MB                                                     |
| Container Port    | 3000                                                       |
| Network Mode      | awsvpc (each task gets its own ENI)                        |
| Subnet            | Private subnet (10.0.1.0/24)                               |
| Public IP         | DISABLED - container is not internet-accessible            |
| Desired Count     | 1 running task                                             |
| Execution Role    | ecsTaskExecutionRole with AmazonECSTaskExecutionRolePolicy |

**Step 4 - Application Load Balancer**

| **ALB Setting**       | **Value**                             |
| --------------------- | ------------------------------------- |
| Name                  | devops-alb                            |
| Scheme                | Internet-facing                       |
| Subnets               | Both public subnets (two AZs for HA)  |
| Listener HTTP (80)    | Forward to Target Group devops-tg     |
| Target Group Protocol | HTTP on port 3000                     |
| Target Type           | IP (required for Fargate awsvpc mode) |
| Health Check Path     | /health                               |
| Health Check Interval | 30 seconds                            |
| Healthy Threshold     | 2 consecutive successes               |
|                       |                                       |

**Step 5 - Security Groups**

| **Security Group**  | **Inbound Rule**                 | **Purpose**                                         |
| ------------------- | -------------------------------- | --------------------------------------------------- |
| devops-alb-sg (ALB) | HTTP 80 from 0.0.0.0/0           | Allow web traffic to reach ALB                      |
| devops-alb-sg (ALB) | HTTPS 443 from 0.0.0.0/0         | Allow secure traffic to reach ALB                   |
| devops-ecs-sg (ECS) | TCP 3000 from devops-alb-sg only | Only ALB can reach containers - not public internet |

## 3.5 CI/CD Pipeline

A GitHub Actions workflow was implemented to fully automate the build and deployment process on every code push.
<IMAGE HERE>

| **Pipeline Stage** | **Action**                                            | **Tool**                                      |
| ------------------ | ----------------------------------------------------- | --------------------------------------------- |
| Trigger            | Any push to main branch                               | GitHub Actions                                |
| Checkout           | Pull latest code from repository                      | actions/checkout@v4                           |
| Docker Hub Auth    | Check-in to docker hub via variables stored in repo   | docker/login-action@v3                        |
| AWS Auth           | Authenticate using IAM access keys from secrets       | aws-actions/configure-aws-credentials@v4      |
| ECR Login          | Authenticate Docker to push to ECR                    | aws-actions/amazon-ecr-login@v2               |
| Docker Build       | Build image from Dockerfile in ./app                  | docker build command                          |
| Docker Push        | Push image tagged :latest to ECR repository           | docker push command                           |
| ECS Deploy         | Force new deployment service - ECS pulls latest image | aws ecs update-service --force-new-deployment |

**GitHub Actions Secrets Configured**

- AWS_ACCESS_KEY_ID - IAM user access key for CI/CD deployments
- AWS_SECRET_ACCESS_KEY - IAM user secret key
- DOCKERHUB_USERNAME - Docker Hub username (malavsh)
- DOCKERHUB_TOKEN - Docker Hub personal access token

**IAM Permissions Issue Resolved**

During setup, the IAM user MalavTest encountered permission errors because it lacked the necessary policies. This was resolved by:

Logging in with the root AWS account

Checking for and removing any permissions boundary on the user

Attaching the following managed policies:

AmazonEC2ContainerRegistryFullAccess,

AmazonECS_FullAccess,

AmazonVPCFullAccess,

ElasticLoadBalancingFullAccess,

CloudFrontFullAccess,

AmazonS3FullAccess,

SecretsManagerReadWrite,

CloudWatchFullAccess

## 3.6 CloudFront Configuration

| **CloudFront Setting**   | **Value**                          |
| ------------------------ | ---------------------------------- |
| Origin                   | ALB DNS name                       |
| Cache Behavior - /health | Cache enabled (static response)    |
| Cache Behavior - /login  | No cache (dynamic, auth endpoint)  |
| Cache Behavior - /items  | No cache (dynamic, protected data) |
|                          |                                    |
|                          |                                    |

# 4\. Case 2 - Secure Public Bucket Storage

## 4.1 Architecture Overview

The objective was to serve static content (HTML, CSS, images) to the public WITHOUT ever allowing direct access to the S3 bucket URL. All access must go through CloudFront using Origin Access Control.
<IMAGE HERE>

## 4.2 S3 Bucket Configuration

- Bucket name: devops-case2-static-ms
- Region: ap-south-1
- All public access: BLOCKED (BlockPublicAcls, IgnorePublicAcls, BlockPublicPolicy, RestrictPublicBuckets all set to true)
- Server-side encryption: SSE-S3 enabled (AES-256)
- Versioning: Enabled for object recovery
- Access logging: Enabled, logs sent to separate logging bucket

**Sample Static Content Uploaded**

- index.html - Main landing page
- style.css - Stylesheet
- logo.png - Sample image asset

## 4.3 CloudFront OAC Setup

Origin Access Control (OAC) is the modern, secure method for CloudFront to authenticate with S3. It uses AWS Signature Version 4 signing, which is more secure than the older Origin Access Identity (OAI).

## 4.4 Security Controls Implemented

| **Security Control** | **Implementation**                                     | **Benefit**                       |
| -------------------- | ------------------------------------------------------ | --------------------------------- |
| Bucket Policies      | Only CloudFront OAC principal allowed in bucket policy | Prevents any direct S3 access     |
| Public Access Block  | All 4 public access block settings enabled             | No accidental public exposure     |
| Encryption at Rest   | SSE-S3 (AES-256) enabled by default                    | Data encrypted on disk            |
| Versioning           | Object versioning enabled                              | Recovery from accidental deletion |
| Access Logging       | S3 server access logs to separate bucket               | Audit trail of all requests       |
|                      |                                                        |                                   |

# 5\. Security Implementation Summary

## 5.1 Case 1 Security Controls

| **Security Measure**      | **How Implemented**                                | **Threat Mitigated**                  |
| ------------------------- | -------------------------------------------------- | ------------------------------------- |
| Private Network Isolation | ECS tasks in private subnet, no public IP          | Direct container exposure             |
| JWT Authentication        | jsonwebtoken library, 1-hour expiry, Bearer token  | Unauthorized API access               |
| Non-root Container        | Custom user created in Dockerfile, USER appuser    | Container escape privilege escalation |
| Minimal Base Image        | node:20-alpine (small attack surface)              | Unpatched OS vulnerabilities          |
| Security Groups           | ECS only accepts traffic from ALB SG on port 3000  | Port scanning, direct access          |
| Environment Secrets       | JWT_SECRET via environment variable, not hardcoded | Secret exposure in source code        |
|                           |                                                    |                                       |
|                           |                                                    |                                       |

## 5.2 Case 2 Security Controls

| **Security Measure**       | **How Implemented**                              | **Threat Mitigated**            |
| -------------------------- | ------------------------------------------------ | ------------------------------- |
| Bucket Public Access Block | All 4 block settings enabled in S3               | Accidental public exposure      |
| OAC Authentication         | CloudFront signs all S3 requests with SigV4      | Unauthorized S3 access          |
| Restrictive Bucket Policy  | Only CloudFront service principal allowed        | Direct bucket URL access        |
| Encryption at Rest         | SSE-S3 AES-256 server-side encryption            | Data breach from storage access |
| HTTPS Enforcement          | CloudFront viewer policy: redirect HTTP to HTTPS | Man-in-the-middle attacks       |
| Access Logging             | S3 access logs to dedicated logging bucket       | Audit trail, anomaly detection  |

# 6\. Cost Estimate

## 6.1 Monthly Cost at Low Traffic

| **AWS Service**           | **Configuration**                 | **Est. Monthly Cost** |
| ------------------------- | --------------------------------- | --------------------- |
| ECS Fargate               | 0.25 vCPU, 0.5GB RAM, 1 task 24/7 | ~\$8 - \$12           |
| Application Load Balancer | 1 ALB, minimal LCU usage          | ~\$16 - \$18          |
| NAT Gateway               | 1 NAT GW + data processing        | ~\$32 - \$35          |
| Amazon ECR                | 1GB image storage                 | ~\$0.10               |
| CloudFront (Case 1)       | 10GB transfer/month               | ~\$0.85               |
| S3 (Case 2)               | 1GB storage + requests            | ~\$0.05               |
| CloudFront (Case 2)       | 10GB transfer/month               | ~\$0.85               |
| TOTAL                     |                                   | ~\$60 - \$70 / month  |

**Cost Optimization Recommendations**

- Use VPC Endpoints for ECR to eliminate NAT Gateway data transfer costs - saves ~\$15-20/month at scale
- Switch ECS to Fargate Spot for non-critical workloads - up to 70% cost reduction
- Set CloudFront cache TTLs to maximize cache hit ratio and reduce origin requests
- Use S3 Intelligent-Tiering for Case 2 content if access patterns vary
- Delete NAT Gateway after assignment to avoid ongoing charges (~\$32/month)

# 7\. Problems Encountered and Solutions

| **Problem**                           | **Root Cause**                                   | **Solution Applied**                                   |
| ------------------------------------- | ------------------------------------------------ | ------------------------------------------------------ |
| PowerShell script error on npm        | Windows blocks PS scripts by default             | Set-ExecutionPolicy RemoteSigned as Admin              |
| VPC not appearing in ALB dropdown     | Missing Internet Gateway and route tables        | Created IGW, attached to VPC, added public route table |
| ECR AccessDeniedException             | IAM user had no ECR permissions                  | Attached AmazonEC2ContainerRegistryFullAccess policy   |
| AttachUserPolicy AccessDenied         | Permissions boundary on IAM user                 | Logged in as root and removed boundary                 |
| ECS ClusterNotFoundException in CI/CD | ECS cluster not created before pipeline ran      | Created cluster via aws ecs create-cluster             |
| ECS ServiceNotFoundException          | ECS service not created before pipeline ran      | Created service via aws ecs create-service             |
| runningCount 0 - tasks not starting   | Private subnet could not reach ECR to pull image | Created NAT Gateway in public subnet, added route      |

**PowerShell Execution Policy Fix**

On a fresh Windows machine, npm commands were blocked by default PowerShell security policy. This was resolved by running the following command as Administrator:

Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned

This allows locally created scripts to run while requiring downloaded scripts to be signed - a safe middle ground for development.

**Missed creating a NAT Gateway**

ECS Fargate tasks in private subnets cannot pull Docker images from ECR without internet access. The error encountered was:

ResourceInitializationError: unable to pull registry auth: dial tcp 13.234.8.23:443: i/o timeout

Solution: A NAT Gateway was created in the public subnet and a route was added to the private subnet route table pointing all outbound traffic (0.0.0.0/0) through the NAT Gateway. This allows the private container to pull images from ECR without being directly exposed to the internet.

**End of Documentation**

_DevOps Cloud Infrastructure Assignment - by Malav_