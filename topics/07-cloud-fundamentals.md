# 7. Cloud Fundamentals (AWS)

## Table of Contents
- [Cloud Concepts](#cloud-concepts)
- [AWS Account Setup](#aws-account-setup)
- [IAM - Identity & Access Management](#iam---identity--access-management)
- [EC2 - Virtual Servers](#ec2---virtual-servers)
- [S3 - Object Storage](#s3---object-storage)
- [VPC - Networking](#vpc---networking)
- [RDS - Managed Database](#rds---managed-database)
- [Route 53 & ELB](#route-53--elb)
- [AWS CLI](#aws-cli)
- [Serverless - Lambda](#serverless---lambda)
- [Projects](#projects)

---

## Cloud Concepts

```
Cloud Service Models:
┌──────────────────────────────────────────────────────────┐
│ Traditional    IaaS           PaaS           SaaS        │
│ (On-Premise)                                              │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│ │Application│  │Application│  │Application│  │Application│ │
│ │Data      │  │Data      │  │Data      │  │Data      │ │
│ │Runtime   │  │Runtime   │  │Runtime   │  │Runtime   │ │
│ │Middleware │  │Middleware │  │Middleware │  │Middleware │ │
│ │OS        │  │OS        │  │OS        │  │OS        │ │
│ │Virtualiz.│  │Virtualiz.│  │Virtualiz.│  │Virtualiz.│ │
│ │Servers   │  │Servers   │  │Servers   │  │Servers   │ │
│ │Storage   │  │Storage   │  │Storage   │  │Storage   │ │
│ │Network   │  │Network   │  │Network   │  │Network   │ │
│ └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
│  You manage    You manage    You manage    You manage    │
│  EVERYTHING    App+Data+     App+Data      NOTHING       │
│                Runtime                      (just use it) │
│                                                           │
│ Examples:      EC2, VMs      Elastic       Gmail,         │
│ Own servers    DigitalOcean  Beanstalk     Slack,         │
│                              Heroku        Office 365     │
└──────────────────────────────────────────────────────────┘

FaaS (Function as a Service):
- AWS Lambda, Azure Functions, Google Cloud Functions
- You only write the function code
- Pay only when function runs

AWS Global Infrastructure:
- Regions: Geographic areas (us-east-1, ap-south-1, eu-west-1)
- Availability Zones: Data centers within a region (us-east-1a, us-east-1b)
- Edge Locations: CDN endpoints for CloudFront (300+ worldwide)
```

---

## AWS Account Setup

```bash
# 1. Go to aws.amazon.com → Create Free Tier account
# 2. Enable MFA on root account (Security → MFA)
# 3. Create an IAM admin user (never use root for daily work)

# Install AWS CLI
# macOS
brew install awscli

# Ubuntu
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
# Output: aws-cli/2.15.0

# Configure credentials
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json

# Verify
aws sts get-caller-identity
```

---

## IAM - Identity & Access Management

```bash
# Create a user
aws iam create-user --user-name devops-student

# Create access keys for CLI
aws iam create-access-key --user-name devops-student

# Attach a policy (admin access - for learning only)
aws iam attach-user-policy \
  --user-name devops-student \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Create a group
aws iam create-group --group-name developers
aws iam add-user-to-group --group-name developers --user-name devops-student

# List users
aws iam list-users
```

### IAM Policy Example (Custom)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowEC2ReadOnly",
            "Effect": "Allow",
            "Action": [
                "ec2:Describe*",
                "ec2:List*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowS3BucketAccess",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-devops-bucket",
                "arn:aws:s3:::my-devops-bucket/*"
            ]
        }
    ]
}
```

```bash
# Create custom policy
aws iam create-policy \
  --policy-name DevOpsStudentPolicy \
  --policy-document file://policy.json
```

---

## EC2 - Virtual Servers

### Launch an Instance (CLI)

```bash
# Find the latest Ubuntu AMI
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text

# Create a key pair
aws ec2 create-key-pair \
  --key-name devops-key \
  --query 'KeyMaterial' \
  --output text > devops-key.pem

chmod 400 devops-key.pem

# Create a security group
aws ec2 create-security-group \
  --group-name devops-sg \
  --description "DevOps security group"

# Allow SSH and HTTP
SG_ID=$(aws ec2 describe-security-groups --group-names devops-sg --query 'SecurityGroups[0].GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 443 --cidr 0.0.0.0/0

# Launch EC2 instance
aws ec2 run-instances \
  --image-id ami-0c7217cdde317cfec \
  --instance-type t2.micro \
  --key-name devops-key \
  --security-groups devops-sg \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-server}]'

# Get the public IP
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-server" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text

# SSH into the instance
ssh -i devops-key.pem ubuntu@<PUBLIC_IP>
```

### User Data (Auto-setup on launch)

```bash
#!/bin/bash
# File: userdata.sh - Runs automatically when EC2 starts

# Update system
apt-get update -y
apt-get upgrade -y

# Install Nginx
apt-get install -y nginx

# Create custom page
cat > /var/www/html/index.html << 'HTML'
<!DOCTYPE html>
<html>
<head><title>DevOps Server</title></head>
<body>
  <h1>Hello from AWS EC2!</h1>
  <p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>
  <p>Region: $(curl -s http://169.254.169.254/latest/meta-data/placement/region)</p>
</body>
</html>
HTML

# Start Nginx
systemctl start nginx
systemctl enable nginx
```

```bash
# Launch with user data
aws ec2 run-instances \
  --image-id ami-0c7217cdde317cfec \
  --instance-type t2.micro \
  --key-name devops-key \
  --security-groups devops-sg \
  --user-data file://userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-server}]'
```

### Manage Instances

```bash
# List all instances
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Stop an instance
aws ec2 stop-instances --instance-ids i-0abc123def456

# Start an instance
aws ec2 start-instances --instance-ids i-0abc123def456

# Terminate (delete) an instance
aws ec2 terminate-instances --instance-ids i-0abc123def456
```

---

## S3 - Object Storage

```bash
# Create a bucket
aws s3 mb s3://my-devops-bucket-2024

# Upload a file
aws s3 cp index.html s3://my-devops-bucket-2024/
aws s3 cp ./backups/ s3://my-devops-bucket-2024/backups/ --recursive

# Download a file
aws s3 cp s3://my-devops-bucket-2024/index.html ./downloaded.html

# List bucket contents
aws s3 ls s3://my-devops-bucket-2024/
aws s3 ls s3://my-devops-bucket-2024/ --recursive --human-readable

# Sync a directory (like rsync)
aws s3 sync ./website/ s3://my-devops-bucket-2024/website/
aws s3 sync s3://my-devops-bucket-2024/website/ ./local-copy/

# Delete a file
aws s3 rm s3://my-devops-bucket-2024/index.html

# Delete a bucket (must be empty first)
aws s3 rb s3://my-devops-bucket-2024 --force

# Static website hosting
aws s3 website s3://my-devops-bucket-2024/ \
  --index-document index.html \
  --error-document error.html
```

### S3 Bucket Policy (Public Read)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-devops-bucket-2024/*"
        }
    ]
}
```

---

## VPC - Networking

```
VPC Architecture:
┌─────────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16                                        │
│                                                          │
│  ┌─────────────────────┐  ┌─────────────────────┐      │
│  │ Public Subnet        │  │ Public Subnet        │      │
│  │ 10.0.1.0/24          │  │ 10.0.2.0/24          │      │
│  │ AZ: us-east-1a       │  │ AZ: us-east-1b       │      │
│  │                      │  │                      │      │
│  │ ┌──────┐ ┌──────┐   │  │ ┌──────┐             │      │
│  │ │Web-01│ │NAT GW│   │  │ │Web-02│             │      │
│  │ └──────┘ └──────┘   │  │ └──────┘             │      │
│  └──────────┬──────────┘  └──────────────────────┘      │
│             │ Internet Gateway                           │
│  ┌──────────┴──────────┐  ┌─────────────────────┐      │
│  │ Private Subnet       │  │ Private Subnet       │      │
│  │ 10.0.3.0/24          │  │ 10.0.4.0/24          │      │
│  │ AZ: us-east-1a       │  │ AZ: us-east-1b       │      │
│  │                      │  │                      │      │
│  │ ┌──────┐ ┌──────┐   │  │ ┌──────┐             │      │
│  │ │DB-01 │ │App-01│   │  │ │DB-02 │             │      │
│  │ └──────┘ └──────┘   │  │ └──────┘             │      │
│  └─────────────────────┘  └─────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

### Create VPC with CLI

```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value=devops-vpc

# Create public subnet
PUB_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' --output text)
aws ec2 create-tags --resources $PUB_SUBNET --tags Key=Name,Value=public-subnet

# Create private subnet
PRIV_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.3.0/24 \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' --output text)
aws ec2 create-tags --resources $PRIV_SUBNET --tags Key=Name,Value=private-subnet

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# Create route table for public subnet
RTB_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RTB_ID --subnet-id $PUB_SUBNET

# Enable auto-assign public IP for public subnet
aws ec2 modify-subnet-attribute --subnet-id $PUB_SUBNET --map-public-ip-on-launch

# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web server security group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
```

---

## RDS - Managed Database

```bash
# Create a DB subnet group (needs 2 AZs)
aws rds create-db-subnet-group \
  --db-subnet-group-name devops-db-subnet \
  --db-subnet-group-description "DevOps DB subnet group" \
  --subnet-ids $PUB_SUBNET $PRIV_SUBNET

# Launch a PostgreSQL RDS instance
aws rds create-db-instance \
  --db-instance-identifier devops-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15 \
  --master-username admin \
  --master-user-password MySecurePass123 \
  --allocated-storage 20 \
  --db-subnet-group-name devops-db-subnet \
  --publicly-accessible \
  --no-multi-az

# Check status
aws rds describe-db-instances \
  --db-instance-identifier devops-db \
  --query 'DBInstances[0].[DBInstanceStatus,Endpoint.Address]'

# Connect to the database
psql -h devops-db.xxxxx.us-east-1.rds.amazonaws.com -U admin -d postgres
```

---

## AWS CLI

```bash
# Common useful commands

# EC2
aws ec2 describe-instances --output table
aws ec2 describe-security-groups
aws ec2 describe-vpcs

# S3
aws s3 ls
aws s3 ls s3://bucket-name/

# CloudWatch (monitoring)
aws cloudwatch list-metrics --namespace AWS/EC2
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --period 300 \
  --statistics Average \
  --dimensions Name=InstanceId,Value=i-0abc123 \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z

# Costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost
```

---

## Serverless - Lambda

### Create a Lambda Function

```python
# File: lambda_function.py
import json
import datetime

def lambda_handler(event, context):
    name = event.get('queryStringParameters', {}).get('name', 'World')

    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({
            'message': f'Hello, {name}!',
            'timestamp': str(datetime.datetime.now()),
            'source': 'AWS Lambda'
        })
    }
```

```bash
# Package the function
zip function.zip lambda_function.py

# Create IAM role for Lambda
aws iam create-role \
  --role-name lambda-basic-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name lambda-basic-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create the function
aws lambda create-function \
  --function-name hello-devops \
  --runtime python3.11 \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::123456789012:role/lambda-basic-role

# Test the function
aws lambda invoke \
  --function-name hello-devops \
  --payload '{"queryStringParameters":{"name":"DevOps Student"}}' \
  response.json

cat response.json

# Create API Gateway (HTTP API)
aws apigatewayv2 create-api \
  --name hello-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:us-east-1:123456789012:function:hello-devops
```

---

## Projects

### Project 1: Web Server on EC2

```bash
#!/bin/bash
# File: deploy-ec2-web.sh
# Deploy a web server on AWS EC2 from scratch

set -e

REGION="us-east-1"
KEY_NAME="devops-web-key"
SG_NAME="web-server-sg"
INSTANCE_NAME="web-server"

echo "=== Step 1: Create Key Pair ==="
aws ec2 create-key-pair \
  --key-name $KEY_NAME \
  --query 'KeyMaterial' \
  --output text > ${KEY_NAME}.pem
chmod 400 ${KEY_NAME}.pem
echo "Key pair created: ${KEY_NAME}.pem"

echo "=== Step 2: Create Security Group ==="
SG_ID=$(aws ec2 create-security-group \
  --group-name $SG_NAME \
  --description "Web server - SSH and HTTP" \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
echo "Security group created: $SG_ID"

echo "=== Step 3: Get Latest Ubuntu AMI ==="
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)
echo "AMI: $AMI_ID"

echo "=== Step 4: Launch EC2 Instance ==="
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID \
  --user-data file://userdata.sh \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$INSTANCE_NAME}]" \
  --query 'Instances[0].InstanceId' --output text)
echo "Instance launched: $INSTANCE_ID"

echo "=== Step 5: Wait for Instance to be Running ==="
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

echo ""
echo "========================================="
echo "  Web server deployed!"
echo "  Instance ID: $INSTANCE_ID"
echo "  Public IP:   $PUBLIC_IP"
echo "  URL:         http://$PUBLIC_IP"
echo "  SSH:         ssh -i ${KEY_NAME}.pem ubuntu@$PUBLIC_IP"
echo "========================================="
```

### Project 2: S3 Static Website

```bash
#!/bin/bash
# File: deploy-s3-website.sh

BUCKET="my-devops-website-$(date +%s)"
REGION="us-east-1"

echo "=== Creating S3 bucket: $BUCKET ==="
aws s3 mb s3://$BUCKET --region $REGION

echo "=== Configuring static website hosting ==="
aws s3 website s3://$BUCKET/ --index-document index.html --error-document error.html

echo "=== Setting bucket policy for public access ==="
aws s3api put-public-access-block \
  --bucket $BUCKET \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

aws s3api put-bucket-policy --bucket $BUCKET --policy "{
  \"Version\":\"2012-10-17\",
  \"Statement\":[{
    \"Sid\":\"PublicRead\",
    \"Effect\":\"Allow\",
    \"Principal\":\"*\",
    \"Action\":\"s3:GetObject\",
    \"Resource\":\"arn:aws:s3:::${BUCKET}/*\"
  }]
}"

echo "=== Creating website files ==="
mkdir -p /tmp/website

cat > /tmp/website/index.html << 'HTML'
<!DOCTYPE html>
<html>
<head><title>My DevOps Website</title></head>
<body style="font-family:Arial; text-align:center; padding:50px; background:#1a1a2e; color:#fff;">
  <h1 style="color:#e94560;">My DevOps Website on S3</h1>
  <p>Hosted on Amazon S3 Static Website Hosting</p>
</body>
</html>
HTML

cat > /tmp/website/error.html << 'HTML'
<!DOCTYPE html>
<html>
<head><title>404 - Not Found</title></head>
<body style="text-align:center; padding:50px;">
  <h1>404 - Page Not Found</h1>
</body>
</html>
HTML

echo "=== Uploading files ==="
aws s3 sync /tmp/website/ s3://$BUCKET/

echo ""
echo "========================================="
echo "  Website deployed!"
echo "  URL: http://${BUCKET}.s3-website-${REGION}.amazonaws.com"
echo "  Bucket: $BUCKET"
echo "========================================="
```

### Project 3: Custom VPC with Subnets

```bash
#!/bin/bash
# File: create-vpc.sh
# Create a production-like VPC with public and private subnets

set -e
REGION="us-east-1"

echo "=== Creating VPC ==="
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value=devops-vpc
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
echo "VPC: $VPC_ID"

echo "=== Creating Subnets ==="
PUB1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone ${REGION}a --query 'Subnet.SubnetId' --output text)
PUB2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone ${REGION}b --query 'Subnet.SubnetId' --output text)
PRIV1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone ${REGION}a --query 'Subnet.SubnetId' --output text)
PRIV2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.4.0/24 --availability-zone ${REGION}b --query 'Subnet.SubnetId' --output text)

