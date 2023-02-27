---
published: true
title: 'Learn serverless on AWS step-by-step - Databases'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/learn-serverless/databases/assets/cover.png
tags: AWS, serverless, javascript, tutorial
series: 'Learn serverless on AWS step-by-step'
canonical_url:
---

_In the [last article][last article], I covered the basics of creating Lambda functions on AWS, using the CDK. In this article, I will cover how to store data in a database using DynamoDB._

## Store data in a serverless database using DynamoDB

AWS offers many ways to store data, but in this article, I will cover the most common service allowing to store data in a serverless way: DynamoDB. DynamoDB is a NoSQL database, which means that it does not use SQL to query data. It is a key-value store: basically, you store data under the form of JSON objects, that can be queried using a key.

Like Lambda and API Gateway that we discovered last time, DynamoDB is managed by AWS and serverless, it means that you don't have to manage the infrastructure, and you just have to pay for the resources you use (storage, requests, etc.). When using a small quantity of data and IOPS (Input Output Per Second), DynamoDB is free of charge so you can start using it without any cost.

In DynamoDB, data is organized into tables. Tables are used to store Key-Value pairs. Each table has a primary key, used to uniquely identify each item in the table. Often, the primary key is composed of a Partition Key (PK) and a Sort Key (SK). The PK is used to identify a partition of the sorted data (a subset of the data), and the SK is used to sort the data within the partition.

All the other keys are called attributes, and you can basically store any kind of data in them: DynamoDB was designed to store multiple kinds of items in the same table. For example, see bellow a table storing users and notes:

![Simple DB](./assets/simpleDB.png 'Simple table in DynamoDB')

The PK is used to determine whether an item is a user or a note. The SK is to uniquely identify the item in the table, using a unique ID (UUID), the other attributes are not always present: users have a userName and an age, but notes only have a noteContent.

Using this design, you can for example list all the users in the table by making a query with the PK set to "user", or get a single note setting the PK to "note" and the SK to its unique ID. Remember that you always have to specify at least the PK when querying data in DynamoDB (querying users and notes at the same time in the example above would be an anti-pattern).

DynamoDB is a very wide a complex topic, and I will not cover all the details in this article. If you want to learn more about DynamoDB, I recommend you to take a look to the [official documentation][dynamodb-docs].

## Example: create a database that stores notes

Let's create a simple application that stores notes in a DynamoDB table. At the end of the article, a user will be able to create and read a note. We could also implement listing, updating and deleting for example, but I will leave this as homework 🤓.

Take a look at the architecture we want to build:

![Architecture](./assets/architecture.png 'Architecture of the project')

The application will be composed of a REST API made of two routes, two Lambda functions and a DynamoDB table. The first route will be a POST used to create a note, and the second one a GET to read a note. The Lambda functions will be triggered by the API Gateway, and will interact with the DynamoDB table.

About the data structure, we will use a design similar to what I described in the image above. The `PK` will be the user ID, and the `SK` will be the note ID. The note content will be stored in the `noteContent` attribute. This structure allows to get any note knowing the user ID and the note ID, and also to list all the notes of a user knowing its user ID.

![Database structure](./assets/dbStructure.png 'Structure of the database')

In this article, I will start from the project I created in the [last article][last article]. If you want to follow along, you can clone the [repository] and continue from the `introduction` branch. If you want to start from scratch, you can use the CDK to create a new project, by following the instructions of the last episode.

### Create the DynamoDB table

First, we need to create the DynamoDB table. Like for Lambda functions, We will use the CDK to do so. In the `my-first-app-stack.ts` file, in the constructor, add the declaration of the table:

```typescript
//... previous code

export class MyFirstAppStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    //... previous code

    const notesTable = new cdk.aws_dynamodb.Table(this, 'notesTable', {
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
  }
}
```

In this snippet, you create a table, set 'PK' as the partition key and 'SK' as the sort key, they both store `string` data. The billing mode is set to PAY_PER_REQUEST, which means that you will only pay for the resources you use. You can also set a fixed price for the table, but it is not recommended for small applications. (see this very nice [article][pay-per-use-ddb] for more information).

### Create two Lambda functions interacting with the database

Now that we have a database, we need to create two Lambda functions that will interact with it. The first one will be used to create a note, and the second one to read a note. Like always (usual business 😎) we will use the CDK to create the functions.

