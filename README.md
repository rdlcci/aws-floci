# aws-floci-lab

Learn AWS by building, breaking, and measuring — entirely on your laptop, no AWS
account, no bill.

See **[SKILL.md](./SKILL.md)** for the full curriculum, repo structure, and the
"break-it experiment" for every component (VPC, NAT, IGW, ALB/NLB, EC2, EBS, AMI,
ASG, ECR, ECS, S3, RDS, DynamoDB, ElastiCache, API Gateway, CloudFront, Route53,
SQS/SNS/EventBridge, Lambda, CloudWatch, IAM, KMS, Secrets Manager).

## Quick start

```bash
# 1. Start Floci
docker compose up -d

# 2. Configure AWS CLI to point at it
aws configure --profile floci   # dummy creds , region us-east-1

# AWS Access Key ID: test (or any dummy value)
# AWS Secret Access Key: test (or any dummy value)
# Default region: us-east-1
# Default output format: json (optional)

cat >> ~/.aws/config << 'EOF'

[profile floci]
region = us-east-1
output = json
endpoint_url = http://localhost:4566
EOF

# 3. Sanity check
aws s3 mb s3://sanity-check --profile floci
aws s3 ls --profile floci
```

## Where to start

Go to `00-foundations/vpc-networking/` first — everything else in this repo sits
inside a VPC. Then follow the module order in `SKILL.md` top to bottom, or jump
straight to `10-architectures/with-and-without-cache/` if you want the payoff
experiment first and plan to backfill the fundamentals as you hit gaps.