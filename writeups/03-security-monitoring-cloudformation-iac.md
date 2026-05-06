# OmegaNimbus Day 3: Security Monitoring & Infrastructure as Code

**Author:** Albert Sicart  
**Domain:** omeganimbus.com  
**Stack:** GuardDuty · CloudTrail · Security Hub · CloudFormation · S3 · DynamoDB · Lambda · API Gateway · CloudFront · IAM  
**Date:** May 2026

---

## Overview

Building on Days 1 and 2 (S3, CloudFront, ACM, Lambda, SES, DynamoDB, CodePipeline), this session focused on two objectives:

1. **Security Monitoring** — activating GuardDuty, CloudTrail, and Security Hub to establish real threat detection and audit capabilities
2. **Infrastructure as Code** — defining the entire OmegaNimbus infrastructure as a single CloudFormation stack, migrating from manual console configuration to reproducible, version-controlled IaC

---

## Part 1 — Security Monitoring

### GuardDuty

GuardDuty is AWS's intelligent threat detection service. It continuously monitors for malicious activity and unauthorized behavior by analyzing data sources including VPC Flow Logs, DNS logs, and CloudTrail events — without requiring any agents or additional infrastructure.

**Activation:** Single-click from the GuardDuty console. No configuration required to start monitoring.

**What it detects:**
- Unusual API calls from unexpected geolocations
- Access from known malicious IPs
- Credential exfiltration attempts
- Cryptocurrency mining activity
- Anomalous data access patterns in S3 and DynamoDB

**Free tier:** 30-day trial. Post-trial cost is minimal for low-traffic personal accounts (cents/month).

**Key concept for SCS-C02:** GuardDuty operates at the account level and integrates natively with EventBridge for automated response workflows — for example, triggering a Lambda to isolate a compromised EC2 instance when a high-severity finding is generated.

---

### CloudTrail

CloudTrail provides a complete audit log of every API call made in the AWS account — who did what, when, and from which IP address. It is the foundational service for forensic investigation and compliance.

**Configuration:**

- **Trail name:** `omeganimbus-trail`
- **Storage:** New S3 bucket — `omeganimbus-cloudtrail-logs`
- **Encryption:** KMS key alias `omeganimbus-cloudtrail-key` (SSE-KMS)
- **Log events:** Management events — Read + Write
- **Multi-region:** Yes — captures events from all regions

**Why KMS encryption matters:** Encrypting CloudTrail logs with a customer-managed KMS key (CMK) adds a second layer of access control. Even if someone gains access to the S3 bucket, they cannot read the logs without also having permission to use the KMS key. This is a security best practice and a common exam topic in SCS-C02.

**Key concept:** CloudTrail records *control plane* events (API calls that change infrastructure). For *data plane* events (reads/writes to S3 objects or DynamoDB items), you need to explicitly enable Data Events — these generate significantly more volume and cost.

---

### Security Hub

Security Hub centralizes security findings from GuardDuty, Inspector, Macie, and other services into a single dashboard. It also evaluates the account against industry security standards and benchmarks.

**Activated standards:**
- AWS Foundational Security Best Practices
- CIS AWS Foundations Benchmark

**What it shows:** A prioritized list of security findings with severity levels (Critical, High, Medium, Low). Many findings are configuration issues in newly created accounts — for example, MFA not enabled on the root account, or S3 buckets with public access enabled. These are expected and provide concrete improvement tasks.

**Key concept for SCS-C02:** Security Hub acts as the aggregation layer in the AWS security ecosystem. The typical architecture is: GuardDuty/Inspector/Macie → Security Hub → EventBridge → Lambda/SNS for automated response.

---

## Part 2 — Infrastructure as Code (CloudFormation)

### What is Infrastructure as Code

CloudFormation is AWS's native IaC service. Instead of configuring resources manually through the console, you define the entire infrastructure in a YAML or JSON template. CloudFormation creates, updates, and deletes resources as a single unit called a **stack**.

