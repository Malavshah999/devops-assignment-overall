# DevOps Assignment — Submission

---

## Candidate Information

| Field | Details |
|---|---|
| **Full Name** | Malav Shah |
| **Email** | shahmalav199@gmail.com |
| **Phone** | +918355817127 |
| **Submission Date** | 26th March 2026 |
| **Cloud Platform** | Amazon Web Services (AWS) |
| **Region** | ap-south-1 (Mumbai) |

---

## Git Repository

| Case | Repository URL |
|---|---|
| Case 1 — Serverless Dockerized Service | https://github.com/Malavshah999/devops-assignment-case1.git |
| Case 2 — Secure Bucket Storage | Included in the same repository under `/case2/` |

---

## Live Endpoints

| Endpoint | URL |
|---|---|
| ALB Health Check | `http://devops-alb-YOUR_URL.ap-south-1.elb.amazonaws.com/health` |
| ALB Login | `http://devops-alb-YOUR_URL.ap-south-1.elb.amazonaws.com/login` |
| ALB Protected Items | `http://devops-alb-YOUR_URL.ap-south-1.elb.amazonaws.com/items` |
| CloudFront — Case 1 | `https://YOUR_URL.cloudfront.net/health` |
| CloudFront — Case 2 | `https://YOUR_URL.cloudfront.net/index.html` |

---

## Test Credentials

### Case 1 — API Login

```
Username: admin
Password: password123
```

**How to test step by step:**

**Step 1 — Get JWT token via login:**
```bash
curl -X POST http://YOUR_ALB_DNS/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password123"}'
```
Response will include a token like:
```json
{ "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.XXXXX" }
```

**Step 2 — Access protected endpoint with token:**
```bash
curl http://YOUR_ALB_DNS/items \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

**Step 3 — Verify unauthorized access is blocked:**
```bash
curl http://YOUR_ALB_DNS/items
# Expected: {"error":"Unauthorized"}
```

### Case 2 — Static Storage

No credentials required. Test by:

```bash
# This should return 403 (bucket is private)
curl -I https://devops-case2-static-ms.s3.ap-south-1.amazonaws.com/index.html

# This should return 200 (via CloudFront)
curl -I https://YYYYY.cloudfront.net/index.html
```

---

## Implementation Summary

### Case 1 — Serverless Dockerized Service

**What was built:**
REST API with JWT authentication, containerized with Docker, and deployed on AWS ECS Fargate in a fully private network.

**Architecture flow:**
```
User → CloudFront CDN → Application Load Balancer (public subnet) → ECS Fargate (private subnet)
```

**Key design decisions:**

- **ECS Fargate** was chosen over EC2 because it is serverless — no servers to patch, provision, or manage
- **Private subnet** for ECS tasks ensures the container is never directly reachable from the internet even if the security groups are misconfigured
- **NAT Gateway** in the public subnet allows ECS tasks to pull Docker images from ECR without being publicly exposed
- **JWT tokens** with 1-hour expiry provide stateless authentication — no session storage required
- **Non-root Docker user** limits blast radius if the container is ever compromised
- **node:20-alpine** base image is ~50MB versus ~900MB for full node — minimal attack surface

---

### Case 2 — Secure Public Bucket Storage

**What was built:**
A private S3 bucket serving static content exclusively through CloudFront using Origin Access Control (OAC). No direct S3 access is possible under any circumstance.

**Architecture flow:**
```
User → CloudFront CDN → S3 Bucket (private, OAC authenticated)
```

**Key design decisions:**

- **OAC (Origin Access Control)** was chosen over the older OAI (Origin Access Identity) because OAC uses AWS SigV4 signing which is more secure, supports SSE-KMS encryption, and is the current AWS recommendation
- **All 4 public access block settings** are enabled on the bucket — this prevents any accidental policy that could expose content directly
- **Bucket policy** restricts access to only the specific CloudFront distribution using `AWS:SourceArn` condition — even other CloudFront distributions cannot access this bucket
- **SSE-S3 encryption** protects data at rest with no added cost
- **Versioning** provides a safety net for accidental deletions
- **Access logging** to a separate bucket ensures there is always an audit trail even if the main bucket is compromised

---

## CI/CD Pipeline — Explanation and Usage

### Pipeline Location

```
case1/.github/workflows/deploy.yml
```

### How It Works

The pipeline triggers automatically on every push to the `main` branch and runs two jobs:

**Job 1 — build-and-push**
1. Checks out the latest code from GitHub
2. Authenticates to AWS using the stored IAM credentials
3. Authenticates Docker to Amazon ECR
4. Builds the Docker image from `./app/Dockerfile`
5. Pushes the image tagged as `:latest` to ECR

**Job 2 — deploy** (runs after Job 1 succeeds)
1. Authenticates to AWS
2. Calls `aws ecs update-service --force-new-deployment`
3. ECS pulls the new `:latest` image from ECR
4. Rolls out new containers, draining old ones gracefully via ALB
5. ALB health checks confirm new container is healthy before traffic switches

### GitHub Secrets Required

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `DOCKERHUB_USERNAME` | Docker Hub username (malavsh) |
| `DOCKERHUB_TOKEN` | Docker Hub personal access token |

**How to add secrets:**
1. Go to your GitHub repository
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret** for each entry above

### Triggering the Pipeline Manually

```bash
# Make any change and push to trigger pipeline
git add .
git commit -m "Trigger deployment"
git push origin main
```

Or trigger from GitHub: **Actions → Select workflow → Run workflow**

### Checking Pipeline Status

Go to **GitHub → Actions tab** — you will see:
- Green checkmark = successful build and deploy
- Red X = something failed — click to see exact error logs

---

## Project Folder Structure

```
DevOps_Assignment_Malav/
├── SUBMISSION.md                  ← This file
├── README.md                      ← Overall project documentation
│
├── case1/
│   ├── app/
│   │   ├── index.js               ← Express.js application
│   │   ├── package.json           ← Node dependencies
│   │   ├── Dockerfile             ← Container build instructions
│   │   └── node_modules           ← Node Modules
│   ├── .github/
│   │   └── workflows/
│   │       └── deploy.yml         ← CI/CD pipeline definition
│   ├── task-definition.json       ← ECS Fargate task config
│   ├── proofs/                    ← Screenshots of all AWS configs
│   ├── docs/
│   │   └── architecture.png       ← Architecture diagram
│   └── README_Case1.md            ← Case 1 setup guide
│
└── case2/
    ├── content/
    │   ├── index.html             ← Sample static page
    │   ├── style.css              ← Sample stylesheet
    │   └── logo.png               ← Sample image
    ├── proofs/                    ← Screenshots of S3 and CloudFront
    ├── docs/
    │   └── architecture.png       ← Architecture diagram
    └── README.md                  ← Case 2 setup guide
