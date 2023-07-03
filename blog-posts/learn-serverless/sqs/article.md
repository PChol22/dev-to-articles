---
published: true
title: 'Getting started with AWS serverless - SQS'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/learn-serverless/sqs/assets/cover.png
tags: AWS, serverless, javascript, tutorial
series: 'Learn serverless on AWS step-by-step'
canonical_url:
---

## TL;DR

In this series, I try to explain the basics of serverless on AWS, to enable you to build your own serverless applications. With [last article][article eventbridge], we discovered how to use EventBridge to build event-driven applications. Today, we will dive deeper into events management by taking a look at SQS and its integration with lambda functions.

## Introduction

SQS is Amazon's Simple Queue Service. As its name suggests, it is a fully managed queue service, that allows you to store messages while waiting for them to be processed. It is a very useful service to decouple your applications, and to build event-driven applications. It is also a very good way to handle asynchronous tasks, and to manage your application's load.

In this article, we will use SQS to find a solution to a problem: imagine you have an external API that only allows 1 connection at a time (for example, to avoid spamming). How do you prevent it to be overwhelmed by your users, but still make sure that every user request will be eventually processed? This is where SQS comes in handy!

One of SQS use cases is to store messages and limit the throughput of your application. If a lambda function processes your messages, you can limit the number of concurrent executions of this function (here we would set it to 1), and this Lambda function will process all the messages stored in the queue 1 by 1.

Resumed in a small schema it would look like this:

![sqs explained](./assets/sqs-explained.png '1 msg at a time SQS explained')

## What are we going to build?

Based on this use case, let's build a fake ordering app, where a constraint is that only 1 order can be processed at a time. We will use SQS to store the orders, and a lambda function to process them. Using this method, there is a possibility that a user has to wait several minutes before his order is processed: to fix this, we will publish an event with EventBridge when the order is processed, and the user will be notified by an email (using SES) when his order is ready.

The app should look like this once we are done:

![app architecture](./assets/architecture.png 'App architecture')

## Creating the SQS queue and its target Lambda function

As always, you are going to use AWS CDK combined with TypeScript to provision this application. If you need a refresher, you can check the [first article of this series][article lambda], where I go deeper into the setup of the project.

Let's start by the core of our application: the SQS queue and the lambda function that will process the orders.

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import path from 'path';

export class LearnServerlessStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // create a FIFO SQS queue
    const ordersQueue = new cdk.aws_sqs.Queue(this, 'ordersQueue', {
      visibilityTimeout: cdk.Duration.seconds(180),
      fifo: true,
    });

    // defined an event source for the queue, with a batch size of 1
    const eventSource = new cdk.aws_lambda_event_sources.SqsEventSource(ordersQueue, {
      batchSize: 1,
    });

    // create a Lambda function that will process the orders, bind it to the event source
    const executeOrder = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'executeOrder', {
      entry: path.join(__dirname, 'executeOrder', 'handler.ts'),
      handler: 'handler',
      reservedConcurrentExecutions: 1,
      timeout: cdk.Duration.seconds(30),
    });
    executeOrder.addEventSource(eventSource);
  }
}
```

With this code snippet, you will provision a SQS Queue and a Lambda function. Every message sent to the queue will trigger the lambda function, and the lambda function will process the messages 1 by 1, because of the concurrency (1 execution at a time), and the batch size (each message is composed of 1 order).

I set a timeout of 30 seconds for the lambda function (for demo purposes, I want the fake processing to be very long), and the visibility timeout to 150 seconds: AWS recommends to set the visibility timeout to 6 times the timeout of your lambda function, so that the message is not processed twice if the lambda function fails. It's a tricky topic, learn more [here][visibility timeout].

## Provision the rest of the infrastructure

### Non-lambda resources

As seen on the introduction schema, we also need to provision an event bus, an API gateway and a SES Identity. Let's do it!

```typescript
import { orderExecutedHtmlTemplate } from './orderExecutedHtmlTemplate';
// ...previous code

