---
published: true
title: 'Getting started with AWS serverless: Deploy a frontend!'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/learn-serverless/front/assets/cover.png
description: 'Learn how to deploy a frontend on AWS, and how to make it interact with your serverless backend!'
tags: serverless, AWS, javascript, tutorial
series: 'Learn serverless on AWS step-by-step'
canonical_url:
---

## TL;DR

In this series, I try to explain the basics of serverless on AWS, to enable you to build your own serverless applications. During the last 11 articles, we tackled together a lot of "backend" services, creating small apps and interacting with them with HTTP requests. Today, let's start to close the loop and create a frontend, deployed on AWS, and able to interact with our backend services.

**What will we do today?**

- Create a simple monorepo containing a frontend and a backend
- Create a minimalistic backend
- Create a frontend interacting with it, and deploy it on AWS!

_All the code of this article is available on [this repository][repository]._

⬇️ I post serverless content very regularly, if you want more ⬇️

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

_**Quick announcement:** I also work on a library called [🛡 sls-mentor 🛡][sls-mentor]. It is a compilation of 30 serverless best-practices, that are automatically checked on your AWS serverless projects (no matter the framework). It is free and open source, feel free to check it out!_

{% cta https://github.com/sls-mentor/sls-mentor %} Find sls-mentor on Github ⭐️ {% endcta %}

## How to bring frontend and backend together: the monorepo

Since the beginning of this series, I made the choice to create my backend apps using TypeScript. One of the underlying reasons was to be able to share code between my frontend and my backend, because the main frontend frameworks: react, next, vue, etc... are all compatible with TypeScript. This will pay off today, because we will be able to create a monorepo containing both our frontend and our backend, plus the code able to deploy the frontend on AWS.

The architecture of the monorepo will look like this:

![Monorepo architecture](./assets/architecture.png 'Monorepo architecture')

It will contain three packages:

- Backend: deploy an API, some lambdas, and a database
- Frontend: a simple react app, built using vite. The frontend will rely on the API url defined in the backend package
- Frontend deploy: a simple package that will deploy the built code of the frontend on AWS, using S3 and CloudFront

Everything will interact together thanks to nx, a tool that allows to have multiple package.json files in the same repository, and to run commands on all of them at once. It is a very powerful tool, and I strongly advise you to check it out!

## Build a monorepo containing a frontend and a serverless backend

### Create the monorepo

To get started, run the following command (be sure to use node 18 to follow this article)

```bash
npx create-nx-workspace@latest
```

Chose a name for your workspace and select default options for the rest. This command creates a monorepo. This monorepo is designed to contain packages into a `packages` folder. This is where we will create our backend, frontend and frontend-deploy packages.

### Create the backend package

To build our backend, let's use the AWS CDK, as always in this series. If you need a refresher, you can check the [other articles of my series][series]. Run these commands in your cli to create the backend package:

```bash
mkdir packages && cd packages
mkdir backend && cd backend
npx cdk init app --language typescript
```

This creates a new CDK project. The important files in this projects are: `bin/backend.ts` and `lib/backend-stack.ts`. The first one is the entry point of the CDK app where you can specify the region and account you want to deploy to. The second one contains the stack that will be deployed, here is where we will write our code.

Lets create a very simple app, a REST API with 2 routes: createUser and listUsers. It will be made of 2 lambdas, and a DynamoDB table and a REST API from API Gateway.

![Backend architecture](./assets/backend-architecture.png 'Backend architecture')

Here is the code of the stack:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { join } from 'path';

export class BackendStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Create the API
    const api = new cdk.aws_apigateway.RestApi(this, 'RestApi', {});

    // Create the /users route, add CORS to it to allow the front to call it
    const users = api.root.addResource('users');
    users.addCorsPreflight({
      allowOrigins: cdk.aws_apigateway.Cors.ALL_ORIGINS,
      allowMethods: cdk.aws_apigateway.Cors.ALL_METHODS,
      allowHeaders: cdk.aws_apigateway.Cors.DEFAULT_HEADERS,
    });

    // Create the DynamoDB table
    const table = new cdk.aws_dynamodb.Table(this, 'UsersTable', {
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

    // Create the listUsers lambda, link it to the API
    const listUsers = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'ListUsers', {
      entry: join(__dirname, 'functions', 'listUsers.ts'),
      handler: 'handler',
      environment: {
        TABLE_NAME: table.tableName,
      },
      bundling: {
        minify: true,
        externalModules: ['@aws-sdk/client-dynamodb'],
      },
      runtime: cdk.aws_lambda.Runtime.NODEJS_18_X,
    });
    table.grantReadData(listUsers);
    users.addMethod('GET', new cdk.aws_apigateway.LambdaIntegration(listUsers));

    // Create the createUser lambda, link it to the API
    const createUser = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'CreateUser', {
      entry: join(__dirname, 'functions', 'createUser.ts'),
      handler: 'handler',
      environment: {
        TABLE_NAME: table.tableName,
      },
      bundling: {
        minify: true,
        externalModules: ['@aws-sdk/client-dynamodb'],
      },
      runtime: cdk.aws_lambda.Runtime.NODEJS_18_X,
    });
    table.grantWriteData(createUser);
    users.addMethod('POST', new cdk.aws_apigateway.LambdaIntegration(createUser));
  }
}
```

Nothing really new in this code snippet if you already read the beginning of my series. We create a DynamoDB table, and two lambdas. The lambdas are linked to the API, and the DynamoDB table is linked to the lambdas.

**The only new thing is the CORS configuration of the /users route**. In my previous articles, I did not need it as I was using Postman to call my API. Now, I need to be able to call it from my frontend, so I need to allow CORS. To stay simple, I allow all origins, all methods and all headers. TO go further, you should only allow your frontend origin, and only the methods and headers you need.

Final step of the backend development, create the code of the 2 lambda functions. Create a `functions` folder in the backend package, and create the two following files:

```typescript
// createUser.ts
import { DynamoDBClient, PutItemCommand } from '@aws-sdk/client-dynamodb';

