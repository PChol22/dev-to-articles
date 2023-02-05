---
published: true
title: 'Learn serverless on AWS step-by-step - Lambda functions'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/learn-serverless/introduction/assets/cover-img.png
tags: AWS, serverless, javascript, tutorial
series: 'Learn serverless on AWS step-by-step'
canonical_url:
---

_In this article, we will learn how to get started with serverless on AWS. It will require to create an AWS account and a credit card, but don't worry, it's free to start!_

## TL;DR

In this article, I teach you how to get started with serverless on AWS. If you already have an AWS account, feel free to skip to the [Create an AWS CDK project](#create-an-aws-cdk-project) section. I plan to release a series of articles about serverless, to learn step-by-step the specificities of this technology, so stay tuned!

## Why serverless?

A massive gold rush towards cloud technologies has begun during these last years. The cloud is becoming the new standard for software development. It is a revolution that is changing the way we develop software, and it is far from being over. As developers, we have an important role to play is this transformation, and it is an opportunity for us to claim our place in this new paradigm.

When using the cloud, we think less about the infrastructure of our servers. If used properly, it allows us to focus of the business logic of the features we build: the spotlight is on the code we write, not on the servers we run it on. This is a huge advantage for developers, as it allows us to focus on what we do best: writing code.

**Serverless** is the pinnacle of this cloud revolution: servers are gone, developers become architects, and when paired with powerful IaC (Infrastructure as Code) tools, it allows us to build complex architectures in a matter of minutes. This revolution is still in its infancy, and in my opinion, NOW is the time to get on the train!

## Serverless and infrastructure as code

Serverless is a cloud computing execution model where the cloud provider dynamically manages the allocation and provisioning of servers. You bring some code, and it is ran on demand on the cloud, as many times as there are users. Its principal argument is that it's pay-per-use: applications scale to zero and are basically **free** to develop until there are users.

Serverless is a very broad term, and it can be applied to many different cloud providers. In this article, we will focus on AWS, as it is the most popular cloud provider, and it is the one I am most familiar with.

Infrastructure as code (IaC) is a way to manage your infrastructure using code. It allows you to automate the creation, modification, and deletion of your infrastructure. Paired with serverless, it allows you to build complex architectures using your favorite programming language, without the need of complex configuration files.

## Getting started on AWS

### Create an AWS account and configure your credentials

To get started, you will need to create an AWS account. You can do so by going to [https://aws.amazon.com](https://aws.amazon.com/). You will need to provide a credit card, but don't worry, until you have a large number of users, you won't be charged anything.

Once you have created your account, you will need to configure your credentials. You can do so by going to the [AWS console](https://console.aws.amazon.com/), and clicking on your username in the top right corner. Then, click on "My Security Credentials". You will then be able to create a new access key. You will need to save this key somewhere **safe**, as it will be used to authenticate your requests to AWS.

![access-keys](./assets/access-key.png 'access keys')

⚠️ It is best practice to not use the root account for development, as it has full access to all AWS services. Instead, you should create a new user with the least amount of permissions possible, check this [tutorial](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) for more information.

### Install the AWS CLI

The idea behind serverless is to be able to develop the business logic of your application locally on your computer, while deploying it on the cloud once everything is ready (you don't want to code directly in google chrome 🙃). You will need a CLI to do so.

The AWS CLI is a tool that allows you to interact with AWS using the command line. You can install it by following the instructions on [this page](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).

Once you are done, you can configure a profile by running the following command:

```sh
aws configure
```

Provide the access key that you generated earlier, specify your AWS region (eu-west-1 in europe for example), and the output format (json).

![profile](./assets/profile.png 'profile')

### Create an AWS CDK project

The [AWS Cloud Development Kit (CDK)][aws-cdk] is an open-source software development framework used to provision your cloud infrastructure. Basically, CDK allows you to describe with code the infrastructure you want to deploy on AWS, it is called Infrastructure as Code (IaC). CDK supports many programming languages, I will focus on TypeScript in this article, but JavaScript for example works almost the same way.

To create your project, run the following command in your CLI:

```sh
mkdir my-first-app && cd my-first-app
npx cdk init app --language typescript
npm run cdk bootstrap
```

It will create a new repository, install the dependencies, and bootstrap your project on the AWS cloud. And you are DONE with the setup! Time to start coding!

[aws-cdk]: https://aws.amazon.com/cdk/

## Lambda functions

Lambda functions are the core of serverless. They are small architectural blocks that can execute code on the cloud on demand. They are tiny servers that are provisioned on demand to answer to user requests. In theory, a single Lambda function can answer to millions of requests per second! Best fact: they are pay-per-use, so you only pay for the time they are running (it is basically free while your app is in development).

In this part, you are going to create your first Lambda function, and deploy it on AWS. It will be a simple "Hello World" function, but enough to get started.

Let's have a look at the code that was generated by the CDK. You will find it in the `lib` folder. You will see that there is a file called `my-first-app-stack.ts`. This is the file where you will write your infrastructure as code. Let's start by creating a simple Lambda function.

```ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import path from 'path';

export class MyFirstAppClass extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new cdk.aws_lambda_nodejs.NodejsFunction(this, 'myFirstLambda', {
      entry: path.join(__dirname, 'myFirstLambda', 'handler.ts'),
      handler: 'handler',
    });
  }
}
```

_If you encounter an error with the import of 'path' add `esModuleInterop: true` in the compilerOptions of your tsconfig.json file._

In this code snippet, I create a new Lambda function called `myFirstLambda` using the `NodejsFunction` construct. This construct allows to easily create a Lambda function using Node.js, taking care of packaging for you (you won't have to worry about importing dependencies etc...), it also supports TypeScript natively!

The `entry` parameter is the path to the handler file (defining the code that will be executed on the cloud). The `handler` parameter is the name of the JavaScript function that will be executed when the Lambda is invoked.

This means that you will need to create a folder called `myFirstLambda` in the `lib` folder, and a file called `handler.ts` in it. This file should export a `handler`function. For example, the content of this file could be the following:

```ts
export const handler = (): Promise<string> => Promise.resolve('Hello World!');
```

A simple rule of thumbs is that the code executed by Lambda functions always must be asynchronous (in reality it's a bit more complicated, but for now, remember their return type should be a Promise). Here, I use Promise.resolve to return a string asynchronously, even if everything is synchronous in this example.

```sh
npm run cdk deploy
```

OOPS! An error occurs! I forgot to add `esbuild` as a dev-dependency. Esbuild is the tool used to package the NodejsFunction. I just need to run the following command to fix it:

```sh
npm i --save-dev esbuild
```

Let's re-run the deploy command. It should work fine now. On the AWS console, I can see that a new Lambda function has been created:

![lambda](./assets/firstLambda.png 'firstLambda')

Click on it and head to the `test` tab. You can invoke the Lambda function by clicking on the `Test` button.

![test](./assets/firstLambdaExecution.png 'firstLambdaExecution')

In green you can see the logs of the invocation: "Hello World!". You just created your first Lambda function!

![test](./assets/firstLambdaResults.png 'firstLambdaExecution')

## A first look at AWS API Gateway

Your first Lambda function is nice, but it's not very useful. Nobody can access it from the internet and it doesn't do anything useful.

To fix this, you will need an API, like in classic "serverful" applications. The AWS API Gateway services provides functionalities to create REST APIs that can invoke Lambda functions -> it is everything needed for now!

Let's fix that by creating a new Lambda function plugged an API endpoint that will invoke it. You will use Api Gateway and the CDK for that.

First, let's create a REST API that will be used to invoke our Lambda function. To do so, we will use the `RestApi` construct. Add the following code to your `my-first-app-stack.ts` file:

```ts
// Under the instantiation of your first Lambda function

const myFirstApi = new cdk.aws_apigateway.RestApi(this, 'myFirstApi', {});
const diceResource = myFirstApi.root.addResource('dice');
```

I also added a resource called `dice` to the API. It basically tells AWS our API now has a `/dice` API route registered. For example, if the API is deployed on `https://my-api.com`, the endpoint will be `https://my-api.com/dice`.

Let's create a `rollADice` Lambda function that will return a random number between 1 and 6. Create a new folder called `rollADice` in the `lib` folder, and a file called `handler.ts` in it. The content of this file should be the following:

```ts
export const handler = async (): Promise<{ statusCode: number; body: number }> => {
  const randomNumber = Math.floor(Math.random() * 6) + 1;

  return Promise.resolve({ statusCode: 200, body: randomNumber });
};
```

Notice the return type of the function. It is a Promise of an object with a `statusCode` and a `body`. This is the format expected by API Gateway to be returned by a Lambda function.

Finally, let's associate this source code to a Lambda function. Add the following code to your `my-first-app-stack.ts` file:

```ts
// Under the instantiation of the API resource

const rollADiceFunction = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'rollADiceFunction', {
  entry: path.join(__dirname, 'rollADice', 'handler.ts'),
  handler: 'handler',
});

diceResource.addMethod('GET', new cdk.aws_apigateway.LambdaIntegration(rollADiceFunction));
```

At the same time, I created a new Lambda function, and associated it's execution with the route `/dice` and the http method `GET` (`POST`, `PUT` etc... are also available).

Time to re-deploy the infrastructure! On the AWS console, you should see your new API Gateway:

![apiGateway](./assets/firstApi.png 'apiGateway')

Click on it and head to the `stages` tab. You can see the URL of your API.

![apiGateway](./assets/firstApiUrl.png 'apiGateway')

Copy it and paste it in your browser or in postman, and add /dice at the end. You should see a number between 1 and 6 in the body of the response!

![apiGateway](./assets/firstApiCall.png 'apiGateway')

If you pay attention to the delay of the first response, you will notice that it is quite long (~100-200ms). This is because the Lambda function is cold. It means that it has not been invoked for a while, and the AWS infrastructure needs to spin up a new instance of the Lambda function to execute it. New executions will be faster, in fact, you will feel like the Lambda function is running on your computer!

## Homework 🤓

Now that you have a basic understanding of how to create a Lambda function and an API endpoint, it's time to create your own Lambda function and API endpoint. My proposition is to create a Lambda that rolls any number of dices, and returns the result. For example, if you call the endpoint `GET /dice/3`, it should return a random number between 3 and 18.

My only tip is how to use path parameters in API Gateway. Like we did on `root` to create a first resource, add a resource to `diceResource` with `'{diceCount}'` as value.

This will create a new route with a path parameter called `diceCount` that you can use in your Lambda function. Just create a new Lambda and use the type `{ pathParameters: { diceCount } }: { pathParameters: { nbOfDices: string } }` as the input of your new handler. The rest should be easy to figure out!

Do not hesitate to ask for help if you get stuck! I will be happy to help you! You can also check this [repository] on my github to see the solution, and the previous two Lambda functions we created together.

## Conclusion

I plan to continue this series of articles on a bi-monthly basis. I will cover new topics like storing data in a database, creating event-driven applications, and more. If you have any suggestions, do not hesitate to contact me!

I would really appreciate if you could react and share this article with your friends and colleagues. It will help me a lot to grow my audience. Also, don't forget to subscribe to be updated when the next article comes out!

I you want to stay in touch here is my [twitter account].

[repository]: https://github.com/PChol22/learn-serverless
[twitter account]: https://twitter.com/PierreChollet22