In the `my-first-app-stack.ts` file, in the constructor, add the declaration of the two functions:

```typescript
//... previous code

const createNote = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'createNote', {
  entry: path.join(__dirname, 'createNote', 'handler.ts'),
  handler: 'handler',
  environment: {
    TABLE_NAME: notesTable.tableName, // VERY IMPORTANT
  },
});

const getNote = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'getNote', {
  entry: path.join(__dirname, 'getNote', 'handler.ts'),
  handler: 'handler',
  environment: {
    TABLE_NAME: notesTable.tableName, // VERY IMPORTANT
  },
});

notesTable.grantWriteData(createNote); // VERY IMPORTANT
notesTable.grantReadData(getNote); // VERY IMPORTANT
```

⚠️ Notice two differences with the Lambda functions we created in the [last article][last article]. **These two differences are the essence of the relationship between the table and the functions**:

- We set environment variables containing the name of the table. Thanks to this, the runtime code of our lambda (defined in handler.ts) will be able to know which table to interact with, by using `process.env.TABLE_NAME`.

- We grant the Lambda functions the right to interact with the database. This is very important, otherwise the Lambda functions will not be able to access the database. This rights management is done using IAM policies, nothing too complicated for now, but be sure there will be an article covering this (huge) topic in the future 😉.

### Link the Lambda functions to the REST API

Now that we have created the Lambda functions, we need to link them to the REST API. Under the definition of the table in `my-first-app-stack.ts`, add the following code:

```typescript
// myFirstApi was already defined in the previous article
const notesResource = myFirstApi.root.addResource('notes').addResource('{userId}');

notesResource.addMethod('POST', new cdk.aws_apigateway.LambdaIntegration(createNote));

notesResource.addResource('{id}').addMethod('GET', new cdk.aws_apigateway.LambdaIntegration(getNote));
```

Basically, we add two resources to the REST API:

- A POST /notes/{userId} resource, which will trigger the `createNote` Lambda function. It will also have a body containing the content of the note.
- A GET /notes/{userId}/{id} resource, which will trigger the `getNote` Lambda function.

### Create the code of the two Lambda functions

Before writing the code, you need to install two packages needed in this article: `@aws-sdk/client-dynamodb` and `uuid`. The first one is the official AWS SDK for DynamoDB, needed to communicate with the database, and the second one is a package to generate unique IDs.

```bash
npm install @aws-sdk/client-dynamodb uuid
npm install --save-dev @types/uuid
```

Let's create the code for the two Lambda functions. In a `createNote` folder, create a `handler.ts` file, and add the following code:

```typescript
import { DynamoDBClient, PutItemCommand } from '@aws-sdk/client-dynamodb';
import { v4 as uuidv4 } from 'uuid';

const client = new DynamoDBClient({});

export const handler = async (event: {
  body: string;
  pathParameters: { userId?: string };
}): Promise<{ statusCode: number; body: string }> => {
  const { content } = JSON.parse(event.body) as { content?: string };
  const { userId } = event.pathParameters ?? {};

  if (userId === undefined || content === undefined) {
    return {
      statusCode: 400,
      body: 'bad request',
    };
  }

  const noteId = uuidv4();

  await client.send(
    new PutItemCommand({
      TableName: process.env.TABLE_NAME,
      Item: {
        PK: { S: userId },
        SK: { S: noteId },
        noteContent: { S: content },
      },
    }),
  );
  return {
    statusCode: 200,
    body: JSON.stringify({ noteId }),
  };
};
```

Take your time to understand the code:

- The handler is a function whose parameters are pathParameters and body. (based on the configuration the REST API)
- We extract a userId from the pathParameters and the content of the future note from the parsed body.
- We generate a unique noteId using the `uuid` library, it will be the SK of the note.
- We use the AWS SDK to send a PutItemCommand to the database.
  - The PK is "note" and the SK is the noteId (like you saw in the first schema of the article)
  - The noteContent is the content of the note, it is an additional key.
  - All keys are defined using the `S` type, an AWS-special syntax indicating that the stored value will be a string.
  - We use process.env.TABLE_NAME to provide the name of the table, which is defined in the environment variables of the Lambda function.
- Finally, we return the noteId to the client and a success status code, in order to be able to retrieve the note later.

