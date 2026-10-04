# Static Website Hosting with AWS CloudFormation

A complete Infrastructure as Code solution for hosting static websites using AWS CloudFormation, S3, and CloudFront.

---

## 📁 Project Structure

```
.
├── static_website_stack.yaml      # CloudFormation template (main file)
├── DEPLOYMENT_GUIDE.md            # Step-by-step deployment instructions
├── index.html                      # Sample homepage
├── 404.html                        # Sample error page
└── README.md                       # This file
```

---

## 🎯 Project Goals

✅ Learn CloudFormation basics  
✅ Understand S3 bucket configuration  
✅ Set up CloudFront CDN distribution  
✅ Implement IAM security policies  
✅ Master intrinsic functions and outputs  

---

## 🏗️ Architecture Overview

```
┌─────────────────────┐
│   CloudFront CDN    │
│  (Global Caching)   │
└──────────┬──────────┘
           │ (HTTPS)
           │
    ┌──────┴───────┐
    │ S3 Bucket    │
    │ (Static      │
    │  Files)      │
    └──────────────┘

User → CloudFront Edge → S3 Origin
```

**Flow:**
1. User requests website via CloudFront domain
2. CloudFront checks if content is cached
3. If not cached, retrieves from S3 bucket
4. Content is cached at edge locations
5. Future requests are served from cache

---

## 🔑 Key CloudFormation Concepts

### 1. **Template Structure**
```yaml
AWSTemplateFormatVersion: '2010-09-09'     # Version (always use this)
Description: 'Template description'        # Human-readable description
Parameters:                                # Input values
  SiteDomain:                             # Parameter name
    Type: String                          # Data type
    Default: 'my-site'                    # Default value
Resources:                                # AWS resources to create
  MyBucket:                               # Logical ID
    Type: AWS::S3::Bucket                 # Resource type
    Properties:                           # Configuration
      BucketName: my-bucket               # Property
Outputs:                                  # Export values
  BucketArn:
    Value: !GetAtt MyBucket.Arn          # Reference value
```

### 2. **Intrinsic Functions Used**

| Function | Purpose | Example |
|----------|---------|---------|
| `!Ref` | Reference resource property | `!Ref StaticWebsiteBucket` |
| `!GetAtt` | Get resource attribute | `!GetAtt MyBucket.Arn` |
| `!Sub` | String substitution | `!Sub '${SiteDomain}-bucket'` |
| `!GetAZs` | List availability zones | `!GetAZs ''` |

### 3. **Parameters**
Parameters allow template reusability:
```yaml
Parameters:
  Environment:
    Type: String
    AllowedValues:
      - dev
      - staging
      - prod
```

Then reference: `!Ref Environment`

### 4. **Outputs**
Export useful values for reference:
```yaml
Outputs:
  BucketName:
    Value: !Ref StaticWebsiteBucket
    Export:
      Name: !Sub '${AWS::StackName}-BucketName'
```

---

## 🔐 Security Features

### 1. **S3 Bucket Policy**
- Allows CloudFront OAI to read files
- Denies all other public access
- Implements least-privilege principle

### 2. **Public Access Block**
```yaml
PublicAccessBlockConfiguration:
  BlockPublicAcls: true
  BlockPublicPolicy: true
  IgnorePublicAcls: true
  RestrictPublicBuckets: true
```
This prevents accidental public exposure.

### 3. **Origin Access Identity (OAI)**
- CloudFront identity to access S3
- Users cannot access S3 directly
- Private URL required CloudFront

### 4. **HTTPS Enforcement**
```yaml
ViewerProtocolPolicy: redirect-to-https
```
All HTTP requests redirect to HTTPS.

---

## ⚡ Caching Strategy

Different file types have different cache durations:

| File Type | TTL | Reason |
|-----------|-----|--------|
| HTML | 60s | Content changes frequently |
| CSS/JS | 1 year | Assets rarely change |
| Default | 5min | Balance between fresh and cached |

**How it works:**
1. First request: Fetched from S3 (origin)
2. Cached at edge location for TTL duration
3. Subsequent requests: Served from cache
4. After TTL expires: Fresh copy fetched from S3

