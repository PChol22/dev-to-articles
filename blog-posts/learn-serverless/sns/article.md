---
published: true
title: 'Learn serverless on AWS step-by-step - SNS'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/learn-serverless/sns/assets/cover.png
tags: AWS, serverless, javascript, tutorial
series: 'Learn serverless on AWS step-by-step'
canonical_url:
---

## TL;DR

In this series, I try to explain the basics of serverless on AWS, to enable you to build your own serverless applications. With [last article][article-aurora], we discovered how to deploy and interact with SQL databases on AWS, using Aurora Serverless. In this article, we will tackle SNS topics, which allow to create pub/sub patterns in your applications!

**What will we do today?**

In this article, we will create a SNS topic, and use it to send notifications to multiple Lambda functions. We will see how we can leverage this to send targeted notifications to specific parts of our application, depending on the context, and how to use it to decouple it.

⬇️ I post serverless content very regularly, if you want more ⬇️

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

## Amazon Simple Notification Service (SNS)

### What is SNS?

Amazon SNS is a serverless pub/sub service. It allows you to create topics. Topics have producers and consumers. Producers can publish messages to a topic, and consumers can subscribe to a topic to receive messages. When a message is published into a topic, every consumer will eventually receive it, it is called a fan-out pattern.

This is a very powerful pattern, as it allows to decouple producers and consumers. Producers do not need to know who will consume their messages, and consumers do not need to know who will produce them. A single action can trigger multiple consumers.

SNS also enables you to create filters on topics, so that consumers can subscribe to a subset of messages. This allows to create targeted notifications, and to decouple even more your application. For example, a RequestDelivery Lambda function could only be interested in messages related to delivery requests, and not in messages related to payment requests.

### Let's build a simple app to demonstrate SNS!

Today, we are going to build a very simple app:

- A `OrderItem` Lambda is triggered by a POST request on a REST API. The user can specify if he wants to receive a notification, as well as if he wants to request a delivery for his order.
- The `OrderItem` Lambda will publish a message to a SNS topic, with the order details.
- Downstream, three lambda functions will be triggered by the SNS topic:

  - A `ExecuteOrder` Lambda, which will simulate a payment.
  - A `RequestDelivery` Lambda, which will simulate a delivery, only if the user requested it.
  - A `Notification` Lambda, which will send a notification to the user, only if he requested it.

The architecture of our app will look like this:

![Architecture](./assets/architecture.png 'Architecture')

Every action will be simulated by a simple `console.log`, but with the combined knowledge of this whole series, you can easily replace them with real actions, on DynamoDB, S3, SES, or any other AWS service!

## Create a SNS topic

To build this app, we will use the AWS CDK. If you are not familiar with it, I suggest you read the [start of my series][article-1] where I talk about it in details. We will create a new CDK project, add the `@aws-cdk/aws-sns` package to it, and create a new SNS topic in the stack, as well as an API Gateway to trigger our OrderItem Lambda.

```typescript
import * as cdk from 'aws-cdk-lib';

import path from 'path';

export class ArticleSNS extends cdk.Stack {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    const topic = new cdk.aws_sns.Topic(this, 'topic');

    const api = new cdk.aws_apigateway.RestApi(this, 'api', {});

    const orderItem = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'OrderItem', {
      entry: path.join(__dirname, 'orderItem', 'handler.ts'),
      handler: 'handler',
      environment: {
        TOPIC_ARN: topic.topicArn,
      },
    });
    topic.grantPublish(orderItem);
    api.root.addResource('orderItem').addMethod('POST', new cdk.aws_apigateway.LambdaIntegration(orderItem));
  }
}
```

Notice that we grant the `publish` permission to the `OrderItem` Lambda, so that it can publish messages to the topic. We also pass the topic ARN to the Lambda as an environment variable, so that it can use it to publish messages.

## Subscribe Lambdas to the SNS topic

Now, let's create the 3 downstream lambda functions and subscribe them to the topic. We will implement filters on the topic, so that each lambda function only receives the messages it is interested in.

```typescript
// ... previous code

const executeOrder = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'ExecuteOrder', {
  entry: path.join(__dirname, 'executeOrder', 'handler.ts'),
  handler: 'handler',
});
topic.addSubscription(new cdk.aws_sns_subscriptions.LambdaSubscription(executeOrder));

const requestDelivery = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'RequestDelivery', {
  entry: path.join(__dirname, 'requestDelivery', 'handler.ts'),
  handler: 'handler',
});
topic.addSubscription(
  new cdk.aws_sns_subscriptions.LambdaSubscription(requestDelivery, {
    filterPolicy: {
      // Only triggers when the "requestDelivery" attribute is set to "true"
      requestDelivery: cdk.aws_sns.SubscriptionFilter.stringFilter({ allowlist: ['true'] }),
    },
  }),
);

const sendNotification = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'SendNotification', {
  entry: path.join(__dirname, 'sendNotification', 'handler.ts'),
  handler: 'handler',
});
topic.addSubscription(
  new cdk.aws_sns_subscriptions.LambdaSubscription(sendNotification, {
    filterPolicy: {
      // Only triggers when the "sendNotification" attribute is set to "true"
      sendNotification: cdk.aws_sns.SubscriptionFilter.stringFilter({ allowlist: ['true'] }),
    },
  }),
);
```

See, nothing too complicated. Using the filterPolicy parameter, we can specify which messages should trigger the lambda function. In our case, we want to trigger the RequestDelivery Lambda only when the `requestDelivery` attribute is set to `true`, and the SendNotification Lambda only when the `sendNotification` attribute is set to `true`.

