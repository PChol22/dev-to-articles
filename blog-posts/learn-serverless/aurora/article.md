---
published: true
title: 'Getting started with AWS serverless - SQL with Aurora'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/learn-serverless/aurora/assets/cover.png
description: ''
tags: serverless, AWS, javascript, tutorial
series: 'Learn serverless on AWS step-by-step'
canonical_url:
---

## TL;DR

In this series, I try to explain the basics of serverless on AWS, to enable you to build your own serverless applications. With [last article][article-sqs], we discovered how to use SQS to buffer events triggering a Lambda function. Today, we will tackle SQL storage, using Aurora Serverless!

**What will we do today?**

- Create an Aurora Serverless SQL database
- Create Lambda functions interacting with the database
- _Bonus: Build a super simple migration system_

⬇️ I post serverless content very regularly, if you want more ⬇️

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

## SQL Serverless storage: HOW IS IT POSSIBLE???

I know, SQL engines aren't really known for its serverless capabilities, as they intrinsically require a server to run. AWS is no exception: their SQL service AWS RDS is managed, but requires the provisioning of a server, and comes with a fixed cost.

But don't leave yet, I have good news for you! AWS released Aurora Serverless, which is a MYSQL / Postgres compatible SQL engine, that scales automatically and is fully managed. It can even auto-pause when not in use: **some (not everyone) will say it fits the definition of serverless!**

## Provision an Aurora Serverless cluster with the AWS CDK

In this article, we will create a SQL database, containing a table `users`. Users will have an id, a firstName and a lastName. We will then create 2 Lambda functions: a AddUser function, that will create a new user in the database, and a GetUsers function, that will retrieve all users from the database. Both functions will be exposed through an API Gateway.

The architecture will look like this:

![Basic schema](./assets/basic-schema.png 'Basic architecture schema')

As usual, we will use the AWS CDK in this article, if you are not familiar with it, I suggest you to read the [previous articles of my series][series] to get started. After having created a new CDK project, we provision a new Aurora cluster.

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import path from 'path';

export class LearnServerlessStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const dbSecret = new cdk.aws_rds.DatabaseSecret(this, 'AuroraSecret', {
      username: 'admin',
    });

    const cluster = new cdk.aws_rds.ServerlessCluster(this, 'AuroraCluster', {
      engine: cdk.aws_rds.DatabaseClusterEngine.AURORA_MYSQL,
      credentials: cdk.aws_rds.Credentials.fromSecret(dbSecret),
      clusterIdentifier: 'my-aurora-cluster',
      defaultDatabaseName: 'my_database',
      enableDataApi: true,
      scaling: {
        autoPause: cdk.Duration.minutes(10),
        minCapacity: 2,
        maxCapacity: 16,
      },
    });
  }
}
```

In this code snippet, we mainly do two things:

- Create a secret to store the database credentials, this will be useful to connect to the database later from Lambda functions. Notice I didn't specify a password, it will be generated automatically by AWS (you don't want to store it in your code !!!).

- Create a ServerlessCluster, which is a Aurora Serverless construct.
  - We specify the engine (MYSQL, postgres is also available)
  - We specify the credentials, using the secret we created earlier
  - We specify a default database name: it will create a database in the cluster for us upon creation
  - **We enable the Data API**, very important feature of Aurora Serverless, it allows us to interact with the database using HTTP requests, which will greatly simplify the development of our Lambda functions.
  - Finally, we setup autoscaling: min and max capacity are measured in ACUs ([learn more][acus]), it will automatically scale up and down depending on the load. Auto-pause is the duration of inactivity before the cluster is paused, which will save us money (only storage is billed when paused).

We could also have added a lot of networking, advanced security features etc..., but the beauty of Aurora Serverless is that it works out of the box with minimal configuration!

## Interact with Aurora Serverless using Lambda functions

Now, let's create 2 Lambda functions that will interact with our new database. The first one will create a new user in the database, the second one will retrieve all users from the database. First, let's provision the Lambda functions using the AWS CDK.

```typescript
// ...previous code, still in the constructor