---

## 📋 What Each Resource Does

### **S3 Bucket**
Stores static website files.
- Versioning: Enabled for backup
- Public Access Block: All blocked
- Bucket Policy: Allows CloudFront OAI only

### **Origin Access Identity (OAI)**
Allows CloudFront to access S3.
```
CloudFront OAI = Special AWS identity
S3 Bucket Policy = Grants OAI permission
```

### **S3 Bucket Policy**
IAM policy attached to bucket:
- `s3:GetObject`: Read file permission
- `s3:ListBucket`: List contents permission
- Principal: CloudFront OAI

### **CloudFront Distribution**
CDN that caches and delivers content.
- Origin: Points to S3 bucket
- Default Behavior: How to handle requests
- Cache Behaviors: Special rules for file types
- Error Responses: Custom error pages

---

## 🚀 Quick Start

### 1. Download Files
Save these files locally:
- `static-website-stack.yaml` (template)
- `index.html` (sample content)
- `404.html` (error page)

### 2. Create Stack
```bash
aws cloudformation create-stack \
  --stack-name my-website \
  --template-body file://static-website-stack.yaml \
  --parameters ParameterKey=SiteDomain,ParameterValue=my-site \
  --region us-east-1
```

### 3. Get Outputs
```bash
aws cloudformation describe-stacks \
  --stack-name my-website \
  --query 'Stacks[0].Outputs'
```

### 4. Upload Content
```bash
BUCKET=your-bucket-name
aws s3 cp index.html s3://$BUCKET/
aws s3 cp 404.html s3://$BUCKET/
```

### 5. Visit Your Site
Open CloudFront URL in browser!

---

## 📚 Important Concepts Explained

### **Logical ID vs Physical ID**
```yaml
Resources:
  StaticWebsiteBucket:              # ← Logical ID (used in template)
    Type: AWS::S3::Bucket
```
Physical ID: `my-awesome-site-prod-123456789`

### **Properties vs Attributes**
```yaml
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:                     # ← Set on creation
      BucketName: my-bucket
      VersioningConfiguration: ...
```
Attributes: `Arn`, `DomainName` (retrieved with `!GetAtt`)

### **Pseudo Parameters**
CloudFormation provides built-in values:
```yaml
!Sub '${AWS::StackName}'            # Stack name
!Sub '${AWS::AccountId}'            # AWS account ID
!Sub '${AWS::Region}'               # AWS region
```

### **Tags**
Organize and identify resources:
```yaml
Tags:
  - Key: Name
    Value: !Sub '${SiteDomain}-bucket'
  - Key: Environment
    Value: !Ref Environment
```

---

## 🔧 Customization

### Change Cache Duration
Edit these sections:
```yaml
DefaultCacheBehavior:
  DefaultTTL: 300        # Change this (seconds)
  MaxTTL: 1200           # Max cache time
```

### Add Custom Domain
Update CloudFront Aliases:
```yaml
CloudFrontDistribution:
  DistributionConfig:
    Aliases:
      - example.com
      - www.example.com
```

### Add Logging
Enable S3 access logs:
```yaml
LoggingConfiguration:
  IncludeCookies: false
  Bucket: my-logs-bucket.s3.amazonaws.com
```

---

## 🐛 Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| 403 Forbidden | CloudFront can't access S3 | Check OAI policy |
| 404 on root | No index.html | Upload index.html to S3 |
| Old content showing | Cache not invalidated | Run invalidation command |
| High costs | No caching | Increase TTL values |

**Run invalidation after updates:**
```bash
aws cloudfront create-invalidation \
  --distribution-id D1234ABCD \
  --paths "/*"
```

---

## 📖 Additional Resources

- **AWS CloudFormation Docs**: https://docs.aws.amazon.com/cloudformation/
- **S3 Best Practices**: https://docs.aws.amazon.com/s3
- **CloudFront Caching**: https://docs.aws.amazon.com/cloudfront
- **IAM Policies**: https://docs.aws.amazon.com/iam

---

Happy Learning! 🚀
