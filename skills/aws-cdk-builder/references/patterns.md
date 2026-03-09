# CDK Architecture Patterns

Reference patterns for common AWS CDK architectures. Always verify current APIs via `search_cdk_documentation` before using — these patterns illustrate structure, not guaranteed current syntax.

## Table of Contents
- [Lambda + DynamoDB](#lambda--dynamodb)
- [REST API Gateway + Lambda](#rest-api-gateway--lambda)
- [S3 + CloudFront (Static Site)](#s3--cloudfront-static-site)
- [ECS Fargate + Application Load Balancer](#ecs-fargate--application-load-balancer)
- [VPC with Private/Public Subnets](#vpc-with-privatepublic-subnets)
- [Event-Driven: SNS + SQS + Lambda](#event-driven-sns--sqs--lambda)
- [Step Functions Workflow](#step-functions-workflow)
- [Security & IAM Patterns](#security--iam-patterns)

---

## Lambda + DynamoDB

```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

const table = new dynamodb.Table(this, 'Table', {
  partitionKey: { name: 'pk', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'sk', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  encryption: dynamodb.TableEncryption.AWS_MANAGED,
  pointInTimeRecoverySpecification: { pointInTimeRecoveryEnabled: true },
  removalPolicy: cdk.RemovalPolicy.RETAIN, // Always explicit for stateful resources
});

const fn = new lambda.Function(this, 'Handler', {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  environment: { TABLE_NAME: table.tableName },
});

// Use grant* methods — never manual IAM policy statements
table.grantReadWriteData(fn);
```

**Search for more detail:** `search_cdk_documentation(query="dynamodb Table encryption pointInTimeRecovery")`

---

## REST API Gateway + Lambda

```typescript
import * as apigw from 'aws-cdk-lib/aws-apigateway';
import * as lambda from 'aws-cdk-lib/aws-lambda';

const handler = new lambda.Function(this, 'ApiHandler', { /* ... */ });

const api = new apigw.RestApi(this, 'Api', {
  restApiName: 'MyService',
  defaultCorsPreflightOptions: {
    allowOrigins: apigw.Cors.ALL_ORIGINS,
    allowMethods: apigw.Cors.ALL_METHODS,
  },
  deployOptions: {
    stageName: 'prod',
    loggingLevel: apigw.MethodLoggingLevel.INFO,
    dataTraceEnabled: true,
  },
});

const items = api.root.addResource('items');
items.addMethod('GET', new apigw.LambdaIntegration(handler));
items.addMethod('POST', new apigw.LambdaIntegration(handler));
```

**With Cognito authorizer:**
```typescript
import * as cognito from 'aws-cdk-lib/aws-cognito';

const userPool = new cognito.UserPool(this, 'UserPool', {
  selfSignUpEnabled: true,
  standardAttributes: { email: { required: true } },
});

const authorizer = new apigw.CognitoUserPoolsAuthorizer(this, 'Authorizer', {
  cognitoUserPools: [userPool],
});

items.addMethod('GET', new apigw.LambdaIntegration(handler), {
  authorizer,
  authorizationType: apigw.AuthorizationType.COGNITO,
});
```

---

## S3 + CloudFront (Static Site)

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';
import * as origins from 'aws-cdk-lib/aws-cloudfront-origins';
import * as s3deploy from 'aws-cdk-lib/aws-s3-deployment';

const bucket = new s3.Bucket(this, 'SiteBucket', {
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  encryption: s3.BucketEncryption.S3_MANAGED,
  enforceSSL: true,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  autoDeleteObjects: true,
});

const distribution = new cloudfront.Distribution(this, 'Distribution', {
  defaultBehavior: {
    origin: origins.S3BucketOrigin.withOriginAccessControl(bucket),
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
  },
  defaultRootObject: 'index.html',
  errorResponses: [
    { httpStatus: 404, responseHttpStatus: 200, responsePagePath: '/index.html' },
  ],
});

new s3deploy.BucketDeployment(this, 'Deploy', {
  sources: [s3deploy.Source.asset('./dist')],
  destinationBucket: bucket,
  distribution,
  distributionPaths: ['/*'],
});
```

---

## ECS Fargate + Application Load Balancer

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecsPatterns from 'aws-cdk-lib/aws-ecs-patterns';

const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2 });

const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

// L3 pattern — handles ALB, task definition, and service wiring
const service = new ecsPatterns.ApplicationLoadBalancedFargateService(this, 'Service', {
  cluster,
  memoryLimitMiB: 512,
  cpu: 256,
  desiredCount: 2,
  taskImageOptions: {
    image: ecs.ContainerImage.fromRegistry('nginx:latest'),
    containerPort: 80,
    environment: { ENV: 'production' },
  },
  publicLoadBalancer: true,
  redirectHTTP: true,
});

// Auto-scaling
service.scalableTaskCount.scaleOnCpuUtilization('CpuScaling', {
  targetUtilizationPercent: 70,
});
```

---

## VPC with Private/Public Subnets

```typescript
const vpc = new ec2.Vpc(this, 'Vpc', {
  maxAzs: 3,
  natGateways: 1, // Cost optimization: 1 NAT GW (use 3 for HA production)
  subnetConfiguration: [
    { name: 'Public', subnetType: ec2.SubnetType.PUBLIC, cidrMask: 24 },
    { name: 'Private', subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS, cidrMask: 24 },
    { name: 'Isolated', subnetType: ec2.SubnetType.PRIVATE_ISOLATED, cidrMask: 28 },
  ],
});

// Security group with least-privilege
const appSg = new ec2.SecurityGroup(this, 'AppSg', {
  vpc,
  allowAllOutbound: false, // Explicit control
});
appSg.addEgressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(443), 'HTTPS out');
```

---

## Event-Driven: SNS + SQS + Lambda

```typescript
import * as sns from 'aws-cdk-lib/aws-sns';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';
import * as lambdaEventSources from 'aws-cdk-lib/aws-lambda-event-sources';

const dlq = new sqs.Queue(this, 'DLQ', {
  retentionPeriod: cdk.Duration.days(14),
});

const queue = new sqs.Queue(this, 'Queue', {
  visibilityTimeout: cdk.Duration.seconds(300),
  encryption: sqs.QueueEncryption.KMS_MANAGED,
  deadLetterQueue: { queue: dlq, maxReceiveCount: 3 },
});

const topic = new sns.Topic(this, 'Topic', {
  masterKey: kms.Alias.fromAliasName(this, 'Key', 'alias/aws/sns'),
});

topic.addSubscription(new subscriptions.SqsSubscription(queue));

const consumer = new lambda.Function(this, 'Consumer', { /* ... */ });
consumer.addEventSource(new lambdaEventSources.SqsEventSource(queue, {
  batchSize: 10,
  reportBatchItemFailures: true,
}));
```

---

## Step Functions Workflow

```typescript
import * as sfn from 'aws-cdk-lib/aws-stepfunctions';
import * as tasks from 'aws-cdk-lib/aws-stepfunctions-tasks';

const processTask = new tasks.LambdaInvoke(this, 'Process', {
  lambdaFunction: processLambda,
  outputPath: '$.Payload',
});

const errorHandler = new sfn.Pass(this, 'HandleError', {
  result: sfn.Result.fromObject({ status: 'failed' }),
});

const definition = processTask
  .addCatch(errorHandler, { errors: ['States.ALL'] })
  .next(new sfn.Succeed(this, 'Done'));

const stateMachine = new sfn.StateMachine(this, 'StateMachine', {
  definition,
  tracingEnabled: true,
  logs: {
    destination: new logs.LogGroup(this, 'SfnLogs'),
    level: sfn.LogLevel.ALL,
  },
});
```

---

## Security & IAM Patterns

### Use grant* methods (preferred)
```typescript
bucket.grantRead(lambdaFn);                     // S3 read
table.grantReadWriteData(lambdaFn);             // DynamoDB RW
queue.grantSendMessages(producerFn);            // SQS send
topic.grantPublish(publisherFn);                // SNS publish
secret.grantRead(lambdaFn);                     // Secrets Manager
```

### Scoped inline policy (when grant* is insufficient)
```typescript
lambdaFn.addToRolePolicy(new iam.PolicyStatement({
  actions: ['ssm:GetParameter'],
  resources: [`arn:aws:ssm:${this.region}:${this.account}:parameter/myapp/*`],
  conditions: { StringEquals: { 'aws:RequestedRegion': this.region } },
}));
```

### Suppress CDK Nag rules (with justification)
```typescript
import { NagSuppressions } from 'cdk-nag';

NagSuppressions.addResourceSuppressions(lambdaFn, [
  {
    id: 'AwsSolutions-IAM4',
    reason: 'AWSLambdaBasicExecutionRole is acceptable for this read-only function',
  },
]);
```