const api = new cdk.aws_apigateway.RestApi(this, 'api', {});

const addUser = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'addUser', {
  entry: path.join(__dirname, 'addUser', 'handler.ts'),
  handler: 'handler',
  environment: {
    CLUSTER_ARN: cluster.clusterArn,
    SECRET_ARN: cluster.secret?.secretArn ?? '',
  },
  timeout: cdk.Duration.seconds(30),
});

cluster.grantDataApiAccess(addUser);

const getUsers = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'getUsers', {
  entry: path.join(__dirname, 'getUsers', 'handler.ts'),
  handler: 'handler',
  environment: {
    CLUSTER_ARN: cluster.clusterArn,
    SECRET_ARN: cluster.secret?.secretArn ?? '',
  },
  timeout: cdk.Duration.seconds(30),
});

cluster.grantDataApiAccess(getUsers);

api.root.addResource('add-user').addMethod('POST', new cdk.aws_apigateway.LambdaIntegration(addUser));
api.root.addResource('get-users').addMethod('GET', new cdk.aws_apigateway.LambdaIntegration(getUsers));
```

Here, we create 2 Lambda functions, and plug them into an API. AddUsers will be triggered by a POST request, and GetUsers by a GET request. **Important:** We also grant the Lambda functions access to the Data API of the cluster, so they can interact with the database, and we pass the cluster ARN and secret ARN as environment variables, so we can use them in the Lambda functions code. I set a 30 seconds timeout, because the Data API can be slow to respond when the database is paused.

Now, the interesting part: the code of the Lambda functions. Let's start with the addUser function.

```typescript
import { ExecuteStatementCommand, RDSDataClient } from '@aws-sdk/client-rds-data';
import { v4 as uuid } from 'uuid';

const rdsDataClient = new RDSDataClient({});

export const handler = async ({ body }: { body: string }): Promise<{ statusCode: number; body: string }> => {
  const secretArn = process.env.SECRET_ARN;
  const resourceArn = process.env.CLUSTER_ARN;

  if (secretArn === undefined || resourceArn === undefined) {
    throw new Error('Missing environment variables');
  }

  const { firstName, lastName } = JSON.parse(body) as { firstName?: string; lastName?: string };

  if (firstName === undefined || lastName === undefined) {
    return {
      statusCode: 400,
      body: 'Missing firstName or lastName',
    };
  }

  const userId = uuid();

  await rdsDataClient.send(
    new ExecuteStatementCommand({
      secretArn,
      resourceArn,
      database: 'my_database',
      sql: sql: 'CREATE TABLE IF NOT EXISTS users (id VARCHAR(255) NOT NULL PRIMARY KEY, firstName VARCHAR(255) NOT NULL, lastName VARCHAR(255) NOT NULL); INSERT INTO users (id, firstName, lastName) VALUES (:id, :firstName, :lastName);',
      parameters: [
        { name: 'id', value: { stringValue: userId } },
        { name: 'firstName', value: { stringValue: firstName } },
        { name: 'lastName', value: { stringValue: lastName } },
      ],
    }),
  );

  return {
    statusCode: 200,
    body: JSON.stringify({
      userId,
    }),
  };
};
```

In this code snippet, we do basically 3 things:

- Retrieve the environment variables we passed earlier, and check that they are defined
- Parse the body of the request, and extract the firstName and lastName
- Execute two SQL commands into the database `my_database`: create a `users` table if it doesn't exist, and insert a new user into the table then.

_⚠️ Remember: we can execute a simple rdsDataClient command because we enabled the Data API on the cluster, otherwise it would have been much more complicated._

Now, let's look at the code of the getUsers function.

```typescript
import { ExecuteStatementCommand, RDSDataClient } from '@aws-sdk/client-rds-data';