aws ec2 create-tags --resources $PUB1 --tags Key=Name,Value=public-1a
aws ec2 create-tags --resources $PUB2 --tags Key=Name,Value=public-1b
aws ec2 create-tags --resources $PRIV1 --tags Key=Name,Value=private-1a
aws ec2 create-tags --resources $PRIV2 --tags Key=Name,Value=private-1b
echo "Subnets created"

echo "=== Creating Internet Gateway ==="
IGW=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC_ID
echo "IGW: $IGW"

echo "=== Setting up Route Tables ==="
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB1
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB2

aws ec2 modify-subnet-attribute --subnet-id $PUB1 --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUB2 --map-public-ip-on-launch

echo "=== Creating Security Groups ==="
WEB_SG=$(aws ec2 create-security-group --group-name web-sg --description "Web tier" --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 22 --cidr 0.0.0.0/0

DB_SG=$(aws ec2 create-security-group --group-name db-sg --description "DB tier" --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 5432 --source-group $WEB_SG

echo ""
echo "========================================="
echo "  VPC Infrastructure Created!"
echo "  VPC:          $VPC_ID"
echo "  Public Sub 1: $PUB1 (10.0.1.0/24)"
echo "  Public Sub 2: $PUB2 (10.0.2.0/24)"
echo "  Private Sub 1: $PRIV1 (10.0.3.0/24)"
echo "  Private Sub 2: $PRIV2 (10.0.4.0/24)"
echo "  Web SG:       $WEB_SG"
echo "  DB SG:        $DB_SG"
echo "========================================="
```
