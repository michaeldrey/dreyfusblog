---
layout: ../layouts/BlogPost.astro
title: Using Escape Hatches in AWS CDK
slug: cdk_escapeHatch
description: 'How to use escape hatches in AWS CDK '
tags:
  - technical
  - learning
  - musings
added: 2025-10-05T04:39:15.566Z
---

## What Are Escape Hatches?

Escape hatches are CDK's safety valve for accessing lower-level CloudFormation properties that aren't exposed through the L2 construct APIs. When you need to set a specific CloudFormation property that the CDK construct doesn't support, escape hatches let you drop down to the CloudFormation level without abandoning the CDK abstraction entirely.

## When to Use Them

You'll reach for escape hatches when:

* A CloudFormation property exists but the L2 construct doesn't expose it
* You need to set properties on resources that CDK creates implicitly
* AWS releases new CloudFormation features before CDK supports them
* You need to remove default properties CDK adds automatically

## The Three Primary Methods

### addPropertyOverride

This is your most common escape hatch. It lets you set or override specific CloudFormation properties.

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as cdk from 'aws-cdk-lib';

const instance = new ec2.Instance(this, 'Instance', {
  vpc,
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
  machineImage: ec2.MachineImage.latestAmazonLinux2(),
});

// Access the underlying CFN resource and add a property
const cfnInstance = instance.node.defaultChild as ec2.CfnInstance;
cfnInstance.addPropertyOverride('CreditSpecification.CPUCredits', 'unlimited');
```

The method uses dot notation to traverse the CloudFormation property structure. Each dot represents a nested level in the CloudFormation resource definition.

### addOverride

This is the more powerful version that lets you override any part of the CloudFormation template, including properties outside the resource's Properties section.

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';

const bucket = new s3.Bucket(this, 'MyBucket', {
  versioned: true,
});

const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;

// Override at the Properties level
cfnBucket.addOverride('Properties.AnalyticsConfigurations', [
  {
    Id: 'analytics-config',
    StorageClassAnalysis: {
      DataExport: {
        OutputSchemaVersion: 'V_1',
        Destination: {
          S3BucketDestination: {
            Format: 'CSV',
            BucketArn: 'arn:aws:s3:::analytics-bucket',
          },
        },
      },
    },
  },
]);

// Override metadata
cfnBucket.addOverride('Metadata.CustomKey', 'CustomValue');
```

### addDeletionOverride

When CDK adds properties you don't want, use this to remove them.

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';

const fn = new lambda.Function(this, 'Function', {
  runtime: lambda.Runtime.NODEJS_18_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
});

const cfnFunction = fn.node.defaultChild as lambda.CfnFunction;

// Remove a property CDK added
cfnFunction.addDeletionOverride('Properties.ReservedConcurrentExecutions');
```

## Working with L1 Constructs Directly

Sometimes it's cleaner to use L1 (CloudFormation) constructs directly rather than fighting with L2 overrides.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

// L1 construct gives you direct CloudFormation control
const cfnVpc = new ec2.CfnVPC(this, 'VPC', {
  cidrBlock: '10.0.0.0/16',
  enableDnsHostnames: true,
  enableDnsSupport: true,
  instanceTenancy: 'default',
  // Direct access to all CloudFormation properties
  ipv4IpamPoolId: 'ipam-pool-123',
  ipv4NetmaskLength: 16,
});
```

The tradeoff is that L1 constructs require more boilerplate and don't provide CDK's helpful defaults or automatic resource creation.

## Using cfnOptions

Every CDK construct has a cfnOptions property that lets you configure CloudFormation-specific behaviors.

```typescript
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

const table = new dynamodb.Table(this, 'Table', {
  partitionKey: { name: 'id', type: dynamodb.AttributeType.STRING },
});

const cfnTable = table.node.defaultChild as dynamodb.CfnTable;

// Set CloudFormation options
cfnTable.cfnOptions.condition = myCondition;
cfnTable.cfnOptions.metadata = {
  Comment: 'This table is managed by CDK',
};
cfnTable.cfnOptions.updatePolicy = {
  enableVersionUpgrade: true,
};
```

