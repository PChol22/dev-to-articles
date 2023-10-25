---
published: true
title: 'Getting started with AWS serverless - EventBridge'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/learn-serverless/eventbridge/assets/cover.png
tags: AWS, serverless, javascript, tutorial
series: 'Learn serverless on AWS step-by-step'
canonical_url:
---

## TL;DR

In this series, I try to explain the basics of serverless on AWS, to enable you to build your own serverless applications. With [last article][article ses], we discovered how to send emails using SES. In this article, let's dive into EventBridge, a service that allows you to build event-driven applications.

## Introduction

During the last 6 articles of this series, we only built synchronous applications: the user sent a request using an API, the request was processed, and then the user received a response with the result. This is a very common pattern, but it is not the only one. Sometimes, you want to treat the information in the background, without the user waiting for the result, and then notify the user when the processing is done. Events allow you to do that!

Building an application based on events can be achieved using multiple AWS service. One of the most common is EventBridge, which is the subject of this article. EventBridge allows you to create rules that will be triggered when an event occurs. These rules can then trigger a lambda function, or send a message to a queue, or even invoke an HTTP endpoint. The possibilities are endless!

### Let's build a flight booking application!

Today, we are going to build a simple flight booking app. The features will be the following:

- The user will be able to request a flight booking using an API
- If there are seats available, we will book the flight for the user
- At the same time, we will send an email to the user to confirm the booking
- Available seats will be automatically updated every day

The architecture will look like this:

![architecture](./assets/architecture.png 'architecture of the event-based app')

There will be 4 lambda functions, to request a booking, register the booking, send the email, and update the seats. The `bookFlight` lambda will send an event with EventBridge that triggers the `registerBooking` and `sendBookingReceipt` lambdas. The `syncFlights` lambda will be triggered every day to update the available seats, using another EventBridge rule. There will also be a DynamoDB table to store the bookings, and a SES Identity to send the emails.

In this architecture, using events allows us to decouple the different parts of the application. This allows us to easily change the implementation of each part without impacting the others. Furthermore, it unlocks the ability to trigger the syncFlights lambda every day without effort.

Except for the EventBridge rules, we already covered all the services used in this architecture. If you want to learn more about them, you can read the previous articles of this series!

## Provisioning the infrastructure

To code this application, I will use, as always, the AWS CDK for TypeScript. I cover the setup of the project in [the first article][article lambda] of this series, if you need a refresher!

### Create EventBridge rules

In the definition of the CDK stack, we can begin by creating an EventBus and the rules that will trigger our lambdas.

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import path from 'path';

export class LearnServerlessStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);
    // Create an eventBus
    const eventBus = new cdk.aws_events.EventBus(this, 'eventBus');
    // Create a rule to trigger the registerBooking and sendBookingReceipt lambdas
    const bookFlightRule = new cdk.aws_events.Rule(this, 'bookFlightRule', {
      eventBus,
      eventPattern: {
        source: ['bookFlight'],
        detailType: ['flightBooked'],
      },
    });
    // Create a rate rule to trigger the syncFlights lambda every day
    const syncFlightsRule = new cdk.aws_events.Rule(this, 'syncFlightsRule', {
      schedule: cdk.aws_events.Schedule.rate(cdk.Duration.days(1)),
    });
  }
}
```

In this code snippet:

- We create an EventBus, which is the entry point for events in EventBridge.
- Then, we create a rule that will trigger the `registerBooking` and `sendBookingReceipt` lambdas. The rule is configured to trigger when an event with the source `bookFlight` and the detailType `flightBooked` is sent to the EventBus.
- Finally, we create a rate rule that will trigger the `syncFlights` lambda every day, using the rate feature of EventBridge.

### Create the other resources: DynamoDB table, SES Identity and API Gateway

Then, let's create the other necessary resources.

```typescript
import { bookingReceiptHtmlTemplate } from './bookingReceiptHtmlTemplate';
// ...previous code