Now, let's create the code for the `getNote` Lambda function. In a `getNote` folder, create a `handler.ts` file, and add the following code:

```typescript
import { DynamoDBClient, GetItemCommand } from '@aws-sdk/client-dynamodb';

const client = new DynamoDBClient({});

export const handler = async (event: {
  pathParameters: { userId?: string; id?: string };
}): Promise<{ statusCode: number; body: string }> => {
  const { userId, id: noteId } = event.pathParameters ?? {};

  if (userId === undefined || noteId === undefined) {
    return {
      statusCode: 400,
      body: 'bad request',
    };
  }

  const { Item } = await client.send(
    new GetItemCommand({
      TableName: process.env.TABLE_NAME,
      Key: {
        PK: { S: userId },
        SK: { S: noteId },
      },
    }),
  );

  if (Item === undefined) {
    return {
      statusCode: 404,
      body: 'not found',
    };
  }

  return {
    statusCode: 200,
    body: JSON.stringify({
      id: noteId,
      content: Item.noteContent.S,
    }),
  };
};
```

This time, the code is a bit simpler:

- We extract the userId and the noteId from the pathParameters, there is no body.
- We use the AWS SDK to send a GetItemCommand to the database.
  - Using the `Key` parameter, we get the item with `PK` equal to "note" and `SK` equal to `noteId`.
  - We also use process.env.TABLE_NAME to provide the name of the table.
- Finally, we return the noteContent of the item we retrieved from the database (using the `.S` syntax to get the string value).

And we are done with the code! 🎉

```bash
npm run cdk deploy
```

### Test the API

Now that the API is deployed, we can test it. To get the URL of the API, check my [last article][last article]. To test my new application, I first send a POST command to /notes/{userId}. I chose the userId "123", the response contains the noteId of the created note, in order to be able to retrieve it later.

![post request](./assets/createNote.png 'POST request')

I can now try to retrieve the note to be sure that it was correctly saved in the database. To do this, I send a GET request to /notes/{userId}/{noteId}.

![get request](./assets/getNote.png 'GET request')

Everything works as expected! 🎉

Finally, let's head to the AWS console to check that the data is correctly stored in the database.

![final database](./assets/finalDatabase.png 'final database')

The item is indeed stored in the database, and the noteContent has the correct value! You can try to create many more notes and retrieve them, and you will see that the data is correctly stored in the database.

## Homework 🤓

This application lacks a lot of features:

- We can't list all the notes of a user, if we lose the noteId we can't retrieve the note.
- We can't update or delete a note.
- Notes only have a content, we can't add a title or a date.

You should be able to implement these features by yourself, but if you need help, I will be happy to help you! You can contact me on [twitter][twitter account]. Some clues:

- You can use the `QueryCommand` to list all the notes of a user, and use KeyConditionExpression and ExpressionAttributeValues to filter the items whose PK is equal to the userId.
  ```typescript
  QueryCommand({
    KeyConditionExpression: 'PK = :userId',
    ExpressionAttributeValues: {
      ':userId': { S: userId },
    },
    TableName: process.env.TABLE_NAME,
  });
  ```
- You can use the `PutItemCommand` to update a note (Create and Update are the same in DynamoDB).
- You can use the `DeleteItemCommand` to delete a note, specifying the PK and SK, like in the `getNote` function.

## Conclusion

I plan to continue this series of articles on a bi-monthly basis. Last episode, I covered the creation of simple Lambda functions triggered by a REST API. I will cover new topics like file storage, creating event-driven applications, and more. If you have any suggestions, do not hesitate to contact me!

I would really appreciate if you could react and share this article with your friends and colleagues. It will help me a lot to grow my audience. Also, don't forget to subscribe to be updated when the next article comes out!

I you want to stay in touch here is my [twitter account][twitter account]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

[repository]: https://github.com/PChol22/learn-serverless
[twitter account]: https://twitter.com/PierreChollet22
[last article]: https://dev.to/kumo/dont-miss-on-the-cloud-revolution-learn-serverless-on-aws-the-right-way-1kac
[pay-per-use-ddb]: https://dev.to/kumo/aws-cdk-and-dynamodb-this-one-configuration-line-that-is-costing-you-hundreds-of-dollars-33kd
[dynamodb-docs]: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html