const rdsDataClient = new RDSDataClient({});

export const handler = async (): Promise<{ statusCode: number; body: string }> => {
  const secretArn = process.env.SECRET_ARN;
  const resourceArn = process.env.CLUSTER_ARN;

  if (secretArn === undefined || resourceArn === undefined) {
    throw new Error('Missing environment variables');
  }

  const { records } = await rdsDataClient.send(
    new ExecuteStatementCommand({
      secretArn,
      resourceArn,
      database: 'my_database',
      sql: 'SELECT * FROM users;',
    }),
  );

  const users =
    records?.map(([{ stringValue: id }, { stringValue: firstName }, { stringValue: lastName }]) => ({
      id,
      firstName,
      lastName,
    })) ?? [];

  return {
    statusCode: 200,
    body: JSON.stringify(users, null, 2),
  };
};
```

In this handler, we do basically 3 things:

- Retrieve the environment variables we passed earlier, and check that they are defined
- Execute a SELECT statement into the `users` table of the database name `my_database`, using the RDS Data API.
- Format the result of the query, and return it as a JSON response.

We are done! Time to deploy and to test our API.

```bash
npm run cdk deploy
```

I do a simple POST request to add a user: ![add user](./assets/add-user.png 'Add user')

And a GET request to retrieve all the users: ![get users](./assets/get-users.png 'Get users')

**What could be improved?**

This code works, but it is not perfect. Here are some things that could be improved:

- **Queries and typing management:** we do pure-SQL here, but we could use an ORM like [TypeORM][typeorm] to have a better typing, and have precise schemas for our tables.
- **Migrations:** notice we try to create the table `users` if it doesn't exist, every time we add a new user. This is not optimal, what if we wanted to add a new column to the table ? We should use a migration system to manage the evolution of our database schema.

Today is not the day to cover ORMs like TypeORM, but we can have a look at migrations.

## Create a super-simple migration system with DynamoDB

What are migrations? Basically they are instructions that describe the evolution of the schema of a database. You want migrations to **be executed in order, and only once per migration**. Migrations can be the creation or deletion of a table, the creation or deletion of a column, the insertion of data, etc...

To simply tackle migrations, what we can do is store which migration was run in a DynamoDB table. We can then check which migrations were already run, and run the ones that were not. Let's create a new CDK stack to do that.

Once we are done, the final architecture will look like this:

![final architecture](./assets/advanced-schema.png 'Final architecture')

First, let's update our CDK code to create a DynamoDB table and a runMigrations Lambda function.

```typescript
const migrationsTable = new cdk.aws_dynamodb.Table(this, 'migrationsTable', {
  partitionKey: {
    name: 'PK',
    type: cdk.aws_dynamodb.AttributeType.STRING,
  },
  sortKey: {
    name: 'SK',
    type: cdk.aws_dynamodb.AttributeType.STRING,
  },
  billingMode: cdk.aws_dynamodb.BillingMode.PAY_PER_REQUEST,
});

const runMigrations = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'runMigrations', {
  entry: path.join(__dirname, 'runMigrations', 'handler.ts'),
  handler: 'handler',
  environment: {
    DYNAMODB_TABLE_NAME: migrationsTable.tableName,
    CLUSTER_ARN: cluster.clusterArn,
    SECRET_ARN: cluster.secret?.secretArn ?? '',
  },
  timeout: cdk.Duration.seconds(180),
});

migrationsTable.grantReadWriteData(runMigrations);
cluster.grantDataApiAccess(runMigrations);
```

We grant to the runMigrations Lambda full access to the DynamoDB table, and access to the RDS Data API.

Now the interesting part, the code of the runMigrations Lambda function.

```typescript
import { DynamoDBClient, GetItemCommand, PutItemCommand } from '@aws-sdk/client-dynamodb';
import { ExecuteStatementCommand, RDSDataClient } from '@aws-sdk/client-rds-data';

