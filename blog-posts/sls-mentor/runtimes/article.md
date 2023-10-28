---
published: true
title: 'Easily Find Deprecated Runtimes on Your Lambda Functions'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/sls-mentor/runtimes/assets/cover.png
description: ''
tags: serverless, AWS, tutorial, beginners
series: sls-mentor
canonical_url:
---

_This article is part of a series about [sls-mentor][sls-mentor], the new open-source tool allowing you to audit your AWS serverless infrastructure._

## What are deprecated runtimes?

AWS Lambda has been around for a while now. It is a great service as it allows to run code on the cloud and is compatible with many programming languages. However, as time goes by, these languages evolve and are getting new upgrades, leaving behind the old versions. This is why AWS Lambda has deprecated some of the runtimes that were available at the beginning. In this article, we will see how to find these deprecated runtimes on your Lambda functions.

Currently, the list of deprecated runtimes is the following:

- python3.6 ❌
- python2.7 ❌
- dotnetcore2.1 ❌
- ruby2.5 ❌
- nodejs10.x ❌
- nodejs8.10 ❌
- nodejs4.3 ❌
- nodejs6.10 ❌
- dotnetcore1.0 ❌
- dotnetcore2.0 ❌
- nodejs4.3-edge ❌
- nodejs ❌

If any of your lambda functions are using one of these runtimes, you should update them to a newer version, for security and performance reasons.

## How to find deprecated runtimes on your Lambda functions

But a problem persists: how do I simply now which Lambda functions are using these deprecated runtimes? This is where [sls-mentor][sls-mentor] comes in. It is a CLI tool that allows you to audit your AWS serverless infrastructure. It can be used to find deprecated runtimes on your Lambda functions.

[sls-mentor][sls-mentor] is a CLI tool that basically works like a big cloud linter. One of its rules detects for you whether a Lambda function is using a deprecated runtime or not.

To try sls-mentor, you only need to run it in your terminal using npx:

```bash
npx sls-mentor
```

sls-mentor does much more than that! It allows you to audit your AWS applications against a set of rules based on best-practices, and it already supports more that 10 services, including S3, CloudFront, API Gateway, Lambda, and more.

{% cta https://github.com/sls-mentor/sls-mentor %} Find sls-mentor on Github ⭐️ {% endcta %}

## Need help?

If you need help using the tool, check our [website][website] or [github][github]. You can also contact me on [twitter][twitter], I will be happy to help you!

If you need more information on deprecated runtimes, check out the [AWS documentation](https://docs.aws.amazon.com/lambda/latest/dg/runtime-support-policy.html).

[sls-mentor]: https://www.sls-mentor.dev/
[website]: https://www.sls-mentor.dev/
[github]: https://github.com/sls-mentor/sls-mentor
[twitter]: https://twitter.com/PierreChollet22