// Create a DynamoDB table to store the bookings
const flightTable = new cdk.aws_dynamodb.Table(this, 'flightTable', {
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

// Create an API Gateway to expose the bookFlight lambda
const api = new cdk.aws_apigateway.RestApi(this, 'api', {});

// Create a SES template to send nice emails
const bookingReceiptTemplate = new cdk.aws_ses.CfnTemplate(this, 'bookingReceiptTemplate', {
  template: {
    htmlPart: bookingReceiptHtmlTemplate,
    subjectPart: 'Your flight to {{destination}} was booked!',
    templateName: 'bookingReceiptTemplate',
  },
});

// This part is common to the previous article. No need to follow it if you already have a SES Identity
const DOMAIN_NAME = 'pchol.fr';

const hostedZone = new cdk.aws_route53.HostedZone(this, 'hostedZone', {
  zoneName: DOMAIN_NAME,
});

const identity = new cdk.aws_ses.EmailIdentity(this, 'sesIdentity', {
  identity: cdk.aws_ses.Identity.publicHostedZone(hostedZone),
});
```

In this code snippet:

- We create a DynamoDB table to store the bookings. The partition key will be the destination of the flight, and the sort key will be the date of the flight.
- We create an API Gateway to expose the bookFlight lambda.
- We create a SES template. The template is based on a HTML template.
- We create a SES Identity. This part is common to the [previous article][article ses]. If you already have a SES Identity, you can skip it.

I defined a simple HTML template that will define how the emails look, using CSS and placeholders. You can find mine here:

```typescript
export const bookingReceiptHtmlTemplate = `<html>
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
        <h1>Your flight was booked!</h1>
      </div>
      <div class="message">
        <p>Your flight was booked on {{flightDate}}, for {{numberOfSeats}} person(s), to {{destination}}!</p>
      </div>
    </div>
    <p class="footer">This is an automated message, please do not try to answer</p>
  </body>
</html>`;
```

### Create the lambda functions and plug everything together

Finally, we can create the lambda functions, and plug them to API Gateway and EventBridge, as well as grant them the necessary permissions and environment variables.

```typescript
// Create the bookFlight lambda
const bookFlight = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'bookFlight', {
  entry: path.join(__dirname, 'bookFlight', 'handler.ts'),
  handler: 'handler',
  environment: {
    TABLE_NAME: flightTable.tableName,
    EVENT_BUS_NAME: eventBus.eventBusName,
  },
});
bookFlight.addToRolePolicy(
  new cdk.aws_iam.PolicyStatement({
    actions: ['events:PutEvents'],
    resources: [eventBus.eventBusArn],
  }),
);
flightTable.grantReadData(bookFlight);
api.root.addResource('book-flight').addMethod('POST', new cdk.aws_apigateway.LambdaIntegration(bookFlight));

// Create the registerBooking lambda
const registerBooking = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'registerBooking', {
  entry: path.join(__dirname, 'registerBooking', 'handler.ts'),
  handler: 'handler',
  environment: {
    TABLE_NAME: flightTable.tableName,
  },
});
flightTable.grantReadWriteData(registerBooking);
bookFlightRule.addTarget(new cdk.aws_events_targets.LambdaFunction(registerBooking));

// Create the sendBookingReceipt lambda
const sendBookingReceipt = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'sendBookingReceipt', {
  entry: path.join(__dirname, 'sendBookingReceipt', 'handler.ts'),
  handler: 'handler',
  environment: {
    SENDER_EMAIL: `contact@${identity.emailIdentityName}`,
    TEMPLATE_NAME: bookingReceiptTemplate.ref,
  },
});
sendBookingReceipt.addToRolePolicy(
  new cdk.aws_iam.PolicyStatement({
    actions: ['ses:SendTemplatedEmail'],
    resources: [`*`],
  }),
);
bookFlightRule.addTarget(new cdk.aws_events_targets.LambdaFunction(sendBookingReceipt));

// Create the syncFlights lambda
const syncFlights = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'syncFlights', {
  entry: path.join(__dirname, 'syncFlights', 'handler.ts'),
  handler: 'handler',
  environment: {
    TABLE_NAME: flightTable.tableName,
  },
});
flightTable.grantWriteData(syncFlights);
syncFlightsRule.addTarget(new cdk.aws_events_targets.LambdaFunction(syncFlights));
```

Here we create 4 lambda functions:

- `bookFlight` has access to the table name and the event bus name. It is triggered by a POST route, and we grant it the permission to publish events on the event bus and read the table.
- `registerBooking` has access to the table name. It is triggered by the event bus using the `bookFlightRule.addTarget` method, and we grant it the permission to read and write the table.
- `sendBookingReceipt` has access to the sender email and the template name. It is triggered by the event bus using the `rule.addTarget` method, and we grant it the permission to send emails using SES.
- `syncFlights` has access to the table name. It is triggered by the event bus using the `syncFlightsRule.addTarget` method, and we grant it the permission to write the table.

And we are done with the infrastructure! Last step, the funny one, is to write the code for each lambda function.

## Write the code for each lambda function

### bookFlight

```typescript
import { DynamoDBClient, GetItemCommand } from '@aws-sdk/client-dynamodb';
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

