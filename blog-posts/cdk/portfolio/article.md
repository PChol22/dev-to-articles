---
published: true
title: 'Easily deploy your portfolio website with AWS CDK 🚀'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/cdk/portfolio/assets/cover-image.png
description: ''
tags: serverless, tutorial, frontend, aws
canonical_url:
---

## Why follow this tutorial?

Coding your own portfolio website is a cool introduction to HTML, CSS and JavaScript frameworks, but there is always a hurdle: once I am done, how do I deploy my website to the world wide web? Other question: how do I accomplish it using state of the art technologies, and following industry standards?

To reach this goal, I will use the AWS CDK (Cloud Development Kit) combined with TypeScript, to provision a 100% "Infrastructure as Code" CloudFront application serving my website all around the world. Here is a quick look of what the architecture of the app will look like at the end:

![CloudFront architecture](./assets/architecture.png 'CloudFront architecture')

I used this deployment method to deploy the website of [sls-mentor][sls-mentor], an open-source AWS serverless audit tool I've been working on these last months. I will go step by step explaining everything I did when I achieved this.

## Set-up a TypeScript CDK project

_> To use the CDK, you first have to configure an AWS profile in your CLI. The [official documentation][aws credentials doc] is very easy to follow if you need help._

Run these 2 commands in your CLI to setup your CDK project.

```sh
mkdir portfolio && cd portfolio
npx cdk init --language typescript
```

`cdk init --language typescript` creates a base TypeScript repository with the following structure.

```sh
portfolio
│   .gitignore
│   cdk.json
│   package.json
│   package-lock.json
│   ts-config.json
└───bin
│   └───portfolio.ts
└───lib
│   └───portfolio-stack.ts
└───node-modules
```

- `bin` folder contains the different stacks of your project. This is where I can set environment variables like the AWS accountId or region.
- `lib` folder contains the details of my stacks. I only have one, and it's where I will provision all the necessary resources to deploy my static website.
- `cdk.json` contains the global CDK configuration. For this simple use-case, there will be no need to modify it.

**⚠️ For certificate reasons, my CDK application must be deployed to region us-east-1 (the region is not important even if end users are not part of it, as we will see). To enable it, add the following snippet in bin/portfolio.ts**

```typescript
new PortfolioStack(app, 'PortfolioStack', {
  // To be added
  env: {
    region: 'us-east-1',
  },
  // End
});
```

To finish the CDK set-up, run the following command in your CLI:

```sh
cdk bootstrap
```

`cdk bootstrap` deploys on your AWS account the necessary resources to be able to later deploy your full app.

Last step: add the folder containing my website:

```sh
portfolio
...
└───front
│   └───src
│       └───index.html
│       └───styles.css
...
```

Here I use simple HTML+CSS but it can be the built files of a React or Angular project too!

## Upload your portfolio on the AWS cloud

First step to host my portfolio on AWS is to create a container hosting the website's files. The easiest solution is to use AWS S3 and to provision a new bucket containing these files.

I need to specify the path of the base html file of my website (`index.html`). It is also one of the rare cases where I need to set `publicReadAccess` to `true` because the content is public (I want everyone to have access to my portfolio).

I have another constraint: I want the content of this bucket to always contain the latest version of the website's source code. CDK offers a construct named `BucketDeployment`, that replaces the content of the bucket with the files found at a specified location (`../front/src`) of my repository at each deployment.

![phase 1](./assets/phase1.png 'phase 1')

Added together, the stack definition of `PortfolioStack`, in the file `lib/portfolio-stack.ts` should look like this:

```typescript
export class PortfolioStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // Path of the folder containing the built website
    const FRONT_SRC_PATH = path.join(__dirname, '../front/src');

    // Create the bucket hosting the website
    const staticWebsiteHostingBucket = new Bucket(this, 'StaticWebsiteHostingBucket', {
      publicReadAccess: true,
      websiteIndexDocument: 'index.html',
    });

    // Sync the bucket's content with the codebase
    new BucketDeployment(this, 'StaticWebsiteHostingBucketSync', {
      sources: [Source.asset(FRONT_SRC_PATH)],
      destinationBucket: staticWebsiteHostingBucket,
    });
  }
}
```

Now, let's run a first deployment with the CLI command:

```sh
cdk deploy
```

It will create the bucket and fill it with the index.html file found in my local repository. I can see it in my AWS console!

![bucket content](./assets/bucket.png 'bucket content')

## Access the portfolio from a custom domain name

My website is now uploaded on the AWS cloud, but it would be nicer if anyone could access it using my domain name (sls-mentor.dev).

If you do not own a domain name, AWS Route 53 offers the possibility to buy ones for a fair price. In my example, `.dev` domains was not available on AWS, so I bought it on [gandi][gandi], but you can buy your domain name anywhere you want. The important thing is to have access to the name servers after it's yours.

How to link my S3 bucket with `sls-mentor.dev`? It is done in 4 simple steps:

- Create a Route 53 hosted zone that will contain the custom DNS records needed to create this connection.
- Create a HTTPS certificate, otherwise, my portfolio will only be accessible via http.
- Create a CloudFront distribution, allowing to distribute my website all around the world, and to use my new HTTPS certificate.
- Create DNS records in my hosted zone, to redirect traffic from `sls-mentor.dev` and `www.sls-mentor.dev` to the CloudFront distribution.

