---
published: true
title: 'AWS Lambda Versions : Time to clean up! - Guardian is watching over you'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/guardian/lambda-version/assets/cover_image.png
description: ''
tags: serverless, lambda, quality, AWS
series:
canonical_url:
---

_This article is part of a series on [Guardian][guardian], an open-source, highly configurable, automated best-practice audit tool for AWS serverless architectures._

# Let's keep track

Lambda versioning is great and might come in handy, but do you really need to keep dozens of outdated copies of your functions' code ? Here are two reasons why you should get rid of them.

## 1. Less is more

It sure can help to have a backup version of your code to rollback to in case of an emergency, but the day it happens, you will feel a lot more comfortable only having to chose between two or three documented, aliased versions than being overwhelmed by dozens of anonymous and forgotten ones.

![Button meme](./assets/button_meme.png 'Button Meme')

## 2. Beware of AWS Lambda quotas

AWS Lambda enforces a regional [75 GB limit][quotas] for all your uploaded packages. This threshold might seem hard to reach and harmless, but it takes into account every version of every Lambda you have deployed: dozens of versions per function multiplied by a large bundle size (see our [previous article][guardian-bundle-size-article]) will take you to over this threshold in no time.

It is possible to raise this soft limit by contacting the support, but it's not a sustainable workaround, and you don't want to be stuck unable to upload your new version when there's an important fix to deploy.

# Easily monitor your Lambdas versioning with Guardian 🛡️

[Guardian][guardian] now offers a **new rule** preventing your lambdas versions counter from going crazy.

![Rule in CI](./assets/rule_CI.png 'Rule in CI')

Guardian also comes with **many other rules** to help you make the best decisions for your Serverless project. It will help you identify where your deployed resources can be optimized to achieve better performance at a lower cost.

# Guardian how-to

```
npm install @kumo-by-theodo/guardian
npx guardian -p <your_aws_profile> -c <your_stack_name>
```

Guardian is available on [NPM][npm-registry]. You will find instructions to use Guardian in your CI.

# See also

There also many other tools helping you manage your Lambdas versioning. One of them is [Serverless Prune Plugin][serverless-prune-plugin], which integrates with the Serverless Framework and allows you to only keep a certain amount of recent versions for each of your functions.

To go deeper into AWS Lambda deployment quotas and how to deal with them, check out [this article from Yan Cui][quotas-article].

[guardian]: https://github.com/Kumo-by-Theodo/guardian
[quotas]: https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html
[serverless-prune-plugin]: https://www.serverless.com/plugins/serverless-prune-plugin
[guardian-bundle-size-article]: https://dev.to/kumo/aws-lambda-101-shave-that-bundle-down-48c7
[quotas-article]: https://hackernoon.com/mind-the-75gb-limit-on-aws-lambda-deployment-packages-163b93c8eb72
[npm-registry]: https://www.npmjs.com/package/@kumo-by-theodo/guardian