## Real-World Example: RDS Custom Parameters

Here's a practical example where you need escape hatches because the L2 construct doesn't expose certain RDS features.

```typescript
import * as rds from 'aws-cdk-lib/aws-rds';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

const database = new rds.DatabaseInstance(this, 'Database', {
  engine: rds.DatabaseInstanceEngine.postgres({
    version: rds.PostgresEngineVersion.VER_15_3,
  }),
  vpc,
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
});

const cfnDatabase = database.node.defaultChild as rds.CfnDBInstance;

// Enable Performance Insights with custom retention
cfnDatabase.addPropertyOverride('EnablePerformanceInsights', true);
cfnDatabase.addPropertyOverride('PerformanceInsightsRetentionPeriod', 731);
cfnDatabase.addPropertyOverride('PerformanceInsightsKMSKeyId', kmsKey.keyArn);

// Set custom CloudWatch log exports
cfnDatabase.addPropertyOverride('EnableCloudwatchLogsExports', [
  'postgresql',
  'upgrade',
]);

// Configure deletion protection
cfnDatabase.addPropertyOverride('DeletionProtection', true);
```

## Accessing Nested Resources

CDK creates some resources implicitly. You can access these through the construct tree.

```typescript
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecs_patterns from 'aws-cdk-lib/aws-ecs-patterns';

const service = new ecs_patterns.ApplicationLoadBalancedFargateService(this, 'Service', {
  taskImageOptions: {
    image: ecs.ContainerImage.fromRegistry('amazon/amazon-ecs-sample'),
  },
});

// Access the auto-generated target group
const targetGroup = service.targetGroup.node.defaultChild as elbv2.CfnTargetGroup;
targetGroup.addPropertyOverride('HealthCheckIntervalSeconds', 10);
targetGroup.addPropertyOverride('HealthyThresholdCount', 2);
targetGroup.addPropertyOverride('UnhealthyThresholdCount', 3);
```

## Combining Multiple Approaches

You can mix L2 constructs with escape hatches for a pragmatic approach.

```typescript
import * as ecs from 'aws-cdk-lib/aws-ecs';

// Use L2 for the nice API
const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
  memoryLimitMiB: 512,
  cpu: 256,
});

const container = taskDef.addContainer('app', {
  image: ecs.ContainerImage.fromRegistry('myapp'),
  logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'myapp' }),
});

// Drop to L1 for properties not exposed
const cfnTaskDef = taskDef.node.defaultChild as ecs.CfnTaskDefinition;
cfnTaskDef.addPropertyOverride('ContainerDefinitions.0.FirelensConfiguration', {
  Type: 'fluentbit',
  Options: {
    'enable-ecs-log-metadata': 'true',
  },
});

// Configure EFS volume options
cfnTaskDef.addPropertyOverride('Volumes.0.EFSVolumeConfiguration.TransitEncryption', 'ENABLED');
```

## Best Practices

Always cast to the specific CFN type rather than using generic types. This gives you type safety and autocomplete.

```typescript
// Good
const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;

// Avoid
const cfnBucket = bucket.node.defaultChild as cdk.CfnResource;
```

Check the CloudFormation documentation for the exact property names and structure. CDK uses the CloudFormation property names exactly as they appear in the AWS CloudFormation documentation.

Consider filing a GitHub issue when you use escape hatches for commonly-used properties. The CDK team often adds support for frequently-requested features.

Test your overrides carefully. Since you're bypassing CDK's type system, mistakes won't be caught until deployment.

## Debugging Tip

To see the actual CloudFormation template CDK generates, use:

cdk synth

This outputs the complete CloudFormation JSON, which helps you verify your escape hatches are working as intended. You can also inspect specific constructs programmatically:

console.log(JSON.stringify(stack.resolve((cfnBucket as any).\_toCloudFormation()), null, 2));

This technique is invaluable when you're not sure if your override is structured correctly.