const ddbClient = new DynamoDBClient({});
const eventBridgeClient = new EventBridgeClient({});

export const handler = async ({ body }: { body: string }): Promise<{ statusCode: number; body: string }> => {
  const tableName = process.env.TABLE_NAME;
  const eventBusName = process.env.EVENT_BUS_NAME;

  if (tableName === undefined || eventBusName === undefined) {
    throw new Error('Missing environment variables');
  }

  const { destination, flightDate, numberOfSeats, bookerEmail } = JSON.parse(body) as {
    destination?: string;
    flightDate?: string;
    numberOfSeats?: number;
    bookerEmail?: string;
  };

  if (
    destination === undefined ||
    flightDate === undefined ||
    numberOfSeats === undefined ||
    bookerEmail === undefined
  ) {
    return {
      statusCode: 400,
      body: JSON.stringify({
        message: 'Missing required parameters',
      }),
    };
  }

  const { Item } = await ddbClient.send(
    new GetItemCommand({
      TableName: tableName,
      Key: {
        PK: { S: `DESTINATION#${destination}` },
        SK: { S: flightDate },
      },
    }),
  );

  const availableSeats = Item?.availableSeats?.N;

  if (availableSeats === undefined) {
    return {
      statusCode: 404,
      body: JSON.stringify({
        message: 'Flight not found',
      }),
    };
  }

  if (+availableSeats < numberOfSeats) {
    return {
      statusCode: 400,
      body: JSON.stringify({
        message: 'Not enough seats for this flight',
      }),
    };
  }

  await eventBridgeClient.send(
    new PutEventsCommand({
      Entries: [
        {
          Source: 'bookFlight',
          DetailType: 'flightBooked',
          EventBusName: eventBusName,
          Detail: JSON.stringify({
            destination,
            flightDate,
            numberOfSeats,
            bookerEmail,
          }),
        },
      ],
    }),
  );

  return {
    statusCode: 200,
    body: JSON.stringify({
      message: 'Processing flight booking',
    }),
  };
};
```

There are 3 major steps in this lambda function:

- First, I parse the content of the body received from API Gateway. I detail more this approach in [my first article][article lambda].
- Then, wIe fetch the flight details from the flights table. I use the GetItemCommand, refresher on [this article][article dynamodb]. If there are no more available seats, we return an error.
- Finally, I publish an event on the event bus, using the PutEventsCommand. I specify a source and a detail type, matching with the rules we created earlier. I also specify the event bus name, and the detail of the event. The detail of the event will be available to the listener lambda functions.

This lambda return a 200 status code if everything went well, telling the user that the request was taken into account, even if it was not treated entirely yet. The user will receive an email at the end of the process.

### registerBooking

```typescript
import { DynamoDBClient, UpdateItemCommand } from '@aws-sdk/client-dynamodb';

const ddbClient = new DynamoDBClient({});

export const handler = async (event: {
  detail: {
    destination: string;
    flightDate: string;
    numberOfSeats: number;
  };
}): Promise<void> => {
  const { destination, flightDate, numberOfSeats } = event.detail;

  await ddbClient.send(
    new UpdateItemCommand({
      TableName: process.env.TABLE_NAME,
      Key: {
        PK: { S: `DESTINATION#${destination}` },
        SK: { S: flightDate },
      },
      UpdateExpression: 'SET availableSeats = availableSeats - :numberOfSeats',
      ExpressionAttributeValues: {
        ':numberOfSeats': { N: `${numberOfSeats}` },
      },
    }),
  );
};
```

This lambda function is triggered by the event bus, and it updates the number of available seats in the flights table. It uses the UpdateItemCommand, refresher on [this article][article dynamodb].

Notice the typing of the handler: it is different from the usual API Gateway event we were working with before. Remember that the detail of the event that was sent in the event bus is accessible in the `event.detail` property.

This lambda return nothing: it was triggered asynchronously, and no one is waiting for its response!

### sendBookingReceipt

```typescript
import { SESv2Client, SendEmailCommand } from '@aws-sdk/client-sesv2';

const sesClient = new SESv2Client({});

