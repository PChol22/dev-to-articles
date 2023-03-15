---
published: true
title: 'Learn serverless on AWS step-by-step - File storage'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/learn-serverless/storage/asset/cover.png
tags: AWS, serverless, javascript, tutorial
series: 'Learn serverless on AWS step-by-step'
canonical_url:
---

_In the [last article][last article], I covered the basics of interacting with DynamoDB databases, using the CDK. In this article, I will cover how to store data as files using Amazon S3. This series will continue! Follow me on [twitter][twitter account] or DEV to be notified when the next article is published!_

## Store files the serverless way with Amazon S3

Two weeks ago, I covered the basics of interacting with DynamoDB databases, these databases are great for storing structured data, composed of small items: the size limit for a single item is 400 KB. However, what if you want to store large files? In this case, you should use Amazon S3, a service that allows you to store files in the cloud.

Amazon S3 is very powerful, because it can store nearly infinite amounts of data, and stay available 99.999999999% of the time. It is also very cheap, because you only pay for the storage you use, and the amount of data you transfer.

Together, we are going to build a small dev.to clone, allowing users to publish, list and read articles. We will store the content of the articles, which can be quite large, in Amazon S3, and the metadata (article title, author, etc.) in DynamoDB: it will be a great review of the previous article!

In Amazon S3, files are stored in buckets. A bucket is a container for files, and it has a unique name. Each bucket can contain close to an infinite number of files. You can also create subfolders in a bucket, and store files in these subfolders.

Here is a small schema of the architecture we will build:

![Architecture schema](./asset/architecture.png 'architecture schema of the dev-to clone')

The application will contain 3 Lambda functions, a DynamoDB database, a S3 bucket and a REST API, interacting together.

## How to create a S3 Bucket with the CDK ?

Like the other articles in this series, we will use the AWS CDK to create our infrastructure. I will start from the project I worked on in the [last article][last article], and add the necessary code to create a S3 bucket. If you want to follow along, you can clone this [repository][repository] and checkout the `databases` branch.

Creating the bucket is very simple, you just need to add the following code to your `learn-serverless-stack.ts` file, into the constructor:

```typescript
// Previous code

export class LearnServerlessStack extends cdk.Stack {
  // Previous code
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Previous code
    const articlesBucket = new cdk.aws_s3.Bucket(this, 'articlesBucket', {});
  }
}
```

See, nothing complicated! Notice I didn't specify any name for the bucket, the CDK will automatically generate one for me. If you want to specify a name, you can do it by passing it as a parameter, but keep in mind that the name must be unique in the world, or the deployment will fail.

## Create a DynamoDB database and three Lambda functions

Let's create a DynamoDB table to store the metadata of our articles. We will also create three Lambda functions, one to create an article, one to list all the articles, and one to read a specific article.

```typescript
// Previous code

// Create the database
const articlesDatabase = new cdk.aws_dynamodb.Table(this, 'articlesDatabase', {
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

// Create the publishArticle Lambda function
const publishArticle = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'publishArticle', {
  entry: path.join(__dirname, 'publishArticle', 'handler.ts'),
  handler: 'handler',
  environment: {
    BUCKET_NAME: articlesBucket.bucketName,
    TABLE_NAME: articlesDatabase.tableName,
  },
});

// Create the listArticles Lambda function
const listArticles = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'listArticles', {
  entry: path.join(__dirname, 'listArticles', 'handler.ts'),
  handler: 'handler',
  environment: {
    BUCKET_NAME: articlesBucket.bucketName,
    TABLE_NAME: articlesDatabase.tableName,
  },
});

// Create the getArticle Lambda function
const getArticle = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'getArticle', {
  entry: path.join(__dirname, 'getArticle', 'handler.ts'),
  handler: 'handler',
  environment: {
    BUCKET_NAME: articlesBucket.bucketName,
  },
});
```

If you go back to the architecture schema, you can see that there are interactions between the Lambda functions and the S3 Bucket and DynamoDB table:

- The S3 Bucket interacts with `publishArticle` and `getArticle` -> I passed the name of my new bucket as an environment variable to the Lambda functions
- The DynamoDB table interacts with `publishArticle` and `listArticles` -> I passed the name of my new table as an environment variable to the Lambda functions

These environment variables will be used by the Lambda functions to interact with the S3 Bucket and the DynamoDB table.

## Grant permissions to the Lambda functions and create the REST API

Permissions are the base of security in AWS applications. If you don't give explicit permissions to your Lambda functions, they won't be able to interact with the S3 Bucket and the DynamoDB table. We will use the `grantRead` and `grantWrite` methods to give permissions to our Lambda functions.

```typescript
// Previous code
articlesBucket.grantWrite(publishArticle);
articlesDatabase.grantWriteData(publishArticle);

articlesDatabase.grantReadData(listArticles);

articlesBucket.grantRead(getArticle);
```