const client = new DynamoDBClient({});

export const handler = async (event: {
  body: string;
}): Promise<{ statusCode: number; body: string; headers?: Record<string, string> }> => {
  const tableName = process.env.TABLE_NAME;

  if (!tableName) {
    throw new Error('Missing TABLE_NAME');
  }

  const { email, firstName, lastName } = JSON.parse(event.body) as {
    email: string;
    firstName: string;
    lastName: string;
  };

  if (!email || !firstName || !lastName) {
    return {
      statusCode: 400,
      body: 'Missing parameters',
    };
  }

  await client.send(
    new PutItemCommand({
      TableName: tableName,
      Item: {
        PK: { S: 'USER' },
        SK: { S: email },
        firstName: { S: firstName },
        lastName: { S: lastName },
      },
    }),
  );

  return {
    statusCode: 200,
    body: 'User created',
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
    },
  };
};
```

```typescript
// listUsers.ts
import { DynamoDBClient, QueryCommand } from '@aws-sdk/client-dynamodb';

const client = new DynamoDBClient({});

export const handler = async (): Promise<{ statusCode: number; body: string; headers: Record<string, string> }> => {
  const tableName = process.env.TABLE_NAME;

  if (!tableName) {
    throw new Error('Missing TABLE_NAME');
  }

  const { Items } = await client.send(
    new QueryCommand({
      TableName: tableName,
      KeyConditions: {
        PK: {
          ComparisonOperator: 'EQ',
          AttributeValueList: [{ S: 'USER' }],
        },
      },
    }),
  );

  return {
    statusCode: 200,
    body: JSON.stringify(
      Items?.map(item => ({
        email: item.SK.S,
        firstName: item.firstName.S,
        lastName: item.lastName.S,
      })),
    ),
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
    },
  };
};
```

One more time, nothing new in these two handlers, except the CORS configuration. They only use the aws sdk to interact with DynamoDB, and return a 200 or 400 status code depending on the result of the request. The CORS headers in the returned object allow the frontend to call the API. Like in the CDK code, you should restrict the origin, methods and headers to the ones you need.

Time to deploy our backend! Run the following commands:

```bash
npm run cdk bootstrap
npm run cdk deploy
```

Take notes of the API url, we will need it when we will create the frontend.

### Create the frontend package

Time to create the frontend package! We will use vite as it is a very simple way to create a react app. Run the following commands:

```bash
cd packages
npm create vite
cd frontend
npm i
```

Chose react as framework, typescript as language, and default options for the rest. This creates a new react app in the `frontend` folder. First thing to do is to add the API url to a `.env` file at the root of `packages/frontend`

```bash
VITE_API_URL="THE API URL YOU NOTED EARLIER"
```

Now, let's delete every file inside the `src` folder, except `main.tsx`, `App.tsx` and `vite-env.ts`. We want to create an app as simple as possible to focus on the deployment. Here is the code of `App.tsx`:

```tsx
import { useEffect, useState } from 'react';