export const handler = async (event: {
  detail: {
    destination: string;
    flightDate: string;
    numberOfSeats: number;
    bookerEmail: string;
  };
}): Promise<void> => {
  const { destination, flightDate, numberOfSeats, bookerEmail } = event.detail;

  const senderEmail = process.env.SENDER_EMAIL;
  const templateName = process.env.TEMPLATE_NAME;

  if (senderEmail === undefined || templateName === undefined) {
    throw new Error('Missing environment variables');
  }

  await sesClient.send(
    new SendEmailCommand({
      FromEmailAddress: senderEmail,
      Content: {
        Template: {
          TemplateName: templateName,
          TemplateData: JSON.stringify({ destination, flightDate, numberOfSeats }),
        },
      },
      Destination: {
        ToAddresses: [bookerEmail],
      },
    }),
  );
};
```

This is the second lambda triggered by the bookFlight rule. It also has access to the event.detail property, and it uses it to send an email to the user. It uses the SendEmailCommand, refresher on how to achieve it in [this article][article ses].

Same deal, it does not return anything as it is triggered asynchronously.

### syncFlights

```typescript
import { DynamoDBClient, PutItemCommand } from '@aws-sdk/client-dynamodb';

const DESTINATIONS = ['CDG', 'LHR', 'FRA', 'IST', 'AMS', 'FCO', 'LAX'];

const client = new DynamoDBClient({});

export const handler = async (): Promise<void> => {
  const tableName = process.env.TABLE_NAME;

  if (tableName === undefined) {
    throw new Error('Table name not set');
  }

  const flightDate = new Date().toISOString().slice(0, 10);

  await Promise.all(
    DESTINATIONS.map(async destination =>
      client.send(
        new PutItemCommand({
          TableName: tableName,
          Item: {
            PK: { S: `DESTINATION#${destination}` },
            SK: { S: flightDate },
            availableSeats: { N: '2' },
          },
        }),
      ),
    ),
  );
};
```

This lambda is very simple, it is triggered by a cron rule, and it creates a new item in the flights table for each destination. It uses the PutItemCommand, refresher on [this article][article dynamodb]. I only put mocked date into the table for the sake of simplicity. Each time, there are 2 available seats.

As the lambda is triggered by a schedule, it is also asynchronous and does not return anything.

## Testing our application!

We are done with the code of the lambda functions. Time to deploy the application and test it!

```bash
npm run cdk deploy
```

If you do not want to wait 1 day for the syncFlights lambda to be triggered, you can trigger it manually from the AWS console by clicking on the "Test" button in the lambda function page. The lambda does not need any payload so it's easy to trigger manually.

Then, there is only one API call to execute: a POST request on /book-flight to request a booking!

![Postman request success](./assets/postman-success.png 'Postman request success')

We receive a 200 response, telling us that the request was taken into account. Some seconds later, we receive an email with the booking receipt. If you did not receive the email, troubleshoot using cloudwatch and my last SES [article][article ses].

![Email received](./assets/email.png 'Email received')

If everything works correctly, there should only be 1 seat left on the LHR destination, so if I request 2 seats, I should receive a 400 response.

![Postman request failure](./assets/postman-failure.png 'Postman request failure')

This is exactly what happens! In this case, no email is received as the workflow was stopped early.

## Homework 🤓

If you read all my previous articles, you are able to incorporate S3, Cognito and Step-Functions to the application. With this knowledge, you should be able to implement the following features:

- Add user authentication to the application
- Store tickets in S3
- Implement a mocked payment workflow with step functions

You could also plug yourself into real world APIs to get the flights data if you wish!

These are only examples! You can do whatever you want with the application, and I would be happy to see what you come up with! If you want to share your work, do not hesitate to contact me on [twitter][twitter account]!

Articles are becoming more and more complex in their implementation: we are getting closer to real world use cases. I hope you are enjoying the series so far, and that you are learning a lot!

## Conclusion

This tutorial was only a small practical example of what you can do with events on AWS. There are a lot of other use cases, with cleaner and more efficient solutions. I hope it helped you understand the basics of event-driven applications, and how to use them on AWS.

I plan to continue this series of articles on a bi-monthly basis. I already covered the creation of simple lambda functions and REST APIs, as well as interacting with DynamoDB databases and S3 buckets. You can follow this progress on my [repository][repository]! I will cover new topics like front-end deployment, type safety, more advanced patterns, and more... If you have any suggestions, do not hesitate to contact me!

I would really appreciate if you could react and share this article with your friends and colleagues. It will help me a lot to grow my audience. Also, don't forget to subscribe to be updated when the next article comes out!

I you want to stay in touch here is my [twitter account][twitter account]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

[repository]: https://github.com/PChol22/learn-serverless
[twitter account]: https://twitter.com/PierreChollet22
[article lambda]: https://dev.to/kumo/dont-miss-on-the-cloud-revolution-learn-serverless-on-aws-the-right-way-1kac
[article ses]: https://dev.to/kumo/learn-serverless-on-aws-step-by-step-emails-49hp
[article dynamodb]: https://dev.to/kumo/learn-serverless-on-aws-step-by-step-databases-kkg