Finally, let's plug our Lambda functions to the REST API. Based on last articles, an API already exists, and we just need to add a new resource to it. We will create a new resource called `articles`, and add three methods to it: `POST`, `GET` and `GET /{id}`.

```typescript
// Previous code
const articlesResource = myFirstApi.root.addResource('articles');

articlesResource.addMethod('POST', new cdk.aws_apigateway.LambdaIntegration(publishArticle));
articlesResource.addMethod('GET', new cdk.aws_apigateway.LambdaIntegration(listArticles));
articlesResource.addResource('{id}').addMethod('GET', new cdk.aws_apigateway.LambdaIntegration(getArticle));
```

And we are done with the infrastructure! Last but not least, we have to write the code of our Lambda functions (the interesting part 😎).

## Interacting with a S3 Bucket and a DynamoDB table

### Publish an article

Let's start with the `publishArticle` Lambda function. This function will be called when a user wants to publish an article. It will receive the article's title, content and author from the request body, and will store the article in the S3 Bucket and its metadata in the DynamoDB table.

```typescript
import { DynamoDBClient, PutItemCommand } from '@aws-sdk/client-dynamodb';
import { PutObjectCommand, S3Client } from '@aws-sdk/client-s3';
import { v4 as uuidv4 } from 'uuid';

const dynamoDBClient = new DynamoDBClient({});
const s3Client = new S3Client({});

export const handler = async (event: { body: string }): Promise<{ statusCode: number; body: string }> => {
  // parse the request body
  const { title, content, author } = JSON.parse(event.body) as { title?: string; content?: string; author?: string };

  if (title === undefined || content === undefined || author === undefined) {
    return Promise.resolve({ statusCode: 400, body: 'Missing title or content' });
  }

  // generate a unique id for the article
  const id = uuidv4();

  // store the article metadata in the database PK = article, SK = ${id}
  await dynamoDBClient.send(
    new PutItemCommand({
      TableName: process.env.TABLE_NAME,
      Item: {
        PK: { S: `article` },
        SK: { S: id },
        title: { S: title },
        author: { S: title },
      },
    }),
  );

  // store the article content in the bucket
  await s3Client.send(
    new PutObjectCommand({
      Bucket: process.env.BUCKET_NAME,
      Key: id,
      Body: content,
    }),
  );

  // return the id of the article
  return { statusCode: 200, body: JSON.stringify({ id }) };
};
```

In the code above, I used the `@aws-sdk/client-dynamodb` and `@aws-sdk/client-s3` packages to interact with the DynamoDB table and the S3 Bucket. I used the `PutItemCommand` and `PutObjectCommand` to store the article metadata and content in the DynamoDB table and the S3 Bucket.

If you need refreshers on how to interact with DynamoDB, check my [last article][last article];

Note that I used the `process.env.TABLE_NAME` and `process.env.BUCKET_NAME` environment variables to get the name of the DynamoDB table and the S3 Bucket, these environment variables were set in the CDK previously!

### Get an article

The `getArticle` Lambda function will be called when a user wants to get an article. It will receive the article's id from the request path parameters, and will return the article's content stored in the S3 Bucket.

```typescript
import { GetObjectCommand, GetObjectCommandOutput, S3Client } from '@aws-sdk/client-s3';

const client = new S3Client({});

export const handler = async ({
  pathParameters: { id },
}: {
  pathParameters: { id: string };
}): Promise<{ statusCode: number; body: string }> => {
  let result: GetObjectCommandOutput | undefined;

  // get the article content from the bucket using the id as the key
  try {
    result = await client.send(
      new GetObjectCommand({
        Bucket: process.env.BUCKET_NAME,
        Key: id,
      }),
    );
  } catch {
    result = undefined;
  }

  if (result?.Body === undefined) {
    return { statusCode: 404, body: 'Article not found' };
  }

  // transform the body of the response to a string
  const content = await result.Body.transformToString();

  // return the article content
  return {
    statusCode: 200,
    body: JSON.stringify({ content }),
  };
};
```

In the code above, I used the `@aws-sdk/client-s3` package to interact with the S3 Bucket. I used the `GetObjectCommand` to get the article content from the S3 Bucket. The specified key id is the article's id (remember I used this id to create the object in PublishArticle).

The GetObjectCommand returns a `GetObjectCommandOutput` object, which contains the body of the response. If the request encounters no issues, I use the `transformToString` method to transform the body to a string and return it.

### List articles

The `listArticles` Lambda function will be called when a user wants to list all the articles. It will return the list of articles' metadata stored in the DynamoDB table. It won't have any parameters for now.

