---
published: true
title: 'Learn serverless on AWS step-by-step - Schedule tasks with EventBridge Scheduler'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/learn-serverless/scheduler/assets/cover.png
description: 'Discover how to use AWS EventBridge Scheduler to schedule Lambda function tasks in your serverless applications. Step-by-step tutorial with code examples.'
tags: serverless, AWS, javascript, tutorial
series: 'Learn serverless on AWS step-by-step'
canonical_url:
---

## TL;DR

In this series, I try to explain the basics of serverless on AWS, to enable you to build your own serverless applications. With [last article][article-destinations], we discovered how to Lambda function destinations to avoid losing data when an asynchronous Lambda function fails. In this article, we will discover how to schedule tasks with EventBridge Scheduler.

**What will we do today?**

- Create a small memo application, where we can create a memo, and execute it at a specific date and time
- Create a Lambda function that creates a memo
- Create a Lambda function that is triggered at a specific date and time, and executes the memo

The architecture of our application will look like this:

![architecture](./assets/architecture.png 'Architecture of our application')

⬇️ I post serverless content very regularly, if you want more ⬇️

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

_**Quick announcement:** I also work on a library called [🛡 sls-mentor 🛡][sls-mentor]. It is a compilation of 40 serverless best-practices, that are automatically checked on your AWS serverless projects (no matter the framework). It is free and open source, feel free to check it out!_

