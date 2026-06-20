---
name: aws-cloud-patterns
description: Serverless AWS patterns — Lambda handlers, DynamoDB single-table design, S3 event processing, and CDK infrastructure
---

# AWS cloud patterns

## A Lambda handler

```typescript
import { APIGatewayProxyHandlerV2 } from "aws-lambda";
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, GetCommand } from "@aws-sdk/lib-dynamodb";

const client = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export const handler: APIGatewayProxyHandlerV2 = async (event) => {
  const id = event.pathParameters?.id;
  if (!id) {
    return { statusCode: 400, body: JSON.stringify({ error: "Missing id" }) };
  }

  const result = await client.send(
    new GetCommand({ TableName: process.env.TABLE_NAME!, Key: { pk: id } })
  );

  if (!result.Item) {
    return { statusCode: 404, body: JSON.stringify({ error: "Not found" }) };
  }

  return {
    statusCode: 200,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(result.Item),
  };
};
```

Create SDK clients outside the handler so connections survive and get reused between invocations.

## DynamoDB single-table design

```typescript
interface OrderItem {
  pk: string;          // USER#<userId>
  sk: string;          // ORDER#<orderId>
  gsi1pk: string;      // ORDER#<orderId>
  gsi1sk: string;      // ITEM#<itemId>
  entityType: string;  // "Order" | "OrderItem"
  data: Record<string, any>;
  ttl?: number;
}

const params = {
  TableName: "AppTable",
  KeyConditionExpression: "pk = :pk AND begins_with(sk, :prefix)",
  ExpressionAttributeValues: {
    ":pk": `USER#${userId}`,
    ":prefix": "ORDER#",
  },
};
```

Work out your access patterns first and only then model the keys; reach for GSIs to serve the alternative queries.

## Infrastructure in CDK

```typescript
import * as cdk from "aws-cdk-lib";
import { Construct } from "constructs";
import * as lambda from "aws-cdk-lib/aws-lambda-nodejs";
import * as dynamodb from "aws-cdk-lib/aws-dynamodb";
import * as apigateway from "aws-cdk-lib/aws-apigatewayv2";

export class ApiStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const table = new dynamodb.Table(this, "AppTable", {
      partitionKey: { name: "pk", type: dynamodb.AttributeType.STRING },
      sortKey: { name: "sk", type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      pointInTimeRecovery: true,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    const fn = new lambda.NodejsFunction(this, "ApiHandler", {
      entry: "src/handler.ts",
      runtime: cdk.aws_lambda.Runtime.NODEJS_22_X,
      architecture: cdk.aws_lambda.Architecture.ARM_64,
      memorySize: 256,
      timeout: cdk.Duration.seconds(10),
      environment: { TABLE_NAME: table.tableName },
    });

    table.grantReadWriteData(fn);
  }
}
```

## Processing S3 events

```typescript
import { S3Event } from "aws-lambda";
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";

const s3 = new S3Client({});

export async function handler(event: S3Event) {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, " "));

    const obj = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
    const body = await obj.Body?.transformToString();

    await processFile(key, body);
  }
}
```

## What to avoid

- Hardcoding AWS credentials instead of leaning on IAM roles
- Leaving Lambda timeout and memory at careless defaults
- Scanning DynamoDB wholesale rather than querying on key conditions
- One Lambda per CRUD verb when grouping by domain would be cleaner
- No CloudWatch alarms for error rates or throttling
- DynamoDB tables without point-in-time recovery turned on

## Before you ship

- [ ] SDK clients are created outside the handler
- [ ] IAM roles follow least privilege
- [ ] DynamoDB access patterns were designed before the table schema
- [ ] Lambda runs on ARM64 for the cost saving
- [ ] S3 buckets have versioning and lifecycle policies
- [ ] CloudWatch alarms cover Lambda errors, duration, and throttles
- [ ] Infrastructure is defined as code (CDK or Terraform)
- [ ] Secrets live in Parameter Store or Secrets Manager
