# AWS CDK Deployment Guide for Trading System

This guide provides instructions for deploying the Trading System using AWS CDK in both TypeScript and Python.

## TypeScript Implementation

### 1. Initialize CDK Project

```bash
# Create a new directory for CDK infrastructure
mkdir cdk-infrastructure
cd cdk-infrastructure

# Initialize CDK project
npx aws-cdk@latest init app --language typescript

# Install additional CDK libraries
npm install @aws-cdk/aws-ecs @aws-cdk/aws-ecs-patterns @aws-cdk/aws-rds @aws-cdk/aws-ec2 @aws-cdk/aws-cloudfront @aws-cdk/aws-s3
```


### 2. Create CDK Stack Structure (TypeScript)

Create `lib/trading-system-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecs_patterns from 'aws-cdk-lib/aws-ecs-patterns';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';

export class TradingSystemStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // 1. Create VPC
    const vpc = new ec2.Vpc(this, 'TradingSystemVpc', {
      maxAzs: 2,
      natGateways: 2
    });

    // 2. Create ECS Cluster
    const cluster = new ecs.Cluster(this, 'TradingSystemCluster', {
      vpc,
      clusterName: 'trading-system-cluster'
    });

    // 3. Create RDS Databases
    const tradeDataStore = new rds.DatabaseCluster(this, 'TradeDataStore', {
      engine: rds.DatabaseClusterEngine.auroraPostgres({
        version: rds.AuroraPostgresEngineVersion.VER_14_7
      }),
      instanceProps: {
        instanceType: ec2.InstanceType.of(ec2.InstanceClass.R6G, ec2.InstanceSize.XLARGE),
        vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
        vpc
      },
      instances: 2,
      backup: { retention: cdk.Duration.days(30) },
      storageEncrypted: true
    });

    // 4. Create Fargate Services
    const accountService = new ecs_patterns.ApplicationLoadBalancedFargateService(this, 'AccountService', {
      cluster,
      cpu: 1024,
      memoryLimitMiB: 2048,
      desiredCount: 3,
      taskImageOptions: {
        image: ecs.ContainerImage.fromRegistry('account-service:latest'),
        containerPort: 18088,
        environment: {
          SPRING_PROFILES_ACTIVE: 'production',
          DB_HOST: tradeDataStore.clusterEndpoint.hostname
        }
      },
      publicLoadBalancer: true
    });

    // Grant database access
    tradeDataStore.connections.allowFrom(accountService.service, ec2.Port.tcp(5432));

    // Repeat for other services: trade-service, position-service, etc.

    // 5. Create S3 + CloudFront for web-gui
    const webBucket = new s3.Bucket(this, 'WebGuiBucket', {
      websiteIndexDocument: 'index.html',
      publicReadAccess: true
    });

    const distribution = new cloudfront.CloudFrontWebDistribution(this, 'WebGuiDistribution', {
      originConfigs: [{
        s3OriginSource: { s3BucketSource: webBucket },
        behaviors: [{ isDefaultBehavior: true }]
      }]
    });
  }
}
```

### 3. Deploy with CDK (TypeScript)

```bash
cd cdk-infrastructure

# Bootstrap CDK (first time only)
npx cdk bootstrap aws://ACCOUNT-ID/us-east-1

# Synthesize CloudFormation template
npx cdk synth

# Deploy the stack
npx cdk deploy TradingSystemStack
```

---

## Python Implementation

### 1. Initialize Python CDK Project

```bash
# Create a new directory for CDK infrastructure
mkdir cdk-infrastructure
cd cdk-infrastructure

# Initialize Python CDK project
cdk init app --language python

# Activate virtual environment
source .venv/bin/activate  # On macOS/Linux
# .venv\Scripts\activate.bat  # On Windows

# Install required CDK libraries
pip install -r requirements.txt
```

### 2. Create CDK Stack Structure (Python)

Update `cdk_infrastructure/trading_system_stack.py`:

```python
from aws_cdk import (
    Stack,
    Duration,
    aws_ec2 as ec2,
    aws_ecs as ecs,
    aws_ecs_patterns as ecs_patterns,
    aws_rds as rds,
    aws_s3 as s3,
    aws_cloudfront as cloudfront,
    aws_secretsmanager as secretsmanager,
    aws_logs as logs,
    aws_applicationautoscaling as appscaling,
)
from constructs import Construct

class TradingSystemStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # 1. Create VPC
        vpc = ec2.Vpc(
            self, "TradingSystemVpc",
            max_azs=2,
            nat_gateways=2,
            subnet_configuration=[
                ec2.SubnetConfiguration(
                    name="Public",
                    subnet_type=ec2.SubnetType.PUBLIC
                ),
                ec2.SubnetConfiguration(
                    name="Private",
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
                ),
                ec2.SubnetConfiguration(
                    name="Isolated",
                    subnet_type=ec2.SubnetType.PRIVATE_ISOLATED
                )
            ]
        )

        # 2. Create VPC Endpoints
        vpc.add_interface_endpoint(
            "EcrApiEndpoint",
            service=ec2.InterfaceVpcEndpointAwsService.ECR
        )
        vpc.add_interface_endpoint(
            "EcrDkrEndpoint",
            service=ec2.InterfaceVpcEndpointAwsService.ECR_DOCKER
        )
        vpc.add_interface_endpoint(
            "CloudWatchLogsEndpoint",
            service=ec2.InterfaceVpcEndpointAwsService.CLOUDWATCH_LOGS
        )

        # 3. Create ECS Cluster
        cluster = ecs.Cluster(
            self, "TradingSystemCluster",
            vpc=vpc,
            cluster_name="trading-system-cluster",
            container_insights=True
        )

        # 4. Create CloudWatch Log Group
        log_group = logs.LogGroup(
            self, "TradingSystemLogs",
            log_group_name="/aws/ecs/trading-system",
            retention=logs.RetentionDays.ONE_MONTH
        )

        # 5. Create Database Credentials Secret
        db_secret = secretsmanager.Secret(
            self, "DatabaseCredentials",
            secret_name="/trading-system/db-credentials",
            generate_secret_string=secretsmanager.SecretStringGenerator(
                secret_string_template='{"username": "admin"}',
                generate_string_key="password",
                exclude_punctuation=True
            )
        )

        # 6. Create RDS Aurora PostgreSQL - Trade Data Store
        trade_data_store = rds.DatabaseCluster(
            self, "TradeDataStore",
            engine=rds.DatabaseClusterEngine.aurora_postgres(
                version=rds.AuroraPostgresEngineVersion.VER_14_7
            ),
            credentials=rds.Credentials.from_secret(db_secret),
            instance_props=rds.InstanceProps(
                instance_type=ec2.InstanceType.of(
                    ec2.InstanceClass.MEMORY6_GRAVITON,
                    ec2.InstanceSize.XLARGE
                ),
                vpc_subnets=ec2.SubnetSelection(
                    subnet_type=ec2.SubnetType.PRIVATE_ISOLATED
                ),
                vpc=vpc
            ),
            instances=2,
            backup=rds.BackupProps(retention=Duration.days(30)),
            storage_encrypted=True
        )

        # 7. Create RDS Aurora PostgreSQL - User Directory
        user_directory = rds.DatabaseCluster(
            self, "UserDirectory",
            engine=rds.DatabaseClusterEngine.aurora_postgres(
                version=rds.AuroraPostgresEngineVersion.VER_14_7
            ),
            credentials=rds.Credentials.from_secret(db_secret),
            instance_props=rds.InstanceProps(
                instance_type=ec2.InstanceType.of(
                    ec2.InstanceClass.MEMORY6_GRAVITON,
                    ec2.InstanceSize.LARGE
                ),
                vpc_subnets=ec2.SubnetSelection(
                    subnet_type=ec2.SubnetType.PRIVATE_ISOLATED
                ),
                vpc=vpc
            ),
            instances=1,
            read_replicas=2,
            storage_encrypted=True
        )

        # 8. Create Account Service
        account_service = ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "AccountService",
            cluster=cluster,
            cpu=1024,
            memory_limit_mib=2048,
            desired_count=3,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry("account-service:latest"),
                container_port=18088,
                environment={
                    "SPRING_PROFILES_ACTIVE": "production",
                    "DB_HOST": trade_data_store.cluster_endpoint.hostname,
                    "DB_PORT": "5432"
                },
                secrets={
                    "DB_PASSWORD": ecs.Secret.from_secrets_manager(db_secret, "password")
                },
                log_driver=ecs.LogDrivers.aws_logs(
                    stream_prefix="account-service",
                    log_group=log_group
                )
            ),
            public_load_balancer=True
        )
        
        account_service.target_group.configure_health_check(
            path="/actuator/health",
            interval=Duration.seconds(30)
        )
        
        trade_data_store.connections.allow_from(
            account_service.service,
            ec2.Port.tcp(5432)
        )

        # 9. Create Trade Service with Auto-Scaling
        trade_service = ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "TradeService",
            cluster=cluster,
            cpu=1024,
            memory_limit_mib=2048,
            desired_count=3,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry("trade-service:latest"),
                container_port=18092,
                environment={"SPRING_PROFILES_ACTIVE": "production"},
                log_driver=ecs.LogDrivers.aws_logs(
                    stream_prefix="trade-service",
                    log_group=log_group
                )
            ),
            public_load_balancer=True
        )
        
        scalable_target = trade_service.service.auto_scale_task_count(
            min_capacity=2,
            max_capacity=10
        )
        scalable_target.scale_on_cpu_utilization(
            "CpuScaling",
            target_utilization_percent=70
        )

        # 10. Create Position Service
        position_service = ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "PositionService",
            cluster=cluster,
            cpu=1024,
            memory_limit_mib=2048,
            desired_count=3,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry("position-service:latest"),
                container_port=18090,
                environment={
                    "SPRING_PROFILES_ACTIVE": "production",
                    "DB_HOST": trade_data_store.cluster_endpoint.hostname
                },
                log_driver=ecs.LogDrivers.aws_logs(
                    stream_prefix="position-service",
                    log_group=log_group
                )
            )
        )
        
        trade_data_store.connections.allow_from(position_service.service, ec2.Port.tcp(5432))

        # 11. Create Trade Processor
        trade_processor = ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "TradeProcessor",
            cluster=cluster,
            cpu=2048,
            memory_limit_mib=4096,
            desired_count=2,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry("trade-processor:latest"),
                container_port=18091,
                environment={
                    "SPRING_PROFILES_ACTIVE": "production",
                    "DB_HOST": trade_data_store.cluster_endpoint.hostname
                },
                log_driver=ecs.LogDrivers.aws_logs(
                    stream_prefix="trade-processor",
                    log_group=log_group
                )
            )
        )
        
        trade_data_store.connections.allow_from(trade_processor.service, ec2.Port.tcp(5432))

        # 12. Create Security Master
        security_master = ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "SecurityMaster",
            cluster=cluster,
            cpu=512,
            memory_limit_mib=1024,
            desired_count=2,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry("security-master:latest"),
                container_port=18085,
                environment={"NODE_ENV": "production"},
                log_driver=ecs.LogDrivers.aws_logs(
                    stream_prefix="security-master",
                    log_group=log_group
                )
            )
        )

        # 13. Create People Service
        people_service = ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "PeopleService",
            cluster=cluster,
            cpu=512,
            memory_limit_mib=1024,
            desired_count=2,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry("people-service:latest"),
                container_port=18089,
                environment={
                    "ASPNETCORE_ENVIRONMENT": "Production",
                    "DB_HOST": user_directory.cluster_endpoint.hostname
                },
                log_driver=ecs.LogDrivers.aws_logs(
                    stream_prefix="people-service",
                    log_group=log_group
                )
            )
        )
        
        user_directory.connections.allow_from(people_service.service, ec2.Port.tcp(5432))

        # 14. Create Trade Feed with WebSocket Support
        trade_feed = ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "TradeFeed",
            cluster=cluster,
            cpu=1024,
            memory_limit_mib=2048,
            desired_count=2,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry("trade-feed:latest"),
                container_port=18086,
                environment={"NODE_ENV": "production"},
                log_driver=ecs.LogDrivers.aws_logs(
                    stream_prefix="trade-feed",
                    log_group=log_group
                )
            )
        )
        
        # Enable sticky sessions for WebSocket
        trade_feed.target_group.enable_cookie_stickiness(duration=Duration.hours(1))

        # 15. Create S3 Bucket for Web GUI
        web_bucket = s3.Bucket(
            self, "WebGuiBucket",
            bucket_name="trading-system-web-gui",
            public_read_access=False,
            block_public_access=s3.BlockPublicAccess.BLOCK_ALL,
            encryption=s3.BucketEncryption.S3_MANAGED
        )

        # 16. Create CloudFront Distribution
        origin_access_identity = cloudfront.OriginAccessIdentity(self, "WebGuiOAI")
        web_bucket.grant_read(origin_access_identity)

        distribution = cloudfront.CloudFrontWebDistribution(
            self, "WebGuiDistribution",
            origin_configs=[
                cloudfront.SourceConfiguration(
                    s3_origin_source=cloudfront.S3OriginConfig(
                        s3_bucket_source=web_bucket,
                        origin_access_identity=origin_access_identity
                    ),
                    behaviors=[cloudfront.Behavior(is_default_behavior=True)]
                )
            ],
            default_root_object="index.html",
            error_configurations=[
                cloudfront.CfnDistribution.CustomErrorResponseProperty(
                    error_code=404,
                    response_code=200,
                    response_page_path="/index.html"
                )
            ]
        )
```