```typescript
import { DynamoDBClient, QueryCommand } from '@aws-sdk/client-dynamodb';

const client = new DynamoDBClient({});

export const handler = async (): Promise<{ statusCode: number; body: string }> => {
  // Query the list of the articles with the PK = 'article'
  const { Items } = await client.send(
    new QueryCommand({
      TableName: process.env.TABLE_NAME,
      KeyConditionExpression: 'PK = :pk',
      ExpressionAttributeValues: {
        ':pk': { S: 'article' },
      },
    }),
  );

  if (Items === undefined) {
    return { statusCode: 500, body: 'No articles found' };
  }

  // map the results (un-marshall the DynamoDB attributes)
  const articles = Items.map(item => ({
    id: item.SK?.S,
    title: item.title?.S,
    author: item.author?.S,
  }));

  // return the list of articles (title, id and author)
  return {
    statusCode: 200,
    body: JSON.stringify({ articles }),
  };
};
```

This Lambda function is less interesting as it's DynamoDB-only, I use a QueryCommand to list all the items of the table with the PK = 'article'. (remember that I set PK = 'article' and SK = '${id}' when I stored the article metadata in PublishArticle).

## Time to test our app!

First, you need to deploy your application. To do so, you need to run the following command:

```bash
npm run cdk deploy
```

Once you are done, head to Postman and test your new API!

Let's create an article with a POST call containing the right body :

![Create first article](./asset/create-first-article.png 'Create first article')

You can then list all the articles with a GET call:

![List articles (1 article)](./asset/first-list.png 'List articles (1 article)')

And finally, you can get the content of the article you just created with a GET call:

![Get first article](./asset/first-get.png 'Get first article')

But now, time to see the power of S3, let's create a second article with much, much, much more content:

![Create second article](./asset/create-second-article.png 'Create second article')

You can see that the list of articles updated automatically:

![List articles (2 articles)](./asset/second-list.png 'List articles (2 articles)')

And when you GET the content of the second article, you see that the payload is much more than 800kB!!! It would never fit inside a DynamoDB item! This is the power of S3!

![Get second article](./asset/second-get.png 'Get second article')

## Improve quality of our app

We just create a simple S3 Bucket using the minimal possible configuration. But we can improve the quality of our application by adding some features:

- We can register articles in the S3 Bucket with the "Intelligent-Tiering" storage class, which will automatically move the data to the "Infrequent Access" storage class when it's not been accessed for a while
- We can Block public access to our S3 Bucket, except when the Lambda function is the one accessing it
- We can enforce encryption on our S3 Bucket
- ...

To do so, let's update the configuration of our S3 Bucket:

```typescript
// Previously, we had:
const articlesBucket = new cdk.aws_s3.Bucket(this, 'articlesBucket', {});

// Now, we have:
const articlesBucket = new cdk.aws_s3.Bucket(this, 'articlesBucket', {
  lifecycleRules: [
    // Enable intelligent tiering
    {
      transitions: [
        {
          storageClass: cdk.aws_s3.StorageClass.INTELLIGENT_TIERING,
          transitionAfter: cdk.Duration.days(0),
        },
      ],
    },
  ],
  blockPublicAccess: cdk.aws_s3.BlockPublicAccess.BLOCK_ALL, // Enable block public access
  encryption: cdk.aws_s3.BucketEncryption.S3_MANAGED, // Enable encryption
});
```

There are many other small improvements that you can do, to this Bucket but also to your Lambdas or your API! I created a tool called [sls-mentor][sls-mentor] that will help you to improve the quality of your serverless application. It will check your code and your configuration and will give you recommendations to improve your application!

It is basically a big cloud linter, feel free to [check it out][sls-mentor-website] to learn more about AWS!

## Homework 🤓

We created a simple application to publish and read articles. But it's not perfect, there are some things that we can improve:

- We can't delete an article nor update it!
- There is no user management, anyone can publish an article and list all the articles. Based on my [last article][last article], you could use a userId as the PK of the articles table. This way, you could only list the articles of the user, and only the user could delete his articles.
- If you feel motivated, you could try to also store a cover images in S3 fir each article and return it when you GET an article.

What a program! I hope you enjoyed this article, and that you learned something new. If you have any questions, feel free to contact me!

Remember that you can find the code described in this article on my [repository][repository].

## Conclusion

I plan to continue this series of articles on a bi-monthly basis. I already covered the creation of simple lambda functions and REST APIs, as well as interacting with DynamoDB databases. You can follow this progress on my [repository][repository]! I will cover new topics like file storage, creating event-driven applications, and more. If you have any suggestions, do not hesitate to contact me!

I would really appreciate if you could react and share this article with your friends and colleagues. It will help me a lot to grow my audience. Also, don't forget to subscribe to be updated when the next article comes out!

I you want to stay in touch here is my [twitter account][twitter account]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

[repository]: https://github.com/PChol22/learn-serverless
[twitter account]: https://twitter.com/PierreChollet22
[last article]: https://dev.to/kumo/learn-serverless-on-aws-step-by-step-databases-kkg
[sls-mentor]: https://github.com/sls-mentor/sls-mentor
[sls-mentor-website]: https://www.sls-mentor.dev