// Provision a rest API
const restApi = new cdk.aws_apigateway.RestApi(this, 'restApi', {});

// Provision an event bus and a rule to trigger the notification Lambda function
const ordersEventBus = new cdk.aws_events.EventBus(this, 'ordersEventBus');
const notifyOrderExecutedRule = new cdk.aws_events.Rule(this, 'notifyOrderExecutedRule', {
  eventBus: ordersEventBus,
  eventPattern: {
    source: ['notifyOrderExecuted'],
    detailType: ['orderExecuted'],
  },
});

// Provision a SES template to send beautiful emails
const orderExecutedTemplate = new cdk.aws_ses.CfnTemplate(this, 'orderExecutedTemplate', {
  template: {
    htmlPart: orderExecutedHtmlTemplate,
    subjectPart: 'Your order was passed to our provider!',
    templateName: 'orderExecutedTemplate',
  },
});

// This part is common to my SES article. No need to follow it if you already have a SES Identity
const DOMAIN_NAME = 'pchol.fr';

const hostedZone = new cdk.aws_route53.HostedZone(this, 'hostedZone', {
  zoneName: DOMAIN_NAME,
});

const identity = new cdk.aws_ses.EmailIdentity(this, 'sesIdentity', {
  identity: cdk.aws_ses.Identity.publicHostedZone(hostedZone),
});
```

In this snippet, I create all the necessary resources, this is based on previous articles, if you need a refresher on [API Gateway][article lambda], [EventBridge][article eventbridge] or [SES][article ses], you can check them out!

I used a simple HTML template to send the email, exported from a .ts file, it contains the variables `{{itemName}}`, `{{quantity}}` and `{{username}}`, that will be replaced by the values of the order.

```typescript
export const orderExecutedHtmlTemplate = `<html>
  <head>
    <style>
      * {
        font-family: sans-serif;
        text-align: center;
        padding: 0;
        margin: 0;
      }
      .title {
        color: #fff;
        background: #17bb90;
        padding: 1em;
      }
      .container {
        border: 2px solid #17bb90;
        border-radius: 1em;
        margin: 1em auto;
        max-width: 500px;
        overflow: hidden;
      }
      .message {
        padding: 1em;
        line-height: 1.5em;
        color: #033c49;
      }
      .footer {
        font-size: .8em;
        color: #888;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <div class="title">
        <h1>Hello {{username}}!</h1>
      </div>
      <div class="message">
        <p>Your order of {{quantity}} {{itemName}} was passed to our provider!</p>
      </div>
    </div>
    <p class="footer">This is an automated message, please do not try to answer</p>
  </body>
</html>`;
```

### Lambda functions and interactions

To end the provisioning part of this article, let's create the two missing lambda functions, and the interfaces between theme and the other resources.

```typescript
// ... previous code

// Create the request order lambda function
const requestOrder = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'requestOrder', {
  entry: path.join(__dirname, 'requestOrder', 'handler.ts'),
  handler: 'handler',
  environment: {
    QUEUE_URL: ordersQueue.queueUrl,
  },
});

// Grant the lambda function the right to send messages to the SQS queue, add API Gateway as a trigger
ordersQueue.grantSendMessages(requestOrder);
restApi.root.addResource('request-order').addMethod('POST', new cdk.aws_apigateway.LambdaIntegration(requestOrder));

const executeOrder = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'executeOrder', {
  entry: path.join(__dirname, 'executeOrder', 'handler.ts'),
  handler: 'handler',
  environment: {
    EVENT_BUS_NAME: ordersEventBus.eventBusName, // NEW: Add EVENT_BUS_NAME to the environment variables of the executeOrder lambda function
  },
  reservedConcurrentExecutions: 1,
  timeout: cdk.Duration.seconds(30),
});

