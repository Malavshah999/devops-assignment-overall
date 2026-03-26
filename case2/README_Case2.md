# Case 2 — Secure Public Bucket Storage

## Overview

Static content (HTML, CSS, images) is hosted in a private Amazon S3 bucket. The bucket has all public access blocked. Content is served exclusively through a CloudFront CDN using Origin Access Control (OAC), which means:

- Direct S3 URL → **403 AccessDenied** (blocked)
- CloudFront URL → **200 OK** (content served)

---

## Architecture

```
Internet / User
      |
      v
CloudFront Distribution  (Global Edge — HTTPS, caching, OAC signing)
      |
      | (SigV4 signed request via OAC)
      v
S3 Bucket  (Private — all public access blocked)
```
---

## Prerequisites

- **AWS CLI** v2 configured with admin permissions
- **AWS Account** with S3FullAccess and CloudFrontFullAccess

---

## Sample Content Structure

```
case2/content/
├── index.html      — Main landing page
├── style.css       — Stylesheet
└── logo.png        — Sample image asset
```

---

## Step-by-Step Deployment

### Step 1 — Create the S3 Bucket

```bash
aws s3 mb s3://devops-case2-static-ms --region ap-south-1
```

### Step 2 — Block All Public Access

```bash
aws s3api put-public-access-block \
  --bucket devops-case2-static-ms \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true
```

### Step 3 — Enable Server-Side Encryption

```bash
aws s3api put-bucket-encryption \
  --bucket devops-case2-static-ms \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

### Step 4 — Enable Versioning

```bash
aws s3api put-bucket-versioning \
  --bucket devops-case2-static-ms \
  --versioning-configuration Status=Enabled

```

### Step 5 — Upload Sample Content

```bash
aws s3 sync case2/content/ s3://devops-case2-static-ms/ --region ap-south-1
```

Verify upload:
```bash
aws s3 ls s3://devops-case2-static-ms/
```

### Step 6 — Create CloudFront Distribution with OAC

#### 6a — Create OAC via AWS Console

1. Go to **CloudFront → Origin access → Create control setting**
2. Name: `devops-oac`
3. Origin type: **S3**
4. Signing behavior: **Sign requests (recommended)**
5. Signing protocol: **SigV4**
6. Click **Create**
7. Copy the **OAC ID** — you need it in the next step

#### 6b — Create CloudFront Distribution

1. Go to **CloudFront → Create distribution**
2. **Origin domain:** Select your S3 bucket from the dropdown (`devops-case2-static-ms.s3.ap-south-1.amazonaws.com`)
3. **Origin access:** Select **Origin access control settings (recommended)**
4. Select the OAC you just created (`devops-oac`)
5. CloudFront will show a banner — click **Copy policy** (you will paste this into S3 next)
6. **Default root object:** `index.html`
7. **Viewer protocol policy:** Redirect HTTP to HTTPS
8. **Cache policy:** CachingOptimized (pre-built — best for static content)
9. **Price class:** Use all edge locations (or PriceClass_100 to save cost)
10. Click **Create distribution**
11. Copy your **CloudFront domain name** — looks like `XXXXX.cloudfront.net`

### Step 7 — Apply Bucket Policy (Allow Only CloudFront)

After creating the distribution, paste the policy CloudFront gave you. It looks like this — replace the placeholders:

```bash
aws s3api put-bucket-policy \
  --bucket devops-case2-static-ms \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowCloudFrontServicePrincipal",
        "Effect": "Allow",
        "Principal": {
          "Service": "cloudfront.amazonaws.com"
        },
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::devops-case2-static-ms/*",
        "Condition": {
          "StringEquals": {
            "AWS:SourceArn": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_DISTRIBUTION_ID"
          }
        }
      }
    ]
  }'
```

> Replace `YOUR_ACCOUNT_ID` and `YOUR_DISTRIBUTION_ID` with your actual values.

---

## Cache Invalidation

When you update content in S3, CloudFront may serve cached old content. Invalidate the cache:

```bash
# Invalidate all files
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"

# Invalidate specific file
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/index.html"
```
---

## Security Controls Implemented

| Control | Configuration | Benefit |
|---|---|---|
| Public Access Block | All 4 settings enabled | No direct S3 URL access |
| OAC Bucket Policy | Only CloudFront service principal | Cannot bypass via direct URL |
| Encryption at Rest | SSE-S3 AES-256 | Data protected on disk |
| Versioning | Enabled | Recovery from accidental deletion |
| Access Logging | Logs to separate bucket | Full audit trail |
| HTTPS Only | CloudFront redirects HTTP → HTTPS | Encrypted in transit |

---

## Cleanup Instructions

```bash
# 1. Empty the bucket first (required before deletion)
aws s3 rm s3://devops-case2-static-ms --recursive

# 2. Delete the bucket
aws s3 rb s3://devops-case2-static-ms

# 3. Disable CloudFront distribution (via Console — must disable before deleting)
# CloudFront → Distributions → Select → Disable → Wait → Delete
```