Update `app.py`:

```python
#!/usr/bin/env python3
import os
import aws_cdk as cdk
from cdk_infrastructure.trading_system_stack import TradingSystemStack

app = cdk.App()

TradingSystemStack(
    app,
    "TradingSystemStack",
    env=cdk.Environment(
        account=os.getenv('CDK_DEFAULT_ACCOUNT'),
        region=os.getenv('CDK_DEFAULT_REGION', 'us-east-1')
    ),
    description="Trading System Infrastructure deployed via AWS CDK"
)

app.synth()
```

### 3. Deploy with CDK (Python)

```bash
# Ensure virtual environment is activated
source .venv/bin/activate

# Bootstrap CDK (first time only)
cdk bootstrap aws://ACCOUNT-ID/us-east-1

# Synthesize CloudFormation template
cdk synth

# Deploy the stack
cdk deploy TradingSystemStack
```

---

## Common Steps for Both Implementations

### Build and Push Container Images

```bash
# For each service, create Dockerfiles and build images
cd account-service
docker build -t account-service:latest .

# Create ECR repositories
aws ecr create-repository --repository-name account-service
aws ecr create-repository --repository-name trade-service
aws ecr create-repository --repository-name position-service
aws ecr create-repository --repository-name trade-processor
aws ecr create-repository --repository-name security-master
aws ecr create-repository --repository-name people-service
aws ecr create-repository --repository-name trade-feed

# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Tag and push images
docker tag account-service:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/account-service:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/account-service:latest

# Repeat for all services (trade-service, position-service, etc.)
```

