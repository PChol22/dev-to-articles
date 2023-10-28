---
published: true
title: 'Learn 30 serverless best-practices with sls-mentor'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/sls-mentor/reporting/assets/cover-image.png
description: ''
tags: serverless, aws, javascript, opensource
canonical_url:
---

## Your serverless app is not perfect (yet)

Are you currently learning serverless, or are you already an expert? Whatever... AWS offers so many services and possible configurations that **it is hard to keep track of all the best practices**.

With my team, we've been building serverless apps on AWS for several years now. We've learned a lot, and it seemed natural to us to **share our knowledge** with the community, with you!

Introducing [sls-mentor][website], a **free and open-source tool** that automatically analyzes your AWS Serverless application and gives you tips to improve it!

sls-mentor will **rate your application against 30 best practices**, and then assign you a score in each of the following categories:

- 🌳 Green IT 🌳
- 🛡 Security 🛡
- 🚀 Speed 🚀
- 💰 IT Costs 💰
- 💪 Stability 💪

{% cta https://github.com/sls-mentor/sls-mentor %} Find us on Github ⭐️ {% endcta %}

⬇️ I post serverless content very regularly, if you want more ⬇️

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

## How does sls-mentor work?

Nothing simpler! With your AWS credentials loaded in your CLI, run the following command:

```bash
npx sls-mentor@latest --report
```

_If you encounter errors when running the command, try specifying a profile with `-p` and a region with `-r`_

That's all!

_ℹ️ We require credentials with AdminReadOnly permissions. sls-mentor basically executes a bunch of `list` and `describe` API calls against your resources_

## The sls-mentor analysis

sls-mentor will then analyze your app directly from the cloud, and produce a super nice report like this one:

![sls-mentor report](./assets/sls-mentor.gif 'sls-mentor report')

_The report will be generated in your current directory, in a file called `.sls-mentor/index.html`_

Your app is assessed against 5 categories, and we give you 3 tips to quickly improve your score! For each tip, we give you an explanation of the problem, and a way to fix it.

Behind the scenes, sls-mentor is running a set of 30 rules that we've written. Want more details about what went wrong? The CLI will give you for each resource the list of rules that failed, and the reason why.

![sls-mentor CLI](./assets/detailed.png 'sls-mentor CLI')

_ℹ️ If you don't want a report, but just want to see the list of rules that failed, just remove the `--report` flag._

## What next? We need your help!

- Our reporting feature is quite new, we want to improve it! Things like service-wide stats, better recommendations are on our roadmap.
- sls-mentor lacks security rules related to IAM policies for example. We will add them soon!
- AWS is not only about serverless, if you have some _serverful_ knowledge to share with us, feel free to contribute!
- We've got existing issues waiting for contributors, and we are open to new ideas too! Feel free to join us [on GitHub][github]!

## Learn more about sls-mentor

See our [website][website] for more information, or check out our [GitHub repository][github]!

With the team, we already produced some articles featuring sls-mentor rules in depth. Feel free to check them out!

- [Rule UseARM][article-arm64], written by Zineb El Bachiri
- [Rule LightBundle][article-bundle-size], written by Eloi Alain
- [Rule EnableHTTPSOnS3][article-s3-https], written by Vincent Zanetta
- [Rule LimitedAmountOfVersions][article-versions], written by myself
- [Rule NoDeprecatedRuntimes][article-runtimes], written by myself
- [Rule BlockPublicAccess][article-block-public-access], written by myself

A big thanks to everyone I am working with on this project, especially Juliette, Marek, Quentin and Vincent!

{% cta https://github.com/sls-mentor/sls-mentor %} Find us on Github ⭐️ {% endcta %}

[article-versions]: https://dev.to/kumo/aws-lambda-versions-time-to-clean-up-guardian-is-watching-over-you-jkd
[article-bundle-size]: https://dev.to/kumo/aws-lambda-101-shave-that-bundle-down-48c7
[article-arm64]: https://dev.to/kumo/that-one-aws-lambda-hidden-configuration-that-will-make-you-a-hero-guardian-is-watching-over-you-5gi7
[article-s3-https]: https://dev.to/kumo/enable-https-only-on-your-s3-buckets-36eg
[article-runtimes]: https://dev.to/kumo/easily-find-deprecated-runtimes-on-your-lambda-functions-25b8
[article-block-public-access]: https://dev.to/kumo/block-public-access-on-all-your-s3-buckets-easily-o7b
[website]: https://sls-mentor.dev
[github]: https://github.com/sls-mentor/sls-mentor