function App() {
  const [users, setUsers] = useState<{ firstName: string; lastName: string; email: string }[]>([]);
  const [firstName, setFirstName] = useState<string>('');
  const [lastName, setLastName] = useState<string>('');
  const [email, setEmail] = useState<string>('');

  const syncUsers = async () => {
    const res = await fetch(`${import.meta.env.VITE_API_URL}/users`);
    const body = (await res.json()) as { firstName: string; lastName: string; email: string }[];

    setUsers(body);
  };

  useEffect(() => {
    void syncUsers();
  }, []);

  const onFormSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    await fetch(`${import.meta.env.VITE_API_URL}/users`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ firstName, lastName, email }),
    });
    setUsers([...users, { firstName, lastName, email }]);
    await syncUsers();
  };

  return (
    <div>
      <div>
        <h1>Users</h1>
        <form onSubmit={onFormSubmit}>
          <input
            type="text"
            name="firstName"
            value={firstName}
            onChange={e => setFirstName(e.target.value)}
            placeholder="First Name"
          />
          <input
            type="text"
            name="lastName"
            value={lastName}
            onChange={e => setLastName(e.target.value)}
            placeholder="Last Name"
          />
          <input
            type="text"
            name="email"
            value={email}
            onChange={e => setEmail(e.target.value)}
            placeholder="Email Address"
          />
          <button type="submit">Submit</button>
        </form>
      </div>
      <table>
        <thead>
          <tr>
            <th>First Name</th>
            <th>Last Name</th>
            <th>Email Address</th>
          </tr>
        </thead>
        <tbody>
          {users.map(({ email, firstName, lastName }) => (
            <tr key={email}>
              <td>{firstName}</td>
              <td>{lastName}</td>
              <td>{email}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

export default App;
```

The App component does 2 things: it displays a form to create a new user, and it displays a table with all the users. I use `fetch` to interact with the API, and I use the API url from the `.env` file, by using `import.meta.env.VITE_API_URL`. This is a very simple app, but it is enough for our purpose.

Now, we can already test the frontend locally. Run `npm run dev` in your CLI and you will be able to interact from localhost with the AWS backend.

![local website](./assets/local-site.png 'local website')

It works! We can create users, and they are displayed in the table. No data is lost when we refresh the page, because the data is stored in the DynamoDB table on AWS!

Before leaving the frontend, build it using the `npm run build` command. We will need the built code (in the `dist` folder) to deploy it on AWS.

### Deploy the frontend

All this code is really cool, be how can you become the next Mark Zuckerberg if you can't deploy your app on the internet? Let's fix that!

One more time, we will use the AWS CDK to deploy our frontend. We will use S3 to store the built code, and CloudFront to serve it. To do this, we will create a second CDK project package, named `frontend-deploy`. Run the following commands:

```bash
cd packages
mkdir frontend-deploy && cd frontend-deploy
npx cdk init app --language typescript
```

Like for the backend, we have to modify the `lib/frontend-deploy-stack.ts` file to deploy resources to AWS. Here is the code:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { join } from 'path';

export class FrontendDeployStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Create a bucket to store the built code
    // This bucket is public
    // It contains a website index document to serve the index.html file
    const staticSiteBucket = new cdk.aws_s3.Bucket(this, 'StaticSiteBucket', {
      websiteIndexDocument: 'index.html',
      blockPublicAccess: {
        blockPublicAcls: false,
        blockPublicPolicy: false,
        ignorePublicAcls: false,
        restrictPublicBuckets: false,
      },
      publicReadAccess: true,
    });

    // Deploy the built code of packages/frontend
    // to the bucket automatically on every code change
    new cdk.aws_s3_deployment.BucketDeployment(this, 'DeployStaticSite', {
      sources: [cdk.aws_s3_deployment.Source.asset(join(__dirname, '../..', 'frontend', 'dist'))],
      destinationBucket: staticSiteBucket,
    });

    // Create a CloudFront distribution to serve the website
    new cdk.aws_cloudfront.Distribution(this, 'StaticSiteDistribution', {
      defaultBehavior: {
        origin: new cdk.aws_cloudfront_origins.S3Origin(staticSiteBucket),
        viewerProtocolPolicy: cdk.aws_cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
      },
    });
  }
}
```

In this code snippet, we create 3 resources:

- A bucket to store the built code of the frontend
- A bucket deployment to automatically deploy the built code to the bucket
- A CloudFront distribution to serve the website

I will not go to far into the details of these resources. I published a very detailed article about [deploying static website on AWS][article-frontend] a few months ago, it goes further, tackling additional topics like custom domains, SSL certificates, etc... If you want to go further, I strongly advise you to check it out!

Now, we can deploy our frontend! Run the following commands:

```bash
npm run cdk bootstrap
npm run cdk deploy
```

Time to test the full app! Go to the CloudFront distribution url, and you should see the same website as before, but deployed on AWS!

![deployed website](./assets/deployed-site.png 'deployed website')

Want to deploy the website on your own domain? Check out my [article][article-frontend] about it!

## Conclusion

_All the code of this article is available on [this repository][repository]._

### What could be done better?

If you read this article with attention, you probably noticed that I did not use any type safety in the frontend. My monorepo doesn't really share code between its packages. This is because I wanted to keep the code as simple as possible, to focus on the deployment.

Be sure that I will cover these topics in future articles! Type-safety and code sharing are the main reasons why I chose TypeScript for this series, and I will not let you down!

### Now it's your time to shine 🌟

This series will continue, but you are at a point where you can already build your own serverless apps! I encourage you to try to build your own apps, and to deploy them on AWS.

Try to implement some funny ideas, taking advantage of AWS services like SES or SNS to send emails or SMS, or using S3 to store files. You can also try to build a more complex app, with a frontend and a backend, and deploy it on AWS. The possibilities are endless!

### Let's connect!

I would really appreciate if you could react and share this article with your friends and colleagues. It will help me a lot to grow my audience. Also, don't forget to subscribe to be updated when the next article comes out!

I you want to stay in touch here is my [twitter account][twitter]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

[twitter]: https://twitter.com/PierreChollet22
[series]: https://dev.to/pchol22/series/22030
[article-frontend]: https://dev.to/slsbytheodo/easily-deploy-your-portfolio-website-with-aws-cdk-4l9b
[repository]: https://github.com/PChol22/learn-serverless-backendxfrontend
[sls-mentor]: https://www.sls-mentor.dev/