### Update Decorator After Deployment

After successful deployment, update [trading-system-aws-cdk.decorator.json](trading-system-aws-cdk.decorator.json) with actual values:

- Replace placeholder ARNs with real resource ARNs from CloudFormation outputs
- Add actual VPC ID, subnet IDs, and security group IDs
- Update load balancer ARNs and target group ARNs
- Add CloudFront distribution ID
- Include deployment timestamps
- Document actual database endpoints

Example CDK outputs to capture:

```bash
# Get stack outputs
aws cloudformation describe-stacks \
  --stack-name TradingSystemStack \
  --query 'Stacks[0].Outputs' \
  --output table
```

### Key Deployment Considerations

1. **Environment Variables**: Services need database endpoints, ports, and credentials
2. **Security Groups**: Configure proper network segmentation between services
3. **IAM Roles**: Services require permissions for:
   - AWS Secrets Manager (database credentials, API keys)
   - CloudWatch Logs (logging and metrics)
   - X-Ray (distributed tracing)
   - ECR (container image pulls)
4. **Secrets Management**: Use AWS Secrets Manager for:
   - Database credentials
   - API keys
   - Third-party service credentials
5. **Monitoring & Observability**:
   - Enable CloudWatch Container Insights
   - Configure X-Ray tracing for distributed tracing
   - Set up CloudWatch dashboards
   - Create alarms for critical metrics
6. **Load Balancer Configuration**:
   - Configure health checks matching service endpoints (e.g., `/actuator/health`)
   - Set appropriate timeout values
   - Enable sticky sessions for WebSocket services (trade-feed)
7. **Database Configuration**:
   - Configure parameter groups for PostgreSQL optimization
   - Set up automated backups (30-day retention as per decorator)
   - Enable encryption at rest
   - Configure multi-AZ for high availability
8. **Networking**:
   - Use VPC endpoints to avoid NAT gateway costs for AWS services
   - Configure proper subnet segmentation (public/private/isolated)
   - Set up security groups with least-privilege access
9. **Cost Optimization**:
   - Use Fargate Spot for non-critical workloads
   - Configure auto-scaling policies based on CPU/memory utilization
   - Review CloudWatch Logs retention policies

### Architecture Alignment

The CDK stack implements the architecture defined in:
- **CALM Architecture**: [trading-system.architecture.json](trading-system.architecture.json)
- **Deployment Decorator**: [trading-system-aws-cdk.decorator.json](trading-system-aws-cdk.decorator.json)

The decorator serves as the architectural blueprint—the CDK code is the infrastructure-as-code implementation that brings it to life on AWS.