**Why it matters:**
- **Reproducibility** — the same template deploys identical infrastructure in any account or region
- **Version control** — the template lives in GitHub alongside the application code
- **Auditability** — every change is tracked
- **Disaster recovery** — losing the entire AWS account is recoverable by re-running the template

This is what separates a Cloud Engineer from someone who knows how to click through the console.

---

### Architecture

The complete OmegaNimbus infrastructure defined in a single stack:

```
omeganimbus-stack
│
├── S3 Bucket (omeganimbus.com-cfn)
│   └── BucketPolicy (public read)
│
├── DynamoDB Table (omeganimbus-visitors-cfn)
│
├── IAM Roles
│   ├── ContactFormLambdaRole (ses:SendEmail on 2 specific identities)
│   └── VisitorCounterLambdaRole (dynamodb:UpdateItem, dynamodb:GetItem on 1 table)
│
├── Lambda Functions
│   ├── omeganimbus-visitor-counter-cfn (Python 3.12, arm64)
│   └── omeganimbus-contact-form-cfn (Python 3.12, arm64)
│
├── API Gateway (HTTP API)
│   ├── GET /visitors → VisitorCounterFunction
│   └── POST /contact → ContactFormFunction
│
└── CloudFront Distribution
    ├── Origin: S3 website endpoint
    ├── Aliases: omeganimbus.com, www.omeganimbus.com
    ├── Certificate: ACM (us-east-1)
    └── ViewerProtocolPolicy: redirect-to-https
```

**Resources:** 15 total  
**Outputs:** 10 (URLs, ARNs, IDs)

---

### CloudFormation Template

The complete template. Key design decisions explained inline.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: OmegaNimbus - Cloud Security Portfolio Infrastructure

Parameters:
  DomainName:
    Type: String
    Default: omeganimbus.com
    Description: The domain name for the website

