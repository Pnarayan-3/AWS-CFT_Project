# Static Website Hosting Stack - Deployment Guide

## Prerequisites
- AWS CLI installed and configured with credentials
- Basic understanding of CloudFormation
- S3 and CloudFront permissions in your AWS account

---

## Step 1: Validate the Template

Before deploying, validate your CloudFormation template:

```bash
aws cloudformation validate-template \
  --template-body file://static-website-stack.yaml \
  --region us-east-1
```

---

## Step 2: Create the Stack

Deploy the stack with custom parameters:

```bash
aws cloudformation create-stack \
  --stack-name my-static-website-stack \
  --template-body file://static-website-stack.yaml \
  --parameters \
    ParameterKey=SiteDomain,ParameterValue=my-awesome-site \
    ParameterKey=Environment,ParameterValue=prod \
  --region us-east-1
```

**Or use the AWS Console:**
1. Go to CloudFormation → Create Stack
2. Upload the `static-website-stack.yaml` file
3. Fill in parameters:
   - **SiteDomain**: my-awesome-site
   - **Environment**: prod
4. Click Create Stack

---

## Step 3: Wait for Stack Creation

Monitor stack creation:

```bash
aws cloudformation describe-stacks \
  --stack-name my-static-website-stack \
  --query 'Stacks[0].StackStatus' \
  --region us-east-1
```

Wait for status: `CREATE_COMPLETE`

---

## Step 4: Get the Outputs

Retrieve stack outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name my-static-website-stack \
  --query 'Stacks[0].Outputs' \
  --region us-east-1
```

You'll get:
- **S3BucketName**: Your bucket name
- **CloudFrontDomainName**: Your website URL
- **CloudFrontDistributionId**: For cache invalidation

---

## Step 5: Create Sample Website Content

Create an `index.html` file:

```html
<!DOCTYPE html>
<html>
<head>
    <title>My Static Website</title>
</head>
<body>
    <h1>Welcome to My Website!</h1>
    <p>Hosted on S3 with CloudFront CDN</p>
</body>
</html>
```

Create a `404.html` error page:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Page Not Found</title>
</head>
<body>
    <h1>404 - Page Not Found</h1>
    <p><a href="/">Go back to home</a></p>
</body>
</html>
```

---

## Step 6: Upload Content to S3

Replace `my-awesome-site-prod-XXXXXX` with your actual bucket name:

```bash
# Upload a single file
aws s3 cp index.html s3://my-awesome-site-prod-XXXXXX/

# Upload error page
aws s3 cp 404.html s3://my-awesome-site-prod-XXXXXX/

# Upload entire directory
aws s3 cp . s3://my-awesome-site-prod-XXXXXX/ --recursive
```

---

## Step 7: Access Your Website

Get your CloudFront domain:

```bash
aws cloudformation describe-stacks \
  --stack-name my-static-website-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`CloudFrontDomainName`].OutputValue' \
  --region us-east-1
```

Visit: `https://d1234abcd.cloudfront.net` (your domain will be different)

---

## Step 8: Clear CloudFront Cache (After Updates)

When you update your website files, invalidate the cache:

```bash
DIST_ID=$(aws cloudformation describe-stacks \
  --stack-name my-static-website-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`CloudFrontDistributionId`].OutputValue' \
  --output text \
  --region us-east-1)

aws cloudfront create-invalidation \
  --distribution-id $DIST_ID \
  --paths "/*" \
  --region us-east-1
```

---

## Understanding the Template Components

### 1. **S3 Bucket**
- Stores your website files
- Blocks all public access (CloudFront handles access)
- Has versioning enabled for backup

### 2. **Origin Access Identity (OAI)**
- Allows CloudFront to access S3 privately
- Prevents direct S3 URL access

### 3. **Bucket Policy**
- Grants CloudFront OAI permission to read files
- Implements least-privilege access

### 4. **CloudFront Distribution**
- Caches content worldwide
- Different TTLs for different file types:
  - HTML: 60 seconds (fresh content)
  - Assets (CSS, JS): 1 year (never change)
  - Default: 5 minutes
- Redirects HTTP to HTTPS automatically
- Gzip compression enabled

### 5. **Error Handling**
- 404 errors show `/404.html`
- Helps with user experience

---

## Key Learning Points

✅ **Parameters**: Make templates reusable with different values  
✅ **Intrinsic Functions**: `!Ref`, `!GetAtt`, `!Sub` combine resources  
✅ **IAM Policies**: Implement least-privilege access  
✅ **Outputs**: Export values for easy reference  
✅ **Caching Strategy**: Different TTLs for different content types  

---

## Cleanup (Delete Stack)

When you're done, clean up to avoid charges:

```bash
aws cloudformation delete-stack \
  --stack-name my-static-website-stack \
  --region us-east-1
```

**Note**: The S3 bucket must be empty before deletion. Empty it first:

```bash
BUCKET=$(aws cloudformation describe-stacks \
  --stack-name my-static-website-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`S3BucketName`].OutputValue' \
  --output text \
  --region us-east-1)

aws s3 rm s3://$BUCKET --recursive
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Stack creation failed | Check CloudFormation Events tab for error messages |
| Files not uploading | Verify S3 bucket name and AWS CLI permissions |
| Website not accessible | Wait 5-10 minutes for CloudFront to propagate |
| Old content showing | Invalidate CloudFront cache (see Step 8) |
| 403 Forbidden errors | Ensure CloudFront OAI has access to S3 bucket |

---

Happy deploying! 🚀