// Array describing the migrations we want to run
const migrations: { id: string; statement: string }[] = [
  {
    id: 'migration-1',
    statement:
      'CREATE TABLE IF NOT EXISTS users (id VARCHAR(255) NOT NULL PRIMARY KEY, firstName VARCHAR(255) NOT NULL, lastName VARCHAR(255) NOT NULL);',
  },
  {
    id: 'migration-2',
    statement: `INSERT INTO users (id, firstName, lastName) VALUES ('1', 'John', 'Doe');`,
  },
];

const rdsDataClient = new RDSDataClient({});
const dynamoDBClient = new DynamoDBClient({});

export const handler = async (): Promise<void> => {
  const secretArn = process.env.SECRET_ARN;
  const resourceArn = process.env.CLUSTER_ARN;
  const dynamoDBTableName = process.env.DYNAMODB_TABLE_NAME;

  if (secretArn === undefined || resourceArn === undefined || dynamoDBTableName === undefined) {
    throw new Error('Missing environment variables');
  }

  // Run migrations in order
  for (const { id, statement } of migrations) {
    // Check if migration has already been executed
    const { Item: migration } = await dynamoDBClient.send(
      new GetItemCommand({
        TableName: dynamoDBTableName,
        Key: {
          PK: { S: 'MIGRATION' },
          SK: { S: id },
        },
      }),
    );

    if (migration !== undefined) {
      continue;
    }

    // Execute migration
    await rdsDataClient.send(
      new ExecuteStatementCommand({
        secretArn,
        resourceArn,
        database: 'my_database',
        sql: statement,
      }),
    );

    // Mark migration as executed
    await dynamoDBClient.send(
      new PutItemCommand({
        TableName: dynamoDBTableName,
        Item: {
          PK: { S: 'MIGRATION' },
          SK: { S: id },
        },
      }),
    );

    console.log(`Migration ${id} executed successfully`);
  }
};
```

The code can seem complex, but it's actually pretty simple.

- First, we define **inside our code** a list of migrations we want to run. Each migration is described by an unique id and a SQL statement.
- Then, inside the handler:
  - We parse the environment variables we need.
  - We iterate **in order** over the migrations we want to run.
  - For each migration, we check if it was already run by checking if the migration is present in the DynamoDB table.
  - If the migration was not run, we execute it using the RDS Data API.
  - Finally, we mark the migration as executed by saving it in the DynamoDB table.

That's all! If you want, you can plug this lambda function to an API, or keep it manually triggered. You can also add more migrations to the list, and they will be executed in order.

## Conclusion

This article was a basic introduction to Aurora Serverless. We saw how to create a cluster, how to connect to it, and how to run very simple migrations. With this simple knowledge, you can already achieve so much, like creating multiple databases, executing complex requests, etc...

If you want to go further, I strongly advise you to check ORMs like [TypeORM][typeorm] or [Prisma][prisma]. They will help you a lot to interact with your database, create migrations, and they are compatible with Aurora Serverless.

I plan to continue this series of articles on a bi-monthly basis. I already covered the creation of simple lambda functions and REST APIs, as well as interacting with DynamoDB databases and S3 buckets. You can follow this progress on my [repository][repository]! I will cover new topics like front-end deployment, type safety, more advanced patterns, and more... If you have any suggestions, do not hesitate to contact me!

I would really appreciate if you could react and share this article with your friends and colleagues. It will help me a lot to grow my audience. Also, don't forget to subscribe to be updated when the next article comes out!

I you want to stay in touch here is my [twitter account][twitter]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

[article-sqs]: https://dev.to/kumo/learn-serverless-on-aws-step-by-step-sqs-26c8
[twitter]: https://twitter.com/PierreChollet22
[repository]: https://github.com/PChol22/learn-serverless
[series]: https://dev.to/pchol22/series/22030
[acus]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html
[typeorm]: https://typeorm.io/#/
[prisma]: https://www.prisma.io/