Resources:

  # ─── S3 BUCKET ───
  WebsiteBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${DomainName}-cfn'
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: index.html
      PublicAccessBlockConfiguration:
        BlockPublicAcls: false
        BlockPublicPolicy: false
        IgnorePublicAcls: false
        RestrictPublicBuckets: false
      Tags:
        - Key: Project
          Value: OmegaNimbus
        - Key: Environment
          Value: Production

  WebsiteBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref WebsiteBucket
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: PublicReadGetObject
            Effect: Allow
            Principal: '*'
            Action: s3:GetObject
            Resource: !Sub 'arn:aws:s3:::${WebsiteBucket}/*'

  # ─── DYNAMODB VISITOR COUNTER ───
  VisitorCounterTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: omeganimbus-visitors-cfn
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      Tags:
        - Key: Project
          Value: OmegaNimbus

  # ─── IAM ROLE FOR LAMBDA (CONTACT FORM) ───
  ContactFormLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: omeganimbus-contact-form-role-cfn
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: omeganimbus-ses-send-only
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Sid: AllowSESSendEmailOnly
                Effect: Allow
                Action: ses:SendEmail
                Resource:
                  - !Sub 'arn:aws:ses:${AWS::Region}:${AWS::AccountId}:identity/omeganimbus.com'
                  - !Sub 'arn:aws:ses:${AWS::Region}:${AWS::AccountId}:identity/sicart@protonmail.ch'

  # ─── IAM ROLE FOR LAMBDA (VISITOR COUNTER) ───
  VisitorCounterLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: omeganimbus-visitor-counter-role-cfn
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: omeganimbus-dynamodb-visitors-only
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Sid: AllowDynamoDBVisitorCounter
                Effect: Allow
                Action:
                  - dynamodb:UpdateItem
                  - dynamodb:GetItem
                Resource: !GetAtt VisitorCounterTable.Arn

  # ─── LAMBDA - VISITOR COUNTER ───
  VisitorCounterFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: omeganimbus-visitor-counter-cfn
      Runtime: python3.12
      Architectures:
        - arm64
      Handler: index.lambda_handler
      Role: !GetAtt VisitorCounterLambdaRole.Arn
      Environment:
        Variables:
          TABLE_NAME: !Ref VisitorCounterTable
      Code:
        ZipFile: |
          import json
          import boto3
          import os

          def lambda_handler(event, context):
              dynamodb = boto3.resource('dynamodb', region_name='eu-north-1')
              table = dynamodb.Table(os.environ['TABLE_NAME'])

              if event.get('requestContext', {}).get('http', {}).get('method') == 'OPTIONS':
                  return {
                      'statusCode': 200,
                      'headers': {
                          'Access-Control-Allow-Origin': 'https://omeganimbus.com',
                          'Access-Control-Allow-Headers': 'Content-Type',
                          'Access-Control-Allow-Methods': 'GET, OPTIONS'
                      },
                      'body': ''
                  }

              response = table.update_item(
                  Key={'id': 'total'},
                  UpdateExpression='ADD visits :inc',
                  ExpressionAttributeValues={':inc': 1},
                  ReturnValues='UPDATED_NEW'
              )

              count = int(response['Attributes']['visits'])

              return {
                  'statusCode': 200,
                  'headers': {
                      'Access-Control-Allow-Origin': 'https://omeganimbus.com',
                      'Access-Control-Allow-Headers': 'Content-Type',
                      'Access-Control-Allow-Methods': 'GET, OPTIONS'
                  },
                  'body': json.dumps({'visits': count})
              }

  # ─── LAMBDA - CONTACT FORM ───
  ContactFormFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: omeganimbus-contact-form-cfn
      Runtime: python3.12
      Architectures:
        - arm64
      Handler: index.lambda_handler
      Role: !GetAtt ContactFormLambdaRole.Arn
      Environment:
        Variables:
          SENDER_EMAIL: contact@omeganimbus.com
          RECIPIENT_EMAIL: sicart@protonmail.ch
      Code:
        ZipFile: |
          import json
          import boto3
          import os

          def lambda_handler(event, context):
              if event.get('requestContext', {}).get('http', {}).get('method') == 'OPTIONS':
                  return {
                      'statusCode': 200,
                      'headers': {
                          'Access-Control-Allow-Origin': 'https://omeganimbus.com',
                          'Access-Control-Allow-Headers': 'Content-Type',
                          'Access-Control-Allow-Methods': 'POST, OPTIONS'
                      },
                      'body': ''
                  }

              try:
                  body = json.loads(event.get('body', '{}'))
                  name = body.get('name', 'Unknown')
                  email = body.get('email', 'Unknown')
                  message = body.get('message', '')

                  ses = boto3.client('ses', region_name='eu-north-1')
                  ses.send_email(
                      Source=os.environ['SENDER_EMAIL'],
                      Destination={'ToAddresses': [os.environ['RECIPIENT_EMAIL']]},
                      Message={
                          'Subject': {'Data': f'OmegaNimbus contact: {name}'},
                          'Body': {'Text': {'Data': f'From: {name} <{email}>\n\n{message}'}}
                      }
                  )

                  return {
                      'statusCode': 200,
                      'headers': {
                          'Access-Control-Allow-Origin': 'https://omeganimbus.com',
                          'Access-Control-Allow-Headers': 'Content-Type',
                          'Access-Control-Allow-Methods': 'POST, OPTIONS'
                      },
                      'body': json.dumps({'message': 'Email sent successfully'})
                  }

              except Exception as e:
                  return {
                      'statusCode': 500,
                      'headers': {
                          'Access-Control-Allow-Origin': 'https://omeganimbus.com',
                      },
                      'body': json.dumps({'error': str(e)})
                  }

  # ─── CLOUDFRONT ───
  CloudFrontDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Enabled: true
        Comment: OmegaNimbus portfolio CDN
        DefaultRootObject: index.html
        Aliases:
          - omeganimbus.com
          - www.omeganimbus.com
        ViewerCertificate:
          AcmCertificateArn: arn:aws:acm:us-east-1:247906201225:certificate/3c485857-34d5-487e-9641-63cc78834df7
          SslSupportMethod: sni-only
          MinimumProtocolVersion: TLSv1.2_2021
        Origins:
          - Id: S3Origin
            DomainName: !Sub '${WebsiteBucket}.s3-website.${AWS::Region}.amazonaws.com'
            CustomOriginConfig:
              HTTPPort: 80
              OriginProtocolPolicy: http-only
        DefaultCacheBehavior:
          TargetOriginId: S3Origin
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
          AllowedMethods:
            - GET
            - HEAD
          CachedMethods:
            - GET
            - HEAD
        HttpVersion: http2
        PriceClass: PriceClass_100

  # ─── API GATEWAY ───
  OmegaNimbusApi:
    Type: AWS::ApiGatewayV2::Api
    Properties:
      Name: omeganimbus-api-cfn
      ProtocolType: HTTP
      CorsConfiguration:
        AllowOrigins:
          - https://omeganimbus.com
        AllowMethods:
          - GET
          - POST
          - OPTIONS
        AllowHeaders:
          - Content-Type

  VisitorCounterInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt VisitorCounterFunction.Arn
      Action: lambda:InvokeFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${OmegaNimbusApi}/*'

  ContactFormInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt ContactFormFunction.Arn
      Action: lambda:InvokeFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${OmegaNimbusApi}/*'

  VisitorCounterIntegration:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId: !Ref OmegaNimbusApi
      IntegrationType: AWS_PROXY
      IntegrationUri: !GetAtt VisitorCounterFunction.Arn
      PayloadFormatVersion: '2.0'

  ContactFormIntegration:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId: !Ref OmegaNimbusApi
      IntegrationType: AWS_PROXY
      IntegrationUri: !GetAtt ContactFormFunction.Arn
      PayloadFormatVersion: '2.0'

  VisitorCounterRoute:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId: !Ref OmegaNimbusApi
      RouteKey: GET /visitors
      Target: !Sub 'integrations/${VisitorCounterIntegration}'

  ContactFormRoute:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId: !Ref OmegaNimbusApi
      RouteKey: POST /contact
      Target: !Sub 'integrations/${ContactFormIntegration}'

  ApiStage:
    Type: AWS::ApiGatewayV2::Stage
    Properties:
      ApiId: !Ref OmegaNimbusApi
      StageName: $default
      AutoDeploy: true

Outputs:
  WebsiteBucketName:
    Value: !Ref WebsiteBucket
  WebsiteURL:
    Value: !GetAtt WebsiteBucket.WebsiteURL
  DynamoDBTableName:
    Value: !Ref VisitorCounterTable
  ApiEndpoint:
    Value: !Sub 'https://${OmegaNimbusApi}.execute-api.${AWS::Region}.amazonaws.com'
  CloudFrontDomain:
    Value: !GetAtt CloudFrontDistribution.DomainName
  CloudFrontDistributionId:
    Value: !Ref CloudFrontDistribution
```

---

### Key CloudFormation Concepts

**`!Ref`** — returns the logical name or primary identifier of a resource. `!Ref WebsiteBucket` returns the bucket name.

**`!GetAtt`** — returns a specific attribute of a resource. `!GetAtt VisitorCounterTable.Arn` returns the full ARN of the DynamoDB table.

**`!Sub`** — substitutes variables in a string. `!Sub 'arn:aws:s3:::${WebsiteBucket}/*'` dynamically builds the ARN using the bucket name. Also supports pseudo-parameters like `${AWS::Region}` and `${AWS::AccountId}`.

**Intrinsic functions are the key to IaC.** Instead of hardcoding ARNs, names, and IDs, resources reference each other dynamically. If a resource name changes, all dependent resources update automatically.

---

### Debugging Notes

**Error 1 — S3 Block Public Access**
```
s3:PutBucketPolicy blocked by BlockPublicPolicy setting
```
Root cause: S3 buckets created in 2023+ have Block Public Access enabled by default at the bucket level. The template must explicitly set all four `PublicAccessBlockConfiguration` flags to `false` before the bucket policy can be applied.

**Error 2 — CloudFront alias conflict (first attempt)**
```
One or more aliases specified for the distribution includes an incorrectly 
configured DNS record that points to another CloudFront distribution.
```
Root cause: AWS validates that the DNS record for an alias points to *this* distribution before accepting it. The original distribution still had `omeganimbus.com` and `www.omeganimbus.com` as aliases. Solution: remove aliases from the original distribution first, then add them to the new one.

**Error 3 — CloudFront alias conflict (second attempt)**
```
One or more of the CNAMEs you provided are already associated with a 
different resource.
```
Root cause: AWS maintains an internal registry of alias-to-distribution mappings independently of DNS. Even after updating DNS in DonDominio, the original distribution still had the aliases registered in AWS's internal system. Solution: edit the original distribution in the CloudFront console and remove the aliases before attempting the update.

**Error 4 — CodePipeline S3 permissions**
```
Not authorized to perform s3:PutObject on omeganimbus.com-cfn
```
Root cause: The CodePipeline service role had S3 permissions scoped to the original bucket (`omeganimbus.com`). Migrating the pipeline to the new bucket required adding `omeganimbus.com-cfn` and `omeganimbus.com-cfn/*` to the `CodePipeline-S3Deploy` policy resource list.

---

### Migration Process

Migrating from manually-created infrastructure to CloudFormation-managed infrastructure required careful sequencing:

1. Deploy stack with S3, DynamoDB, IAM (no conflicts with existing resources)
2. Add Lambda functions via stack update
3. Add API Gateway via stack update
4. Add CloudFront **without aliases** (to avoid conflict with original distribution)
5. Verify new distribution works via `.cloudfront.net` URL
6. Update `index.html` with new API Gateway endpoint URLs
7. Remove aliases from original CloudFront distribution
8. Update stack to add aliases and ACM certificate to new distribution
9. Update DNS in DonDominio to point to new distribution
10. Migrate CodePipeline to new S3 bucket
11. Disable original CloudFront distribution

This incremental approach — building the stack piece by piece while keeping production running — is how real infrastructure migrations are done.

---

### Cost Summary

All services remain within the AWS Free Tier:

| Service | Free Tier | Estimated cost |
|---|---|---|
| S3 | 5 GB storage, 20k GET requests | ~$0 |
| CloudFront | 1 TB transfer, 10M requests | ~$0 |
| Lambda | 1M requests/month | ~$0 |
| API Gateway | 1M HTTP calls/month | ~$0 |
| DynamoDB | 25 GB storage, 25 WCU/RCU | ~$0 |
| CloudTrail | First trail free | ~$0 |
| GuardDuty | 30-day trial | ~$0 |
| Security Hub | 30-day trial | ~$0 |
| KMS | 20k free requests/month | ~$0 |

**Total: ~$0/month**

---

### Key Concepts Practiced

- CloudFormation stack lifecycle (create, update, rollback, delete)
- Intrinsic functions: `!Ref`, `!GetAtt`, `!Sub`
- IAM roles defined as code with inline policies
- Lambda code inline in CloudFormation (`ZipFile`)
- API Gateway HTTP API with Lambda proxy integration
- CloudFront distribution with ACM certificate and custom aliases
- Incremental IaC migration without production downtime
- GuardDuty threat detection activation
- CloudTrail audit logging with KMS encryption
- Security Hub findings aggregation and benchmark evaluation

---

### What's Next

| Feature | AWS Services | Certification relevance |
|---|---|---|
| GuardDuty findings dashboard | Lambda + API Gateway + EventBridge | SCS-C02 |
| Automated security alerts | EventBridge + SNS | SCS-C02 |
| Study Notes section | S3 + CloudFront | Portfolio |
| RDS + ElastiCache | Relational database layer | SAA-C03 |
| WAF integration | AWS WAF + CloudFront | SCS-C02 |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*
