---
published: false
title: 'Our team migrated from Serverless Framework to CDK. Here is How'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/migration-cdk/why-migrate/assets/cover.png
description: ''
tags: serverless, AWS, javascript, discuss
canonical_url:
---

Pros of serverless framework:

- Quick bootstrapping
- Beginner friendly
- Easy to deploy lambdas

Cons of serverless framework:

- Less and less maintained
- Team not very confident with v4 and new business model
- Doesn't scale well
- Tired of magic strings and magic resources (api ...)
- Painful to integrate lambda functions with other resources (dynamodb, s3, ...)

Pros of CDK:

- Pushed by AWS
- More scalable (substacks...)
- Not "lambda-centric"
- Ability to define our IAC based on domain / business logic, not technical constraints (higher order constructs...)

Cons of CDK:

- Boilerplate
- Migration from serverless is hard

**Existing situation:**

- Typescript project, deployed on AWS
- IAC is a mix of serverless framework (lambdas + api), and CDK (stateful resources)

**Why did we go with serverless framework in the first place?**

- Painful to deploy lambda functions using only CDK 3 years ago
- Sls was very useful to get a MVP running quickly
- Sls was the most popular and maintained framework at the time

**Why did we change our mind?**

- CDK released the NodeJsFunction construct, which makes it very easy to deploy lambda functions
- Our project reached a critical mass and working with sub-stacks using sls-framework was painful
- We encountered issues using the sls-esbuild plugin
- Many of our issues / PRs were ignored because V3 is not actively maintained anymore
- Last announces from the serverless framework team were not very encouraging (V4 etc...)

**How did we address the cons of CDK?**

- Boilerplate: We created a library of higher order constructs, integrating with api and eventbridge contracts that we were already using (see swarmion)
- Migration: We started working on a two steps migration plan, first moving stateful resources to CDK, then moving lambdas. We invested time in creating a migration script

**More details on the migration plan:**

- I'm releasing a blog post on the migration plan soon

**Need help with your migration?**

Feel free to contact me, I've acquired a ton of experience and knowledge on this topic and I'm happy to help!

I you want to stay in touch here is my [twitter account][twitter]. I often post or re-post interesting stuff about AWS and serverless, feel free to follow me!

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

[twitter]: https://twitter.com/PierreChollet22