executeOrder.addEventSource(eventSource);
// NEW: grant the lambda function the right to put events to the event bus
executeOrder.addToRolePolicy(
  new cdk.aws_iam.PolicyStatement({
    actions: ['events:PutEvents'],
    resources: [ordersEventBus.eventBusArn],
  }),
);

// Create the notifyOrderExecuted lambda function
const notifyOrderExecuted = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'notifyOrderExecuted', {
  entry: path.join(__dirname, 'notifyOrderExecuted', 'handler.ts'),
  handler: 'handler',
  environment: {
    SENDER_EMAIL: `contact@${identity.emailIdentityName}`,
    TEMPLATE_NAME: orderExecutedTemplate.ref,
  },
});

// Grant the lambda function the right to send emails, add the lambda as a target of the event rule
notifyOrderExecuted.addToRolePolicy(
  new cdk.aws_iam.PolicyStatement({
    actions: ['ses:SendTemplatedEmail'],
    resources: ['*'],
  }),
);
notifyOrderExecutedRule.addTarget(new cdk.aws_events_targets.LambdaFunction(notifyOrderExecuted));
```

We are done with the provisioning part! Let's move on to the most interesting part: the code deployed inside the lambda functions.

### Lambda functions deployed code

Let's start with the requestOrder lambda function. This function is triggered by a POST HTTP request, and will send a message to the SQS queue. It will also return a 200 HTTP status code to the client in case of success.

```typescript
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs';
import { v4 as uuidv4 } from 'uuid';

const client = new SQSClient({});

export const handler = async ({ body }: { body: string }): Promise<{ statusCode: number; body: string }> => {
  const queueUrl = process.env.QUEUE_URL;

  if (queueUrl === undefined) {
    throw new Error('Missing environment variables');
  }

  const { itemName, quantity, username, userEmail } = JSON.parse(body) as {
    itemName?: string;
    quantity?: number;
    username?: string;
    userEmail?: string;
  };

  if (itemName === undefined || quantity === undefined || username === undefined || userEmail === undefined) {
    return Promise.resolve({
      statusCode: 400,
      body: JSON.stringify({ message: 'Missing required parameters' }),
    });
  }

  await client.send(
    new SendMessageCommand({
      QueueUrl: queueUrl,
      MessageBody: JSON.stringify({ itemName, quantity, username, userEmail }),
      MessageGroupId: 'ORDER_REQUESTED',
      MessageDeduplicationId: uuidv4(),
    }),
  );

  return Promise.resolve({
    statusCode: 200,
    body: JSON.stringify({ message: 'Order requested' }),
  });
};
```

This snippet does the following things:

- Usual: parse the body of the POST request to get the 4 values we need
- Send a message to the SQS queue, with a unique ID to avoid duplicates, and a constant group ID to ensure the order of the messages inside this group
- Return a 200 HTTP status code to the client

Next lambda function: executeOrder. This function is triggered by the SQS queue, so it will have a special typing as input. It will fake a 20 seconds connection with an external API, and then send an event on the event bus.

```typescript
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

const client = new EventBridgeClient({});

export const handler = async (event: {
  Records: {
    body: string;
  }[];
}): Promise<void> => {
  const eventBusName = process.env.EVENT_BUS_NAME;

  if (eventBusName === undefined) {
    throw new Error('Missing environment variables');
  }

  const { body } = event.Records[0];

  console.log('Communication with external API started...');
  await new Promise(resolve => setTimeout(resolve, 20000));
  console.log('Communication with external API finished!');

  await client.send(
    new PutEventsCommand({
      Entries: [
        {
          EventBusName: eventBusName,
          Source: 'notifyOrderExecuted',
          DetailType: 'orderExecuted',
          Detail: body,
        },
      ],
    }),
  );
};
```

This snippet does the following things:

- New: parse the SQS input. The type is an array of records. Because we set the batch size to 1, we can assume that the array will always have a length of 1
- Wait for 20 seconds to fake a connection with an external API
- Send an event on the event bus, with the body of the SQS message as the detail. Notice I set for this call the same source and detail type as the event rule target, otherwise the target would not be triggered

Final lambda function: notifyOrderExecuted. This function is triggered by the event bus, so it will have another typing as input (refresher [here][article eventbridge]). It will send an email to the user, using a template stored in SES.

```typescript
import { SESv2Client, SendEmailCommand } from '@aws-sdk/client-sesv2';