```

---

## Cleanup and Teardown Instructions

Run these commands in order after the assignment review is complete to avoid ongoing AWS charges.

### Case 1 Teardown

```bash
# 1. Scale down ECS service to zero tasks
aws ecs update-service --cluster my-cluster --service my-service --desired-count 0 --region ap-south-1

# 2. Delete ECS service
aws ecs delete-service --cluster my-cluster --service my-service --region ap-south-1

# 3. Delete ECS cluster
aws ecs delete-cluster --cluster my-cluster --region ap-south-1

# 4. Delete NAT Gateway (saves ~$32/month — do this first before releasing EIP)
aws ec2 delete-nat-gateway --nat-gateway-id NAT_GW_ID --region ap-south-1

# 5. Wait ~60 seconds, then release Elastic IP
aws ec2 release-address --allocation-id ALLOC_ID --region ap-south-1

# 6. Delete ECR repository and all images
aws ecr delete-repository --repository-name devops-case1 --force --region ap-south-1

# 7. Delete CloudWatch log group
aws logs delete-log-group --log-group-name /ecs/devops-case1 --region ap-south-1

# 8. Delete Load Balancer and Target Group
#    → AWS Console: EC2 → Load Balancers → Select → Actions → Delete
#    → AWS Console: EC2 → Target Groups → Select → Actions → Delete

# 9. Delete CloudFront distribution
#    → AWS Console: CloudFront → Select distribution → Disable → Wait → Delete

# 10. Delete VPC (this also removes subnets, route tables, IGW automatically)
#     → AWS Console: VPC → Your VPCs → Select devops-vpc → Actions → Delete
```

### Case 2 Teardown

```bash
# 1. Empty the content bucket
aws s3 rm s3://devops-case2-static-malav --recursive

# 2. Delete the content bucket
aws s3 rb s3://devops-case2-static-malav

# 3. Empty and delete the logs bucket
aws s3 rm s3://devops-case2-logs-malav --recursive
aws s3 rb s3://devops-case2-logs-malav

# 4. Disable and delete CloudFront distribution
#    → AWS Console: CloudFront → Select distribution → Disable → Wait ~5 min → Delete
```

---

## Known Limitations and Trade-offs

| Limitation | Note |
|---|---|
| Infrastructure not defined as IaC | Would use Terraform in production for reproducible deploys |
| No automated testing in CI/CD | Would add Jest unit tests and Postman/Newman integration tests |
| Single region deployment | Would deploy across multiple regions for global HA |
| No monitoring dashboard | Would configure CloudWatch dashboard and SNS alerts for production |