![phase 2](./assets/phase2.png 'phase 2')

Seems too complicated ? Using CloudFront allows to use the HTTPS protocol, and to deliver my website very quickly around the world thanks to caching. Furthermore, these 4 steps can be coded in a few lines thanks to the CDK, everything left to do is to launch the deployment!

Let's create the hosted zone, by adding this snippet of code under the S3 bucket definition:

```typescript
const DOMAIN_NAME = 'sls-mentor.dev'; // your domain name

// Create a Route53 hosted zone to later create DNS records
// ⚠️ Manual action required: when the hosted zone was created, copy its NS records into your domain's name servers
const hostedZone = new HostedZone(this, 'DomainHostedZone', {
  zoneName: DOMAIN_NAME,
});
```

Then deploy this change to create the hosted zone.

```sh
cdk deploy
```

I have to perform **the only manual input required in this tutorial (⚠️ very important ⚠️)**: after deployment, Route 53 will create a `NS` record containing 4 values in my new hosted zone, I have to copy them into my DNS's name servers (in order to prove my ownership of the domain).

_NS records in the new hosted zone_ ![NS records](./assets/records.png 'NS records')

_Copied into the domain name servers on gandi_ ![Name servers](./assets/name-server.png 'Name servers')

On the screenshots, I did it using a domain name bought on [gandi][gandi]. But it can also be achieved on providers like AWS or any other one.

Now that I'm done with the domain, I can create the certificate, and set the validation method to `fromDns`. It will automatically communicate with my freshly created hosted zone to validate the authenticity of the certificate.

_⚠️ The certificate MUST be created in the region us-east-1 to be compatible with CloudFront, that's why I deploy all my resources in this region. CloudFront allows for "at edge" delivery, which means that it's not a problem if my visitors are not in the US._

```typescript
const WWW_DOMAIN_NAME = `www.${DOMAIN_NAME}`;

// Create the HTTPS certificate (⚠️ must be in region us-east-1 ⚠️)
const httpsCertificate = new Certificate(this, 'HttpsCertificate', {
  domainName: DOMAIN_NAME,
  subjectAlternativeNames: [WWW_DOMAIN_NAME],
  validation: CertificateValidation.fromDns(hostedZone),
});
```

Then, time for the CloudFront distribution. It is the main node of my architecture:

- It communicates with the certificate I just created to allow HTTPS communication.
- It is the link between the us-east-1 bucket and users all around the world: thanks to caching, there will be minimal latency for end users.

I specify my bucket as the origin, enable a REDIRECT_TO_HTTPS policy, set the domain names (with the addition of the www subdomain for optimal compatibility with all browsers), and finally reference my new https certificate.

I also specified an optional responseHeadersPolicy, with the ID corresponding to the AWS `Managed-SecurityHeadersPolicy`. It's a quick win to improve security in my distribution. (This best practice is part of [sls-mentor][sls-mentor], feel free to check it out!)

```typescript
// Create the CloudFront distribution linked to the website hosting bucket and the HTTPS certificate
const cloudFrontDistribution = new Distribution(this, 'CloudFrontDistribution', {
  defaultBehavior: {
    origin: new S3Origin(staticWebsiteHostingBucket, {}),
    viewerProtocolPolicy: ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    responseHeadersPolicy: { responseHeadersPolicyId: '67f7725c-6f97-4210-82d7-5512b31e9d03' },
  },
  domainNames: [DOMAIN_NAME, WWW_DOMAIN_NAME],
  certificate: httpsCertificate,
});
```

Last step, redirect requests heading to sls-mentor.dev and www.sls-mentor.dev to the CloudFront distribution. I create 2 DNS records in my hosted zone, which have the CloudFront distribution as target. These records are "A" records, linking my domain name with the IPv4 address of the CloudFront distribution.

```typescript
// Add DNS records to the hosted zone to redirect from the domain name to the CloudFront distribution
new ARecord(this, 'CloudFrontRedirect', {
  zone: hostedZone,
  target: RecordTarget.fromAlias(new CloudFrontTarget(cloudFrontDistribution)),
  recordName: DOMAIN_NAME,
});

// Same from www. sub-domain
new ARecord(this, 'CloudFrontWWWRedirect', {
  zone: hostedZone,
  target: RecordTarget.fromAlias(new CloudFrontTarget(cloudFrontDistribution)),
  recordName: WWW_DOMAIN_NAME,
});
```

And I am done! Last step is to deploy everything one more time.

```sh
cdk deploy
```

_Be aware that changes to DNS records can take up to 24 hours to be propagated, do not worry if the deployment is successful but your website doesn't work instantly_

Time to test it! These 4 links should all redirect to the same website, always using the https protocol, regardless of what I specify:

- http://sls-mentor.dev
- http://www.sls-mentor.dev
- https://sls-mentor.dev
- https://www.sls-mentor.dev

[aws credentials doc]: https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/setup-credentials.html
[gandi]: https://www.gandi.net
[sls-mentor]: https://www.sls-mentor.dev
[github repository]: https://github.com/PChol22/cdk-portfolio