## Publish messages to the SNS topic

Finally, the interesting part: time to write the code of the Lambdas! Let's start with the `OrderItem` Lambda, which will publish the message to the topic.

```typescript
// orderItem/handler.ts
import { PublishCommand, SNSClient } from '@aws-sdk/client-sns';

const client = new SNSClient({});

export const handler = async (event: { body: string }): Promise<{ statusCode: number; body: string }> => {
  const topicArn = process.env.TOPIC_ARN;

  if (topicArn === undefined) {
    throw new Error('TOPIC_ARN is undefined');
  }

  const { requestDelivery, sendNotification, item, quantity } = JSON.parse(event.body) as {
    requestDelivery?: boolean;
    sendNotification?: boolean;
    item?: string;
    quantity?: number;
  };

  if (requestDelivery === undefined || sendNotification === undefined || item === undefined || quantity === undefined) {
    return {
      statusCode: 400,
      body: 'Bad request',
    };
  }

  await client.send(
    new PublishCommand({
      Message: JSON.stringify({ item, quantity }),
      TopicArn: topicArn,
      MessageAttributes: {
        sendNotification: {
          DataType: 'String',
          StringValue: sendNotification.toString(),
        },
        requestDelivery: {
          DataType: 'String',
          StringValue: requestDelivery.toString(),
        },
      },
    }),
  );

  return {
    statusCode: 200,
    body: 'Item ordered',
  };
};
```

This code does three things:

- It gets the topic ARN from the environment variables.
- It parses the body of the request, and extracts the `requestDelivery`, `sendNotification`, `item` and `quantity` attributes.
- It publishes a message to the topic:
  - The message body contains the business data, `item` and `quantity`.
  - The attributes `requestDelivery` and `sendNotification` are set to `true` or `false`, depending on the request. This attributes are the ones that will be used by the filters we defined earlier.

For the three downstream Lambdas, we will use console.log to be able to see in the AWS console if they are triggered or not.

```typescript
// executeOrder/handler.ts
export const handler = async (event: {
  Records: {
    Sns: {
      Message: string;
    };
  }[];
}): Promise<void> => {
  event.Records.forEach(({ Sns: { Message } }) => {
    const { item, quantity } = JSON.parse(Message) as { item: string; quantity: number };

    console.log(`ORDER EXECUTED - Item: ${item}, Quantity: ${quantity}`);
  });
};
```

In this Lambda, I log the content of the messages received from the topic. The interesting part is the type of the event parameter: it is an array of records. This is because SNS can send multiple messages at once, so we need to handle this case.

```typescript
// requestDelivery/handler.ts
export const handler = (): Promise<void> => {
  console.log('DELIVERY REQUESTED');
};
```

```typescript
// sendNotification/handler.ts
export const handler = (): Promise<void> => {
  console.log('NOTIFICATION SENT');
};
```

As I already said, if you followed this series from the start, you should be able to replace this console.log statements with real actions, such as updating a DynamoDB or SQL database, send an email with SES, or anything else!

## Time to test it!

First, let's deploy the app thanks to the AWS CDK:

```bash
npm run cdk deploy
```

Then, let's test it by sending API calls with postman:

In the first call, I set `sendNotification` and `requestDelivery` to false. Only the `ExecuteOrder` Lambda is triggered, as expected. We can see the logs in CloudWatch, with the content of the message.

![Postman](./assets/firstCall.png 'first postman call')

![CloudWatch](./assets/cloudWatchFirstCall.png 'logs of the first call')

In the second call, I set `sendNotification` to true. The `ExecuteOrder` and `SendNotification` Lambdas are triggered. We can see the logs of `SendNotification` in CloudWatch.

![Postman](./assets/secondCall.png 'second postman call')

![CloudWatch](./assets/cloudWatchSecondCall.png 'logs of the second call')

In the third call, I set `requestDelivery` to true. The `ExecuteOrder` and `RequestDelivery` Lambdas are triggered. We can see the logs of `RequestDelivery` in CloudWatch.

![Postman](./assets/thirdCall.png 'third postman call')

![CloudWatch](./assets/cloudWatchThirdCall.png 'logs of the third call')

## Conclusion

This was a very shallow introduction to SNS. I demonstrated how to create a topic, subscribe Lambdas to it, and publish messages to it. I didn't code any real side-effect, but you should be able to do it by yourself, using the knowledge you acquired in the previous articles of this series.

SNS also allows to send emails, SMS or push notifications to mobile apps, but I didn't cover this in this article. I will probably write another article about it in the future!

I plan to continue this series of articles on a bi-monthly basis. I already covered the creation of simple lambda functions and REST APIs, as well as interacting with DynamoDB databases and S3 buckets. You can follow this progress on my [repository][repository]! I will cover new topics like front-end deployment, type safety, more advanced patterns, and more... If you have any suggestions, do not hesitate to contact me!

I would really appreciate if you could react and share this article with your friends and colleagues. It will help me a lot to grow my audience. Also, don't forget to subscribe to be updated when the next article comes out!

I you want to stay in touch here is my [twitter account][twitter]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

[article-aurora]: https://dev.to/kumo/learn-serverless-on-aws-step-by-step-sql-with-aurora-5hn1
[twitter]: https://twitter.com/PierreChollet22
[repository]: https://github.com/PChol22/learn-serverless
[article-1]: https://dev.to/kumo/dont-miss-on-the-cloud-revolution-learn-serverless-on-aws-the-right-way-1kac
