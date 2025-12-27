# AutoMSP Deployment Guide

## Overview
This guide provides step-by-step instructions for deploying AutoMSP to production cloud infrastructure. AutoMSP is designed for deployment on modern cloud platforms with automated CI/CD pipelines.

## Prerequisites

### Required Tools
- Node.js 18+ or 20+
- Docker & Docker Compose
- AWS CLI v2 (or equivalent cloud CLI)
- kubectl (Kubernetes)
- Terraform (for infrastructure as code)
- GitHub CLI

### Cloud Infrastructure
- AWS EC2 or ECS/Fargate for compute
- RDS PostgreSQL for database
- S3 for file storage
- CloudFront for CDN
- Route 53 for DNS
- AWS Secrets Manager for secrets

### Credentials & Access
- AWS IAM credentials with appropriate permissions
- GitHub repository access
- Domain name (DNS)
- SSL certificate (Let's Encrypt or AWS ACM)

## Environment Setup

### 1. Clone Repository
```bash
git clone https://github.com/mynewopportunities/vibe-dev-toolkit.git
cd vibe-dev-toolkit
```

### 2. Install Dependencies
```bash
npm install
```

### 3. Configure Environment Variables
Create `.env.production` file:
```bash
# Application
NODE_ENV=production
PORT=3000
APP_URL=https://api.automsp.com

# Database
DB_HOST=automsp-db.xxxxx.rds.amazonaws.com
DB_PORT=5432
DB_NAME=automsp_prod
DB_USER=automsp_user
DB_PASSWORD=<secure-password>

# Authentication
JWT_SECRET=<secure-jwt-secret>
JWT_EXPIRY=7d

# Stripe
STRIPE_PUBLIC_KEY=pk_live_xxx
STRIPE_SECRET_KEY=sk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# AWS
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=<access-key>
AWS_SECRET_ACCESS_KEY=<secret-key>
AWS_S3_BUCKET=automsp-prod-bucket

# Email
SES_REGION=us-east-1
SES_FROM_ADDRESS=noreply@automsp.com

# Logging
LOG_LEVEL=info
CLOUDWATCH_LOG_GROUP=/aws/automsp/api
```

## Docker Deployment

### 1. Build Docker Image
```bash
docker build -t automsp:latest .
docker build -t automsp:v1.0.0 .
```

### 2. Push to ECR
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

docker tag automsp:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/automsp:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/automsp:latest
```

### 3. Local Testing
```bash
docker-compose up -d
curl http://localhost:3000/health
```

## AWS ECS/Fargate Deployment

### 1. Create ECS Cluster
```bash
aws ecs create-cluster --cluster-name automsp-prod
```

### 2. Create Task Definition
Create `ecs-task-definition.json`:
```json
{
  "family": "automsp-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "automsp-api",
      "image": "<account-id>.dkr.ecr.us-east-1.amazonaws.com/automsp:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:<account-id>:secret:automsp/db-password"
        },
        {
          "name": "STRIPE_SECRET_KEY",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:<account-id>:secret:automsp/stripe-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/aws/automsp/api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

Register task definition:
```bash
aws ecs register-task-definition --cli-input-json file://ecs-task-definition.json
```

### 3. Create ECS Service
```bash
aws ecs create-service \
  --cluster automsp-prod \
  --service-name automsp-api \
  --task-definition automsp-api:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-xxx],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=automsp-api,containerPort=3000"
```

## Database Setup

### 1. Create RDS Instance
```bash
aws rds create-db-instance \
  --db-instance-identifier automsp-prod-db \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --master-username automsp_user \
  --master-user-password <secure-password> \
  --allocated-storage 100 \
  --storage-type gp3 \
  --backup-retention-period 30
```

### 2. Run Database Migrations
```bash
NODE_ENV=production npm run migrate
NODE_ENV=production npm run seed
```

## DNS & SSL Configuration

### 1. Configure Route 53
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --change-batch file://dns-changes.json
```

### 2. Create SSL Certificate
```bash
aws acm request-certificate \
  --domain-name api.automsp.com \
  --subject-alternative-names automsp.com *.automsp.com \
  --validation-method DNS
```

## Monitoring & Logging

### 1. CloudWatch Logs
```bash
aws logs create-log-group --log-group-name /aws/automsp/api
aws logs put-retention-policy --log-group-name /aws/automsp/api --retention-in-days 30
```

### 2. CloudWatch Alarms
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name automsp-api-error-rate \
  --alarm-description "Alert on high error rate" \
  --metric-name ErrorRate \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold
```

### 3. Application Metrics
- Response times
- Error rates
- Database connection pools
- Memory usage
- CPU usage
- Request throughput

## Backup & Disaster Recovery

### 1. RDS Backups
- Automated daily backups (30-day retention)
- Manual snapshots before major changes
- Cross-region replication for critical data

### 2. Application State
- Database snapshots
- S3 versioning enabled
- Regular backups to separate account

### 3. Recovery Procedures
- RTO (Recovery Time Objective): 1 hour
- RPO (Recovery Point Objective): 15 minutes
- Regular disaster recovery drills

## Scaling Configuration

### 1. Horizontal Scaling
```bash
aws ecs update-service \
  --cluster automsp-prod \
  --service automsp-api \
  --desired-count 5
```

### 2. Auto Scaling
```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/automsp-prod/automsp-api \
  --min-capacity 2 \
  --max-capacity 10
```

## Security Hardening

### 1. VPC Configuration
- Private subnets for databases
- Security groups with least privilege
- NACLs for additional protection
- VPC endpoints for AWS services

### 2. Secrets Management
- Use AWS Secrets Manager
- Rotate credentials every 30 days
- Audit all secret access
- Never commit secrets to Git

### 3. SSL/TLS
- Enable HSTS headers
- TLS 1.2 minimum
- Regular certificate updates
- OCSP stapling enabled

## Health Checks

### 1. Application Health
```bash
GET /health
GET /health/db
GET /health/external-apis
```

### 2. Load Balancer Health Checks
- Path: `/health`
- Port: 3000
- Interval: 30 seconds
- Healthy threshold: 2
- Unhealthy threshold: 3

## Troubleshooting

### Common Issues

#### Service fails to start
1. Check CloudWatch logs
2. Verify environment variables
3. Check database connectivity
4. Review IAM permissions

#### High latency
1. Check database query performance
2. Review CloudFront cache hit ratio
3. Check application logs for slow operations
4. Monitor external API response times

#### Database connection errors
1. Verify RDS security groups
2. Check connection pool settings
3. Review database logs
4. Restart RDS instance if needed

## Maintenance & Updates

### 1. Regular Patching
- OS security updates
- Dependency updates (npm)
- Database minor versions
- Container base images

### 2. Performance Tuning
- Database query optimization
- Cache configuration
- Load balancer settings
- Auto-scaling thresholds

## Deployment Checklist

- [ ] Environment variables configured
- [ ] Database migrations completed
- [ ] SSL certificate installed
- [ ] DNS records updated
- [ ] Health checks passing
- [ ] Monitoring & alerting configured
- [ ] Backup procedures validated
- [ ] Load testing completed
- [ ] Security audit passed
- [ ] Team trained on deployment

## Rollback Procedures

### Quick Rollback
```bash
aws ecs update-service \
  --cluster automsp-prod \
  --service automsp-api \
  --force-new-deployment
```

### Full Rollback
1. Stop current ECS service
2. Deploy previous task definition version
3. Verify health checks
4. Monitor error rates

## Support & Resources

- AWS Documentation: https://docs.aws.amazon.com
- ECS Best Practices: https://docs.aws.amazon.com/AmazonECS/latest/developerguide
- Terraform AWS Provider: https://registry.terraform.io/providers/hashicorp/aws
- GitHub Actions Documentation: https://docs.github.com/en/actions
