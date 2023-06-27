---
published: true
title: 'Deploy a NEXT.js app for FREE on AWS with SST'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/sst/deploy-next/assets/cover.png
description: ''
tags: serverless, AWS, javascript, tutorial
canonical_url:
---

## TL;DR

**What is SST ?**

- SST is an IaC framework built on the AWS CDK, it allows you to deploy applications on AWS using TypeScript.
- SST integrates with OpenNext, an open-source implementation of Vercel's deployment system.
- This allows SST to deploy Next.js applications on your own AWS account, and to be in control of your infra!

**What will you learn in this article ?**

- In this article, learn in a few steps how to **deploy a Next.js application on AWS with SST**.
- _Bonus_: I will also show you how to **deploy your website on your own domain name**, using SST!

I post serverless content very regularly, follow me on [Twitter][twitter] to stay up to date!

## Why Next.js and SST?

Next.js is in fashion these days, many consider this framework to be the future of React. Next.js allows you to built static and server-side rendered applications with React, that load way quicker than usual SPAs, and guarantee a better SEO and user experience.

Traditionally, there were two main ways to deploy a Next.js application:

- On your own server, with a Node.js process running the application -> high entry cost ❌, tough to manage ❌, but you "own" your infrastructure ✅
- On Vercel, in a serverless way -> easy to use ✅, but zero control over the infrastructure ❌, and expensive for large applications ❌

Meet SST, a recent IaC framework built on top of the AWS CDK. SST allows you to deploy applications on AWS using TypeScript. SST is a great alternative to the older Serverless Framework, and it is much more powerful than the CDK alone.

One of SST's best features is its ability to integrate with OpenNext, an open-source implementation of Vercel's serverless deployment system. **This allows SST to deploy Next.js applications on your own AWS account**.

SST deploys Next.js apps in a serverless way on **your** AWS account which means that it's free for small applications ✅ and pay-per-use ✅. Icing on the cake: you are in control of your infrastructure ✅!

## Deploy a Next.js app on AWS in 2 simple steps

_Make sure you have Node 16+ installed on your machine, Node 18 is recommended!_

First, let's create a new Next.js app:

```bash
npx create-next-app@latest my-app
```

Follow the installation steps, and make sure you chose to use TypeScript (SST is built for TypeScript).

Then, head to the newly created folder and install SST.

```bash
cd my-app
npx create-sst@latest
npm install
```

When SST prompts you to setup in "Drop-in mode", answer yes: SST will automatically detect that you are using Next.js and will setup the project accordingly.

**You are basically done!** You can now deploy your Next.js app on AWS:

```bash
npx sst deploy
```

At the end of the deployment, SST prompts you with the public URL of your application:

![SST deployment end](./assets/sst-deploy-end.png 'public link of the deployed application')

Simply enter the link in your browser, and see that your website is on the world wide web! Now it's your turn to built the next killer app 🚀

![Cloudfront deployed site](./assets/cloudfront-site.png 'Site deployed on cloudfront')

What if you want your website to be deployed on your own domain name? Keep reading!

## Deploy your website on your own domain name

_⚠️ This part is optional and isn't free anymore, as you need to buy a domain name!_

_To buy your own domain name, you can use AWS Route 53 or any other domain name provider. See [my other article][cloudfront-article] if you need help._

As I said, SST is built on the AWS CDK, which means that you can integrate any resource from AWS (serverless or not) in your Next.js project. In this part, you will use AWS Route 53 and Amazon Certificate Manager to deploy your website on your own domain name.

First, let's see the file SST added to your project:

```typescript
// File: sst.config.ts

import { SSTConfig } from "sst";
import { NextjsSite } from "sst/constructs";

export default {
  config(_input) {
    return {
      name: "my-app",
      region: "us-east-1",
    };
  },
  stacks(app) {
    app.stack(function Site({ stack }) {
      const site = new NextjsSite(stack, "site");

      stack.addOutputs({
        SiteUrl: site.url,
      });
    });
  },
} satisfies SSTConfig;
```

This file instantiates a `Site` stack, composed of a `NextjsSite` construct. This construct is the one that deploys your Next.js application on AWS.

You can add any other AWS resource to your stack. In this case, you will create a hosted zone and a SSL certificate.

- A hosted zone is basically a DNS that will route from your domain name to the "real" URL of your website.
- A SSL certificate is required to deploy your website on a HTTPS endpoint.
- Deploy everything in the `us-east-1` region, as it is required for the SSL certificate to work.

The `sst.config.ts` file should look like this at the end:

```typescript
// File: sst.config.ts

import { SSTConfig } from "sst";
import { NextjsSite } from "sst/constructs";

import * as cdk from "aws-cdk-lib";

const ROOT_DOMAIN_NAME = "pchol.fr"; // Your domain name here
const DOMAIN_NAME = `sst.${ROOT_DOMAIN_NAME}`; // Any prefix you want, or just the root domain

export default {
  config(_input) {
    return {
      name: "my-app",
      region: "us-east-1", // Keep it that way
    };
  },
  stacks(app) {
    app.stack(function Site({ stack }) {
      // Create a hosted zone on your domain name
      const hostedZone = new cdk.aws_route53.HostedZone(stack, "HostedZone", {
        zoneName: ROOT_DOMAIN_NAME,
      });

      // Create a SSL certificate linked to the hosted zone
      const certificate = new cdk.aws_certificatemanager.Certificate(stack, "Certificate", {
        domainName: DOMAIN_NAME,
        validation: cdk.aws_certificatemanager.CertificateValidation.fromDns(hostedZone),
      });

      // Add the hosted zone and the certificate to the Next.js site
      const site = new NextjsSite(stack, "site", {
        customDomain: {
          domainName: DOMAIN_NAME,
          cdk: {
            hostedZone,
            certificate,
          }
        }
      });

      stack.addOutputs({
        SiteUrl: site.url,
      });
    });
  },
} satisfies SSTConfig;
```

You are done! Re-deploy your app and that's all! You can find the full code [here on github][github] if you need it.

```bash
npx sst deploy
```

⚠️ Only one small manual step: during the deployment, copy the 4 values in the NS record from your new hosted zone to your domain name provider. You can find them in the AWS console, in the Route 53 service, in the hosted zone you created. More details [here][cloudfront-article]!

_NS records in the new hosted zone_ ![NS records](./assets/records.png 'NS records')

_Copied into the domain name servers_ ![Name servers](./assets/name-server.png 'Name servers')

Head to your domain name (here [sst.pchol.fr](sst.pchol.fr)) and see that your website is now deployed on your own domain name!

![Route 53 deployed site](./assets/cloudfront-site.png 'Site deployed on route 53')

## Conclusion

This is only the beginning of your journey with SST and AWS! You can now integrate any resource, like S3 Buckets, DynamoDB tables or Lambda functions, in your Next.js app. This will allow you to develop advanced backend features!

Follow [this tutorial][sst tutorial] from the SST team if you want to learn more, in general, their documentation is great to get started!

If you want to go further, I wrote a [8 articles series on how to learn AWS][learn serverless], using the AWS CDK. Everything I covered can be integrated into your SST app, so feel free to check it out!

Finally, I would greatly appreciate if you follow me on [Twitter][twitter] and share this article if you liked it! I am open to all your questions and feedbacks if you need help!

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

[cloudfront-article]: https://dev.to/kumo/easily-deploy-your-portfolio-website-with-aws-cdk-4l9b
[sst tutorial]: https://docs.sst.dev/start/nextjs
[learn serverless]: https://dev.to/pchol22/series/22030
[twitter]: https://twitter.com/PierreChollet22
[github]: https://github.com/PChol22/next-sst
