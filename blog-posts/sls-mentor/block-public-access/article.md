---
published: true
title: 'Block public access on all your S3 Buckets easily'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/sls-mentor/block-public-access/assets/cover.png
tags: AWS, serverless, javascript, tutorial
series: sls-mentor
canonical_url:
---

_I publish articles at least twice a month, so if you are interested in serverless, AWS, or JavaScript, you can follow me on [Twitter][twitter] or DEV!_

One month ago, AWS sent an email to all their customers: **New S3 Buckets will have public access blocked by default. 🤯**

![AWS email](./assets/email.png 'AWS email')

This choice is due to security concerns: most use cases of S3 Buckets do not need public access, and it is easy to forget to block it. This is why AWS decided to block it by default.

However, **this change will not affect existing S3 Buckets**. If you want your existing applications to be up-to-date, you will have update "Block public access" configuration on all your S3 Buckets.

## List all your S3 Buckets without "Block public access"

To enable "Block public access" on all your S3 Buckets, you first need to list all your S3 Buckets without this option enabled.

Introducing [sls-mentor][sls-mentor]: a new open-source tool allowing you to audit your AWS serverless infrastructure. Like linters, it is based on rules, and **it implements a rule verifying that all your S3 Buckets have "Block public access" enabled**.

Simply run this command in your CLI:

```bash
npx sls-mentor@latest -p <your-aws-cli-profile>
```

It will list all your S3 Buckets without "Block public access" enabled! 🚀

![sls-mentor output 2](./assets/sls-mentor-rules.png 'sls-mentor output-2')

![sls-mentor output](./assets/sls-mentor-resources.png 'sls-mentor output')

Now you know where to update your infrastructure!

## How to enable "Block public access" on S3 Buckets

I am a user of [AWS CDK][aws-cdk], so I will show you how to enable "Block public access" on S3 Buckets that were created with it.

You only need to reach the part of the code where you create your S3 Bucket, and add the following line:

```typescript
import * as cdk from 'aws-cdk-lib';

new cdk.aws_s3.Bucket(this, 'MyBucket', {
  // Existing code

  blockPublicAccess: cdk.aws_s3.BlockPublicAccess.BLOCK_ALL, // New line !

  // Existing code
});
```

It's as simple as that! I showed you the TypeScript version, but CDK is available in many languages, and the syntax is quite similar every time.

## We are looking for contributors!

[sls-mentor][sls-mentor] is a new open-source tool, and we are looking for contributors to implement new rules like this one!

Feel free to check out our [GitHub repository][sls-mentor-github] and open a pull request or an issue! We have issues for all levels of experience, so don't hesitate to contribute!

_I publish articles at least twice a month, so if you are interested in serverless, AWS, or JavaScript, you can follow me on [Twitter][twitter] or DEV!_

[sls-mentor]: https://www.sls-mentor.dev
[sls-mentor-github]: https://github.com/sls-mentor/sls-mentor
[aws-cdk]: https://aws.amazon.com/cdk/
[twitter]: https://twitter.com/PierreChollet22