const client = new SESv2Client({});

export const handler = async (event: {
  detail: {
    itemName: string;
    quantity: number;
    username: string;
    userEmail: string;
  };
}): Promise<void> => {
  const senderEmail = process.env.SENDER_EMAIL;
  const templateName = process.env.TEMPLATE_NAME;

  if (senderEmail === undefined || templateName === undefined) {
    throw new Error('Missing environment variables');
  }

  const { itemName, quantity, username, userEmail } = event.detail;

  await client.send(
    new SendEmailCommand({
      FromEmailAddress: senderEmail,
      Content: {
        Template: {
          TemplateName: templateName,
          TemplateData: JSON.stringify({ itemName, quantity, username }),
        },
      },
      Destination: {
        ToAddresses: [userEmail],
      },
    }),
  );
};
```

This snippet does the following things:

- Parse the EventBridge input. It was automatically parsed from string to object, we just have to pick properties we need.
- Send a templated email using SES. Remember that the TemplateData must contain exactly the same keys as the template you created in SES, otherwise the send will silently fail.

We are done with the code! Let's finish this article by testing our app!

## Testing our application

For this test, I'm going to make 2 consecutive API calls to the /request-order endpoint. If everything is alright, I should receive an email after ~20 seconds, and a second email after ~40 seconds (because the executeOrder Lambda only processes one message at a time, and sleeps for 2O seconds).

Here are the 2 requests I made:

![request-1](./assets/request-1.png 'first request')

![request-2](./assets/request-2.png 'second request')

I ordered 4 bananas and 43 cookies! (I am very hungry...)

Now let's check my emails:

![email-1](./assets/email-1.png 'first email')

![email-2](./assets/email-2.png 'second email')

I received the 2 emails, with the correct quantities! Trust me when I say that I received the first email after ~20 seconds, and the second one after ~40 seconds 😇.

## Homework 🤓

We only built a minimalistic application, and there are a lot of things we can improve. Here are some ideas that you should definitely be able to try if you followed this series:

- Add a database to store the orders, and a GET endpoint to retrieve them
- Only allow authenticated users to request orders
- Interact with a real API to list the items and their prices

You could also build a small front-end interacting with this back-end, but I will cover this in a future article 😉.

## Conclusion

This tutorial was only a small practical example of what you can do with events and SQS on AWS. SQS can be adapted to way more use cases, and I encourage you to check the [documentation][sqs documentation] to learn more about it!

I plan to continue this series of articles on a bi-monthly basis. I already covered the creation of simple lambda functions and REST APIs, as well as interacting with DynamoDB databases and S3 buckets. You can follow this progress on my [repository][repository]! I will cover new topics like front-end deployment, type safety, more advanced patterns, and more... If you have any suggestions, do not hesitate to contact me!

I would really appreciate if you could react and share this article with your friends and colleagues. It will help me a lot to grow my audience. Also, don't forget to subscribe to be updated when the next article comes out!

I you want to stay in touch here is my [twitter account][twitter account]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

[repository]: https://github.com/PChol22/learn-serverless
[twitter account]: https://twitter.com/PierreChollet22
[article eventbridge]: https://dev.to/kumo/learn-serverless-on-aws-step-by-step-eventbridge-27aa
[article lambda]: https://dev.to/kumo/dont-miss-on-the-cloud-revolution-learn-serverless-on-aws-the-right-way-1kac
[sqs documentation]: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
[visibility timeout]: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
[article ses]: https://dev.to/kumo/learn-serverless-on-aws-step-by-step-emails-49hp
