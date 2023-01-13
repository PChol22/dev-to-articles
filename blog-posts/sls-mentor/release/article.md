---
published: false
title: 'Audit your AWS Serverless application with sls-mentor [titre WIP]'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/sls-mentor/release/assets/cover.png
description: ''
tags: serverless, AWS, audit, architecture
series: sls-mentor
canonical_url:
---

_This article is part of a series about [sls-mentor][sls-mentor], the new open-source tool allowing you to audit your AWS serverless infrastructure._

## The birth of a best-practices encyclopedia

I have been working for [Kumo][kumo] as a fullstack software engineer these last nine months. During this period, I discovered a technology that changed my perception of web development: AWS serverless. Serverless is great but AWS offers so many services, resources and possible configurations that it may sometimes be hard to find best practices.

At [Kumo][kumo], french developers like me have been using AWS serverless technologies to build applications for the last two years. We decided to create a new open source tool allowing us to organize and share every best practice we discovered during our coding journey. And this is how [sls-mentor][sls-mentor] was born 🐣!

sls-mentor is inspired by a [sls-dev-tools][sls-dev-tools] feature called Guardian (developed by our friends at [Aleios][aleios]), but our goal is to push the experience much further: we want everyone to be able to know, with a simple command, what can be improved on their app!

## Framework agnostic quality checks

To achieve our goal, we decided not to reinvent the wheel: [sls-mentor][sls-mentor] basically **works like a big cloud linter**, scanning your serverless resources configuration and assessing them against a set of rules. But it is also much more than that!

- ✅ Instead of parsing some complicated terraform or cloudformation template, we analyze your resources directly from the cloud: **we are framework-agnostic**.
- ✅ The set of rules is opinionated, but we bring with each rule the rational behind it, as well as a simple guide to comply with it: **we want to share our learnings**.
- ✅ Some frameworks generate resources with hard-to-modify configuration, other pieces of your app have reasons not to comply with every rule: **we made it easy to acknowledge warnings while tackling the actual issues**.

Enough about features, how about a quick demo instead ?

## Getting started with sls-mentor

Using sls-mentor is really easy. Just run this command in your terminal:

```sh
npx sls-mentor
```

It will analyze the resources associated with your `default` AWS profile. If you want to scan another profile, use the `-p <profile>` option. If there is no specified region in your credentials config, use the `-r <region>` option.

### Running sls-mentor on an example app

For the purpose of this article, I developed a small service allowing users to write, get and list blog articles. It is composed of three Lambda functions, one S3 bucket and one Api-Gateway.

Here is a quick architecture schema of the service I built:

![Architecture schema](./assets/architecture.png 'Architecture schema')

You can find the code behind it [here on github][example-code]. The underlying framework is a Typescript combination of serverless framework and cdk. I used the library [swarmion][swarmion] to bootstrap the project and have a nice repository ready for deployment, check them out! Anyway, the code and deployment method are not very important here, as sls-mentor is framework agnostic.

Let's run a first analysis ! I simply run the command in my terminal:

```sh
npx sls-mentor -c sls-mentor-release-core-dev
```

_Here, the -c parameter allows me to specify the name of one or many cloudformation stacks, in order to target the analysis on my new service. Filtering can also be achieved by tags, using the -t option_

![sls-mentor level choice](./assets/levels.png 'sls-mentor level choice')

I am instantly prompted with the choice of a level. Levels allow users to fix issues step-by-step, enabling more and more rules as they increase. Let's start easy with level 1.

### Passing sls-mentor level 1

The first level of sls-mentor features only four rules, that we consider have high impact on your application while being fairly easy to comply with.

![sls-mentor results](./assets/results-1.png 'sls-mentor results')

This is bad, isn't it ? No worries! Level 1 rules can easily be fixed, let's take a look at what went wrong.

- ✅ **Rule "No Mono Package"** passed for my three Lambda functions: this rule ensures that functions are bundled individually, which was already the case for me.
- ❌ **Rule "Use ARM Architecture"** didn't pass for any of my Lambda functions. This rule enforces the use of ARM64 processors instead of x86 ones, as they are faster, cheaper and consume less energy. I will fix this soon!
- ❌ **Rule "Server-side encryption enabled"** passed for my one of my two buckets. It was OK for my serverless-deployment bucket, but KO with my blog-articles bucket. It is important to enable this rule on buckets to ensure your data's safety.
- 😴 **Rule "use SecurityHeadersPolicy"** was skipped as there is no cloudfront distribution in my app yet.

Let's fix my Lambda functions! As I can read it in the documentation specified under every failing resource, I need to switch the processor of my functions from `x86_64` to `arm64`.

In my case, with the serverless framework, it as easy as changing the value of `provider.architecture` in the `serverless.ts` file!

```typescript
// serverless.ts file (~ serverless.yml)

const serverlessConfiguration = {
  service: 'sls-mentor-release-core',
  // ...
  provider: {
    architecture: 'arm64', // switch to arm64
    // ...
  },
  // ...
};

module.exports = serverlessConfiguration;
```

After this action, the analysis of sls-mentor is much better:

![sls-mentor results](./assets/results-2.png 'sls-mentor results')

The only task left to pass level 1 is to enable server-side encryption on my blog-articles bucket Using the AWS CDK, I need to specify in the Bucket construct I used to provision the bucket, that I want to enable encryption. I chose simple `S3_MANAGED` encryption to start.

```typescript
// Using the CDK

const exampleBucket = new Bucket(this, 'BlogArticles', {
  bucketName: 'sls-mentor-blog-articles',
  encryption: BucketEncryption.S3_MANAGED, // Enable encryption
});
```

Last level 1 run of sls-mentor! Everything is green! See, it was not too hard.

![sls-mentor results](./assets/results-3.png 'sls-mentor results')

Level 1 is complete, there are 4 other levels to conquer! To end this article, I will have a quick look at level 2, where I will teach you some advanced sls-mentor tricks...

### A look at sls-mentor level 2 + advanced sls-mentor configuration

Let's take a look at the level 2 results:

![sls-mentor results](./assets/results-4.png 'sls-mentor results')

It's not that bad! Level 2 includes level 1 so half of the work was already done, but some rules like "Lambda - Under max memory" are already passing everywhere.

Here, I want to focus on the rule "S3 - Use intelligent tiering". This rule enforces the usage of intelligent tiering in my buckets, it allows infrequently accessed objects to be stored in lower tiers, thus reducing their carbon footprint and cost.

On my application, I use CDK to deploy my bucket, so I will need to add a new lifecycle rule to my bucket definition, transitioning items to IntelligentTiering on their first day.

```typescript
// Using the CDK

const exampleBucket = new Bucket(this, 'BlogArticles', {
  bucketName: 'sls-mentor-blog-articles',
  encryption: BucketEncryption.S3_MANAGED,
  lifeCycleRules: [
    // add a lifecycle rule
    {
      id: 'auto-intelligent-tiering',
      enabled: true,
      transitions: [
        {
          // New objects transition to intelligent tiering on day 0
          storageClass: StorageClass.INTELLIGENT_TIERING,
          transitionAfter: Duration.days(0),
        },
      ],
    },
  ],
});
```

But there's a catch, on the dashboard, sls-mentor tells me that 2 buckets failed the check! In fact, the second one is my serverless deployment bucket, use during deployment by the Serverless framework. I do not have access to its configuration in my code.

We anticipated this issue: using a simple `sls-mentor.json` configuration file, I can define resources that won't be checked. In the case of my serverless deployment bucket, I just need to write the following lines in the file:

```json
{
  "rules": {
    "useIntelligentTiering": {
      "ignoredResources": ["serverlessdeployment"]
    }
  }
}
```

The ignoredResources array is composed of regex, so I just need to specify `serverlessdeployement` to avoid checking all resources with ARN matching this pattern.

![sls-mentor results](./assets/results-5.png 'sls-mentor results')

Everything is alright now! sls-mentor only checked one bucket (my blog-articles bucket), and it passed the check.

Always remember that **you should never blindly apply a rule**. We are aware that some resources have specific requirements, that sometimes contradict our rules. That's the aim of the configuration file feature (but please use it wisely and do not ignore 100% of your resources 🙏).

### Now it's your job to continue the process

I almost completed level 2, but I will stop my journey here. Frustrating, isn't it ? Now that you are sls-mentor professionals, it should be easy for you to activate SSL on the blog-articles bucket and finish the job 😉.

There are also levels 3, 4 and 5. Fixing issues in these levels becomes gradually more challenging, but there's nothing impossible! We also plan on adding new rules to sls-mentor, and create new levels of difficulty.

## Running sls-mentor as a periodic check

A good practice would be to run sls-mentor periodically in your CI. We have two recommendations:

- Run the job as a post-deploy job (after pull-request merge)
- Run the job as a periodic task (once a week for example)

Our levels feature allows you to keep your CI green by staying at a level you pass, until you decide to increase it. This way, you will ensure you are notified each time a new non-compliant resource is deployed.

### How to target a level on the CI ?

Just run the usual command with the `-l` parameter: it won't display the prompt.

```sh
npx sls-mentor -l 2
```

You can find some CI jobs examples on our [website][sls-mentor] and our [repository][github]!

## The future of sls-mentor

sls-mentor is currently maintained, **we would be glad to receive your contributions!** Feel free to create issues or pull requests to implement new rules or improve existing ones.

🎯 We are working on our documentation (the website is still WIP), to make it easier to understand and share our knowledge.

🎯 We plan on working on integrations like slack bots, to be notified of sls-mentor results.

🎯 And the most important: we want to feature more and more rules, supporting more and more AWS services!

[sls-mentor]: https://www.sls-mentor.pchol.fr
[kumo]: https://twitter.com/kumoserverless
[sls-dev-tools]: https://github.com/aleios-cloud/sls-dev-tools
[aleios]: https://www.aleios.com/
[example-code]: https://github.com/PChol22/sls-mentor-release-example
[swarmion]: https://www.swarmion.dev/
[github]: https://github.com/sls-mentor/sls-mentor
