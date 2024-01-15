---
published: true
title: 'Getting started with AWS serverless: Lambda Destinations'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/learn-serverless/lambda-destinations/assets/cover.png
description: 'Lambda Destinations allow you to route the asynchronous execution results of your Lambdas to other AWS services. In this article, discover how to use them to be notified when your Lambda function fails'
tags: serverless, AWS, javascript, tutorial
canonical_url:
series: 'Learn serverless on AWS step-by-step'
---

## TL;DR

When working with asynchronous Lambda functions, it can be hard to know whether a function invocation succeeded or failed. Furthermore, it can be important to retain a trace of the failed invocation, to be able to debug it or retry it later. Don't worry, AWS has a solution for that: **Lambda destinations**!

I you want to stay in touch here is my [twitter account][twitter]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

### What are we going to build?

Today, we will take advantage of Lambda destinations to be notified when an asynchronous Lambda function fails, and store the failed invocation to retry it later. To do so, we will:

- Build a Lambda function that fails randomly, triggered every minute by a CRON
- Plug a SNS topic to the failure destination of the Lambda function
- Use the SNS topic to send an email, and store the event in a SQS queue

The architecture will look like this:

![Architecture](./assets/architecture.png 'Architecture')

_**Quick announcement:** I also work on a library called [🛡 sls-mentor 🛡][sls-mentor]. It is a compilation of 30 serverless best-practices, that are automatically checked on your AWS serverless projects (no matter the framework). It is free and open source, feel free to check it out!_