{% cta https://github.com/sls-mentor/sls-mentor %} Find sls-mentor on Github ⭐️ {% endcta %}

## How to schedule tasks with AWS EventBridge Scheduler?

AWS EventBridge Scheduler is a service that allows you to schedule tasks in the future. It can be compared to AWS EventBridge Rules: while rules allow you to trigger tasks based on rates or cron expressions, scheduler allows you to trigger tasks at a specific date and time (they also support cron expressions and rates btw...).

Using the AWS EventBridge Scheduler programmatically to trigger a Lambda function has an easy and a not-so-easy step:

- 😁 The easy step is to specify a target, this can easily be done using the ARN of the Lambda function you want to trigger
- 🥵 The not-so-easy step is to give the permission to the scheduler to invoke your Lambda function. This is done by:

  - Creating a role with a `scheduler.amazonaws.com` service principal (assumed by the scheduler)
  - Giving the permission to the scheduler to invoke your Lambda function `lambda:InvokeFunction`
  - Giving the permission to your input Lambda function (`addMemo`) to pass the role to the scheduler `iam:PassRole`

This can be summarized in the following diagram:

![scheduler-role](./assets/execution-role.png 'Creation of an InvokeExecuteMemoRole')

Now, let's see how this works in practice, with a real code example!

## Create a small memo application using AWS EventBridge Scheduler

To create our app, we will use the AWS CDK. If you are not familiar with it, I invite you to read the [first article of this series][article-lambda] where I explain properly how to setup a CDK project.

```bash
npx cdk init app --language typescript
npm i @aws-sdk/client-scheduler
npm i uuid # always useful to generate unique ids
npm i -D esbuild # Needed to bundle our Lambdas!
```

First, let's create the Infrastructure as Code (IAC) of the application. This is done by updating the CDK stack:

```typescript
// stack.ts - Infrastructure as code
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';

import * as cdk from 'aws-cdk-lib';
import { join } from 'path';

export class Part17SchedulerStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // Lambda function triggered by scheduler
    const executeMemo = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'ExecuteMemo', {
      entry: join(__dirname, 'executeMemo.ts'),
      handler: 'handler',
      runtime: cdk.aws_lambda.Runtime.NODEJS_18_X,
      bundling: {
        externalModules: ['@aws-sdk'],
      },
    });

    // Create role for scheduler to invoke executeMemo
    const invokeExecuteMemoRole = new cdk.aws_iam.Role(this, 'InvokeMemoRole', {
      assumedBy: new cdk.aws_iam.ServicePrincipal('scheduler.amazonaws.com'),
    });
    invokeExecuteMemoRole.addToPolicy(
      new cdk.aws_iam.PolicyStatement({
        actions: ['lambda:InvokeFunction'],
        resources: [executeMemo.functionArn],
      }),
    );

    // Lambda function that schedules executeMemo
    const addMemo = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'AddMemo', {
      entry: join(__dirname, 'addMemo.ts'),
      handler: 'handler',
      runtime: cdk.aws_lambda.Runtime.NODEJS_18_X,
      bundling: {
        externalModules: ['@aws-sdk'],
      },
      environment: {
        SCHEDULE_TARGET_ARN: executeMemo.functionArn,
        SCHEDULE_ROLE_ARN: invokeExecuteMemoRole.roleArn,
      },
    });

    // Allow addMemo to create a scheduler
    addMemo.addToRolePolicy(
      new cdk.aws_iam.PolicyStatement({
        actions: ['scheduler:CreateSchedule'],
        resources: ['*'],
      }),
    );

    // Allow addMemo to pass the invokeExecuteMemoRole to the scheduler
    addMemo.addToRolePolicy(
      new cdk.aws_iam.PolicyStatement({
        actions: ['iam:PassRole'],
        resources: [invokeExecuteMemoRole.roleArn],
      }),
    );

    // Trigger addMemo via API Gateway
    const api = new cdk.aws_apigateway.RestApi(this, 'Api', {
      restApiName: 'Part17Service',
    });
    api.root.addResource('addMemo').addMethod('POST', new cdk.aws_apigateway.LambdaIntegration(addMemo));
  }
}
```

What is happening here?

- 1️⃣ We create the executeMemo Lambda function, that will be triggered by the scheduler
- 2️⃣ We create a role that will be assumed by the scheduler, and that will allow it to invoke the executeMemo Lambda function (see intro diagram)
- 3️⃣ We create the addMemo Lambda function, that will create a scheduler
- 4️⃣ We allow the addMemo Lambda function to create a scheduler by adding the `scheduler:CreateSchedule` permission and the `iam:PassRole` (see intro diagram) permission to the addMemo role
- 5️⃣ We create an API Gateway endpoint that will trigger the addMemo Lambda function, and a POST method to trigger it

Now, let's create the Lambda functions that will be triggered by the API Gateway endpoint. First, let's create the `addMemo` Lambda function, that will create a schedule:

```typescript
// addMemo.ts - Lambda function that creates a scheduler
import {
  ActionAfterCompletion,
  CreateScheduleCommand,
  FlexibleTimeWindowMode,
  SchedulerClient,
} from '@aws-sdk/client-scheduler';

import { v4 as uuidv4 } from 'uuid';

const client = new SchedulerClient({});
const scheduleTargetArn = process.env.SCHEDULE_TARGET_ARN as string;
const scheduleRoleArn = process.env.SCHEDULE_ROLE_ARN as string;

if (scheduleTargetArn === undefined || scheduleRoleArn === undefined) {
  throw new Error('Missing environment variables');
}

export const handler = async ({
  body,
}: {
  body: string;
}): Promise<{
  statusCode: number;
  body: string;
}> => {
  const {
    memo,
    date,
    time,
    timezone = 'Europe/Paris',
  } = JSON.parse(body) as { memo?: string; date?: string; timezone?: string; time?: string };

  if (memo === undefined || date === undefined) {
    return {
      statusCode: 400,
      body: 'Bad Request',
    };
  }

  await client.send(
    new CreateScheduleCommand({
      Name: uuidv4(),
      Target: {
        Arn: scheduleTargetArn,
        RoleArn: scheduleRoleArn,
        Input: JSON.stringify({ memo }),
      },
      ScheduleExpressionTimezone: timezone,
      ScheduleExpression: `at(${date}T${time})`,
      FlexibleTimeWindow: {
        Mode: FlexibleTimeWindowMode.OFF,
      },
      ActionAfterCompletion: ActionAfterCompletion.DELETE,
    }),
  );

  return {
    statusCode: 200,
    body: 'Memo scheduled',
  };
};
```

What is happening here?

- 1️⃣ We setup a client and parse environment variables (see IAC where we set them)
- 2️⃣ We parse the body of the request, and extract the memo, date, time and timezone (default to Europe/Paris (where I live 😅))
- 3️⃣ We create a schedule using the AWS SDK for Javascript
  - 🅰️ We set the target, using the `scheduleTargetArn` and `scheduleRoleArn` environment variables
  - 🅱️ We set the schedule expression, using the date and time provided in the request, along with the timezone, and we set the schedule to be automatically deleted after execution

Finally, let's create the `executeMemo` Lambda function, that will be triggered by the scheduler. This function will simply log the memo to the console:

```typescript
// executeMemo.ts - Lambda function that executes a memo
export const handler = async ({ memo }: { memo: string }): Promise<void> => {
  console.log(memo);

  return Promise.resolve();
};
```

Very easy! Notice that the lambda input is the same as the `input` field specified in the `CreateScheduleCommand` of the `addMemo` Lambda function.

## Test our app

We are done! Time to deploy and to test our API route `/addMemo`

```bash
npm run cdk bootstrap
npm run cdk deploy
```

![add memo](./assets/add-memo.png 'Add memo Postman request')

I live in Paris 🇫🇷, so I used my default `Europe/Paris` timezone, but you can specify the timezone of your choice. I specified a time of 22:37, and it is 22:36, if I head to Cloudwatch, I should see my executeMemo Lambda function being triggered in 1 minute.

![cloudwatch](./assets/execute-memo.png 'Execute Memo execution in Cloudwatch')

It worked! I can also see the payload of the event, and it contains the memo I created.

### Going further

What could be improved in this app?

- We could go further and add a `/getMemos` route, that would return all the memos that are not executed yet
- We could also add a `/executeMemo` route, that would execute a memo immediately, and cancel the corresponding schedule
- As always, implementing authentication would be a good idea 😅

## Conclusion

This article was a basic introduction to AWS EventBridge Scheduler. We discovered how to create a scheduled task, and how to execute it. We also discovered how to use the AWS CDK to provision our infrastructure. I hope you enjoyed this article, and that you learned something new!

I plan to continue this [series of articles][series] on a bi-monthly basis. You can follow this progress on my [repository][repository]! I will cover new topics in the future, if you have any suggestions, do not hesitate to contact me!

I would really appreciate if you could react and share this article with your friends and colleagues. It will help me a lot to grow my audience. Also, don't forget to subscribe to be updated when the next article comes out!

I you want to stay in touch here is my [twitter account][twitter]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

[twitter]: https://twitter.com/PierreChollet22
[repository]: https://github.com/PChol22/learn-serverless
[sls-mentor]: https://github.com/sls-mentor/sls-mentor
[article-lambda]: https://dev.to/slsbytheodo/dont-miss-on-the-cloud-revolution-learn-serverless-on-aws-the-right-way-1kac
[article-destinations]: https://dev.to/slsbytheodo/learn-serverless-on-aws-step-by-step-lambda-destinations-f5b
[series]: https://dev.to/pchol22/series/22030
