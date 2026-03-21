# 1 Initialize CDK Project

```bash
# Create a new directory for CDK infrastructure
mkdir cdk-infrastructure
cd cdk-infrastructure

# Initialize CDK project
npx aws-cdk@latest init app --language typescript

# Install additional CDK libraries
npm install @aws-cdk/aws-ecs @aws-cdk/aws-ecs-patterns @aws-cdk/aws-rds @aws-cdk/aws-ec2 @aws-cdk/aws-cloudfront @aws-cdk/aws-s3
```


# 2 Create CDK Stack Structure
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

# 3 Build Container Images

```bash
# For each service, create Dockerfiles and build images
cd ../account-service
docker build -t account-service:latest .
docker tag account-service:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/account-service:latest

# Push to ECR
aws ecr create-repository --repository-name account-service
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/account-service:latest

# Repeat for all services
```



# 4 Deploy with CDK

```bash
cd cdk-infrastructure

# Bootstrap CDK (first time only)
npx cdk bootstrap aws://ACCOUNT-ID/us-east-1

# Synthesize CloudFormation template
npx cdk synth

# Deploy the stack
npx cdk deploy TradingSystemStack
```

# 5 Update Decorator with Actual Values

After deployment, update trading-system-aws-cdk.decorator.json with actual values:

- Replace placeholder ARNs with real ones
- Add actual VPC/subnet/security group IDs
- Update CloudFormation stack outputs
- Add deployment timestamps

**Key Considerations**
- Environment Variables: Services need database endpoints, ports, and credentials
- Security Groups: Configure proper network segmentation
- IAM Roles: Services need permissions for AWS APIs, Secrets Manager, CloudWatch
- Secrets: Use AWS Secrets Manager for database credentials and API keys
- Monitoring: Enable CloudWatch Logs, X-Ray tracing, and custom metrics
-  Load Balancer: Configure health checks matching your service endpoints

The decorator serves as your architectural blueprint - now you need to implement the actual infrastructure code to match it.