{% cta https://github.com/sls-mentor/sls-mentor %} Find sls-mentor on Github ⭐️ {% endcta %}

## What are Lambda destinations?

The answer to this question is really simple: there are two types of Lambda destinations:

- **Success destinations**: they are triggered when a asynchronous Lambda function succeeds
- **Failure destinations**: they are triggered when a asynchronous Lambda function fails

Each time, the destination is triggered with information about the Lambda function invocation. Among this information, you can find the request payload, the response payload, the error message, the error type, etc...

Destinations can be multiple AWS services, such as SNS, SQS, EventBridge, etc... In this article, we will use SNS as a destination to be able to dispatch the failed invocations to an email address and a SQS queue.

## Add a failure destination to a Lambda function

To build this tiny project, I will use (as in the previous articles) the AWS CDK combined with Typescript. If you are not familiar with the CDK, I invite you to read the [first article of this series][article-lambda], where I explain how to set up a CDK project.

The CDK code for this architecture looks like this:

```typescript
// lambda-destinations-stack.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { join } from 'path';

const DEVELOPERS_EMAILS = ['pchol.pro@gmail.com'];

export class LambdaDestinationsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Create a CRON trigger, that will trigger the Lambda function every minute
    const cronTrigger = new cdk.aws_events.Rule(this, 'CronTrigger', {
      schedule: cdk.aws_events.Schedule.expression('rate(1 minute)'),
    });

    // Create a SNS topic that will be used as a failure destination
    const onFailureTopic = new cdk.aws_sns.Topic(this, 'OnFailureTopic');

    // Create the Lambda function that will fail randomly
    const lambdaFunction = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'LambdaFunction', {
      entry: join(__dirname, 'lambda.ts'),
      handler: 'handler',
      runtime: cdk.aws_lambda.Runtime.NODEJS_18_X,
      // Add the failure destination to the Lambda function
      onFailure: new cdk.aws_lambda_destinations.SnsDestination(onFailureTopic),
    });

    // Add the Lambda function as a target of the CRON trigger
    cronTrigger.addTarget(
      new cdk.aws_events_targets.LambdaFunction(lambdaFunction, {
        event: cdk.aws_events.RuleTargetInput.fromObject({
          // Add some data to the event payload, you will see that we will be able to retrieve it later
          veryImportantData: 'this should not be lost !!!',
        }),
        retryAttempts: 1, // 1 retry attempt
      }),
    );

    // Create a SQS queue that will be used to store the failed events
    const failedEventsQueue = new cdk.aws_sqs.Queue(this, 'FailedEventsQueue');

    // Redirect the failed events to the SQS queue
    onFailureTopic.addSubscription(new cdk.aws_sns_subscriptions.SqsSubscription(failedEventsQueue));

    // Also send an email to the developers with the same information to notify them
    DEVELOPERS_EMAILS.forEach(email => {
      onFailureTopic.addSubscription(new cdk.aws_sns_subscriptions.EmailSubscription(email));
    });
  }
}
```

There is not so much code so I wrote it in one block. Let's go through it step by step:

- First, we create a CRON trigger that will trigger the Lambda function every minute
- Then, we create a SNS topic that will be used as a failure destination. We will later add subscriptions to this topic so that it can dispatch the failed events.
- Then, we create the Lambda function that will fail randomly. We add the SNS topic as a failure destination to the Lambda function.
- We add the Lambda function as a target of the CRON trigger. We also add some data to the event payload, to see later that we can retrieve it. We set the number of retries to 1. This means that we will only be notified if the Lambda function fails twice in a row.
- We create a SQS queue that will be used to store the failed events.
- Finally, we add subscriptions to the SNS topic. We add a subscription to the SQS queue, and a subscription to each developer email address.

Final step, let's write the code for the handler of the Lambda function:

```typescript
// lambda.ts
export const handler = async (): Promise<void> => {
  // 80% chance of going wrong
  // 64% counting the retry
  const success = Math.random() > 0.8;

  if (!success) {
    throw new Error('Something went wrong!');
  }

  return Promise.resolve();
};
```

Nothing very interesting here. We throw an error 80% of the time. Combined with the 1 retry attempt, it means that the Lambda function will trigger the failure destination 64% of the time. **It does not trigger the failure destination when there are still retries left**

We are done with the code 🚀! Let's deploy it to AWS!

```bash
npm i -D esbuild # Esbuild is required because we use the NodejsFunction construct
npm run cdk bootstrap # Only the first time you deploy to an AWS account
npm run cdk deploy
```

## Test our Lambda Destinations

Now that the application is deployed, we don't have much to do since it is automatically triggered every minute. Wait a little bit and you will eventually receive an email like this one:

![Email](./assets/email.png 'Email')

In this email, we can see the error message, and most importantly, the request event payload: the data wasn't lost!

If you login to the AWS console and check the SQS queue, it should contain a message (or more if you waited a little bit) that looks similar to what you received by email:

![SQS](./assets/sqs.png 'SQS')

You will later be able to re-use these messages, for example to retry the failed invocations once the issue is fixed.

## What could we do next?

In this article, I only showed you the failure destinations, you can also use success destinations. They are useful to be notified when a Lambda function succeeds, or to create a pipeline of Lambda functions.

It is also possible to be smarter with our handling of the messages stored in the SQS queue. With a bit of development, it should be possible to re-trigger the failed invocations automatically, without necessarily having to do it manually.

Other services like EventBridge can also be used as destinations, feel free to try them too!

## Failure destinations best practices

It is considered best-practice to use failure destinations on all your asynchronous Lambda functions (for data retention and notification purposes). Remember [sls-mentor][sls-mentor]? It automatically checks that you use failure destinations on all your asynchronous Lambda functions, so that you know where to put your efforts first.

{% cta https://github.com/sls-mentor/sls-mentor %} Find sls-mentor on Github ⭐️ {% endcta %}

## Let's connect!

I would really appreciate if you could react and share this article with your friends and colleagues. It will help me a lot to grow my audience. Also, don't forget to subscribe to be updated when the next article comes out!

I you want to stay in touch here is my [twitter account][twitter]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

[twitter]: https://twitter.com/PierreChollet22
[sls-mentor]: https://www.sls-mentor.dev
[article-lambda]: https://dev.to/slsbytheodo/dont-miss-on-the-cloud-revolution-learn-serverless-on-aws-the-right-way-1kac
