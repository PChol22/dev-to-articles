---
published: true
title: 'From Zero to Hero... Send AWS SES Emails Like a Pro!'
cover_image: https://raw.githubusercontent.com/pchol22/kumo-articles/master/blog-posts/emails/assets/cover.png
tags: serverless, AWS, SES, Emails
series:
canonical_url:
---

_This article assumes the reader has basic knowledge of AWS SES (Simple Email Service), like being able to send simple emails using SES and Lambda or to verify an identity._

## TL;DR

This article is structured in three independent parts : three problems and their solutions. If you have a short timing, cover what seems the most important to you first!

- [Handling AWS sender reputation](#1-maintain-your-sender-reputation-with-aws)
- [Avoiding to end up in spams](#2-make-sure-you-dont-end-up-in-your-users-spams)
- [Designing responsive emails that display well on every mail client](#3-design-responsive-emails-that-look-good-in-any-mail-client)

⬇️ I post serverless content very regularly, if you want more ⬇️

{% cta https://twitter.com/PierreChollet22 %} Follow me on twitter 🚀 {% endcta %}

_**Quick announcement:** I also work on a library called [🛡 sls-mentor 🛡][sls-mentor]. It is a compilation of 30 serverless best-practices, that are automatically checked on your AWS serverless projects (no matter the framework). It is free and open source, feel free to check it out!_

{% cta https://github.com/sls-mentor/sls-mentor %} Find sls-mentor on Github ⭐️ {% endcta %}

## ✉️ AWS SES ✉️

As you may already know, AWS SES offers a great way to send emails with your AWS serverless app. Coupled with the AWS SDK and Lambda functions, it allows you to programmatically send emails to your users with minimal setup. However, this ease of use comes at the price of potential mistakes and pitfalls that can happen at every step of your coding journey. Let me guide you through three of these issues and help you design beautiful emails that never end up in your users spam.

## 🚧 🚧 🚧 Three obstacles on your road to clean email communication

A few months ago, I went through the implementation of transactional emails sending on the project I was working on, and identified three major pitfalls that every developer should be aware of (these pitfalls are also relevant when sending promotional emails!).

Using only **minimal configuration** to send your emails:

- 🚧 You will be prevented from sending messages if your sender reputation goes down.
- 🚧 Your emails will probably end up in some of your users spams.
- 🚧 Your emails will probably look bad on some clients (⚠️ gmail is one of them!), or not display at all in some cases, for security reasons.

This article tries to bring solutions to these three problems, that are simple and easy to implement. There is a lot of content, and no specific order, if one of the issues is more important than the others, do not hesitate to read it's part first and tackle it quick!

## 1. Maintain your sender reputation with AWS

First step on your way to success is to make sure that your emails are even sent to your recipients. It may seem trivial but AWS enforces strict rules that may affect the **sender reputation** attached to your domain. In order to monitor this sender reputation, two metrics are mostly used :

- **Bounces:** soft bounces are events happening because of a temporary issue (like **recipient mailbox full**), hard bounces happen because of permanent ones (like a **non-existing recipient address**).

  - 🧐 Over 5% bounce rate, your SES account will be placed under review by AWS.
  - ❌ Over **10%** bounce rate, you will be **prevented from sending emails** until investigation.

- **Complaints:** events that happen when your recipient manually **reports your emails as undesired** (basically spam).

  - 🧐 Over 0.1% complaint rate, your SES account will be placed under review by AWS.
  - ❌ Over **0.5%** complaint rate, you will be **prevented from sending emails** until investigation.

Obviously, you don't want your account to be blacklisted, but I would also advise you to act before being put under review. To prevent all this bad stuff from happening to you, here are two of my personal tips:

### Set up alarms to be warned before its to late

Cloudwatch alarms can be set up to monitor both the bounce and complaint rates, and send you notifications / take actions when they reach a dangerous level. These cloudwatch alarms can be set up at **account level** or **configuration set level**. Allowing you to either monitor your whole accounts reputation, or the reputation linked to specific SES identities.

Let me advise you to **create a configuration set linked to the SES Identity you send emails from**, and monitor it specifically. It will allow you to identify faster where problems are coming from, and possibly shut down only parts of your messaging infrastructure if needed.

_Quick example of alarm configuration that could be used to monitor bounce rate. It will trigger an alarm if bounce rate breached 4% during the last 30 minutes:_

```json
{
  "...": "...",
  "MetricName": "Reputation.BounceRate",
  "Namespace": "AWS/SES",
  "Statistic": "Average",
  "Dimensions": [
    {
      "Name": "ses:configuration-set",
      "Value": "<your_configuration_set_name>"
    }
  ],
  "Period": 300,
  "EvaluationPeriods": 6,
  "DatapointsToAlarm": 1,
  "Threshold": 0.04,
  "ComparisonOperator": "GreaterThanThreshold",
  "TreatMissingData": "notBreaching",
  "...": "..."
}
```

Next step is to plug a SNS (Simple Notification Service) topic into both reputation metrics alarms, and to either :

- Subscribe your devs email address to the topic to be warned in case of issue.
- Subscribe a lambda automatically shutting down email sending in your configuration set.

![Alarms schema](./assets/alarms.png 'Alarms schema')

_Disabling a configuration set using Typescript and the AWS SDK v3:_

```ts
const client = new SESClient({});
await client.send(
  new UpdateConfigurationSetSendingEnabledCommand({
    Enabled: false,
    ConfigurationSetName: '<your_configuration_set_name>',
  }),
);
```

### Avoid sending emails to problematic recipients

With these alarms, you already have a way to quickly react to issues concerning reputation. But you can do more to avoid sending multiple emails to addresses harming your reputation.

One way to achieve this is to add bouncing or complaining addresses to a **suppression list**. Three types of suppression lists exist :

- Global level (AWS as a whole)
- Account level
- Configuration set level

Like for your alarm, I advise you to be as specific as possible and to **setup a a configuration set level suppression list**.

_Sample of configuration set config enabling suppression list of bounces and complaints. This configuration will automatically blacklist bouncing and complaining email addresses from your configuration set. You will be able to remove them manually from the list later._

```json
{
  "...": "...",
  "SuppressionOptions": {
    "SuppressedReasons": ["BOUNCE", "COMPLAINT"]
  },
  "...": "..."
}
```

Great! You are now sure that your mails will be sent 🙃. But will your users receive them in their inbox? Probably not all of them if you don't customize your configuration a bit.

## 2. Make sure you don't end up in your users spams

When sending a user receives an email, there are many factors at stake, that will determine if the email ends up in the mailbox or in the spams.

During my experience developing an email-sending service, I discovered [mail-tester][mail-tester]: a great tool to evaluate the probability of emails ending up in a inbox. Just send a test email to the address they give you, and it will analyze the likelihood of your message being well received.

![Mail tester bad](./assets/mail-tester-bad.png 'Mail tester bad')

This first mark is terrible, but you can fix it using the tips given by the website!

Many factors determine if your email will be considered suspicious, let me cover the most important ones.

### Email content

The easiest to fix (it may already be OK on your app), but also the most important. Mail clients will often automatically classify your email as spam if you don't comply.

- Your email should absolutely **have a subject and a body**.
- Your email **shouldn't be too short**, especially if it contains images.
- If your email contains links, prefer using **full links instead of minified ones**.
- Also, check that every link included in your email is healthy and **redirects to a working website**.
- Finally, a `List-Unsubscribe` header would improve your reputation, especially if you plan to send promotional emails. You will see in part 3 of this post how to implement it.

### Sender domain

The second thing checked by mail clients is the web domain where the email originates from. There are three major things checked by clients to ensure that the sender is safe:

- SPF and IP sending pool. By default, SES uses a common pool of IPs, that have accumulated over time a quite bad reputation. Good practice is to have your own domain to be able to configure SPF to reference a private pool of IP configured on SES.
- DKIM - email signing. Providing proof that the MAIL FROM header is really the sender.
- DMARC - specify how to behave if MAIL FROM and sender are different, in order to avoid other people acting on your domain

In this part of the article, I will guide you through using your own domain name to send emails to your users, and I will show you how to set up DKIM and DMARC on your emails.

If you don't already own a domain name, you can easily buy it for a fair price on **AWS Route 53**. Working with your own domain name will allow you to fine tune its DNS to become a trusted sender.

Let's then create a new identity in AWS SES: you want to create a domain identity, with the domain name corresponding to what you just reserved on AWS Route 53. Let's also assign your configuration set, that is linked to the cloudwatch alarms and suppression list previously set up.

![SES Domain Identity](./assets/ses-domain-identity.png 'SES Domain Identity')

Then, a key feature of domain based identities is the ability to specify a custom MAIL FROM domain, that will increase the mailing clients trust. Choose a subdomain of your domain like `notifications.<your_domain>`.

Using a subdomains adds clarity and trust for end readers, but it also allows you to separate sender reputations by subdomains, and thus avoid affecting your "whole" domain. (you do not impact your transactional emails if, for example, you screw up your promotional emails).

You can leave default behavior on MX failure and if your domain is hosted on Route 53, click the last checkbox to let AWS automatically do the job of updating the domain's DNS.

![SES Mail from domain](./assets/ses-mail-from-domain.png 'SES Mail from domain')

Finally, for maximal trust, let's set up **DKIM** on your SES identity. It is an authentication method guarantying to your recipients the ownership of your sender domain. Just choose easy DKIM, RSA_2048_BIT and check every checkboxes to set up a strong authentication.

![SES DKIM](./assets/ses-dkim.png 'SES DKIM')

Time to end with the icing on the cake: go to your domain's DNS (either on AWS Route 53 or on your own provider), and add a `_dmarc` `TXT` record to it, with value `v=DMARC1; p=none`. This **\_dmarc** record indicates to your recipients that you properly set up DKIM, and that they are allowed to refuse your email if there is an authentication failure.

![Mail tester good](./assets/route-53-dmarc.png 'Mail tester good')

Here you are, 10/10! You can now be quite certain that your users will see your emails. But it's not over yet: by default, AWS SES does little to nothing to help you send beautiful and responsive emails. If you want your emails to look like professional content, you will have to handle it yourself.

![Mail tester 10/10](./assets/mail-tester-10.png 'Mail tester 10/10')

## 3. Design responsive emails that look good in any mail client

### Templated vs raw emails

AWS SES supports two different modes to send emails from your Lambda functions: raw and templated emails. Each of these solutions has its benefits and drawbacks.

- Templated emails 🗒
  - ✅ Offer a quick win to design good looking emails.
  - ❌ Technically limited. For example, as of January 2023, you can't add custom headers or attachments to your messages.
  - ❌ Even using simple html + css in your templates may lead to some unexpected visual results.
- Raw emails ⚙️
  - ❌ Harder to handle (you have to take care of everything)
  - ✅ Allow great customization. Coupled with the right libraries they become the better solution in my opinion.

These two options allow you to send emails containing html content, which are nicer to read on the user's end. Templated emails are directly based on a html template, while a raws email can parse a html template's content to include it in the email (you'll see how to do this later).

Templates often give this feeling of controlling the UI of the mail you are going to send, but compatibility issues are so common that you should use external tools to generate all-clients compatible templates. A picture is worth thousand words, so let me show you a common "template" situation. I designed a simple html + css template to send templated credentials emails to my users. On some email clients like "macOS Mail", my template is perfectly displayed. But on "gmail", everything falls apart.

![MacOS Mail vs Gmail](./assets/macos-gmail-comparison.png 'MacOS Mail vs Gmail')

Why is it happening ? Every email client has different compatibilities with html and css. An awesome website to visualize that is [caniemail][caniemail]. It is the email counterpart of can-i-use and has information on every compatibility issues regarding emails. On caniemail, you can see that `flex-direction: column` isn't supported on gmail desktop, but is on macOS mail: everything becomes a little clearer!. Furthermore, looking at the global ranking, everything on macOS usually works well, while it isn't the case on windows and web-native counterparts. Let's figure out a solution!

![Email clients compatibility](./assets/compatibility.png 'Email clients compatibility')

### Design responsive email templates with MJML

Every mail client has different compatibilities relative to displaying html, css and images. That's why my advice is to use [MJML][mjml], a framework allowing you to create/design mail templates with a markup language close to html, that will be compiled to respect most compatibility issues on most clients. The output template can either be used to send templated or raw emails with SES.

Here it what the MJML code for my email template looks like:

```xml
<mjml>
  <mj-head>
    <mj-raw>
      <meta name="color-scheme" content="light" />
      <meta name="supported-color-schemes" content="light" />
    </mj-raw>
    <mj-style>
      a {
      color: #F48668;
      text-decoration: none;
      }
    </mj-style>
  </mj-head>
  <mj-body background-color="#FAFAFA">
    <mj-wrapper border-radius="8px" padding="15px">
      <mj-section background-color="#F48668" border-radius="8px 8px 0 0">
      </mj-section>
      <mj-section background-color="#FFFFFF" border-radius="0 08px 8px">
        <mj-column>
          <mj-text font-family="Trebuchet MS" color="#173940" font-size="22px" font-weight="600" align="center">
            Welcome to my app!
          </mj-text>
          <mj-spacer></mj-spacer>
          <mj-text font-family="Trebuchet MS" color="#173940" font-size="16px" align="center">
            Hello {{username}}, here is your temporary password:
          </mj-text>
          <mj-text font-family="Trebuchet MS" color="#173940" font-size="16px" font-weight="600" align="center">
            {{password}}
          </mj-text>
          <mj-text font-family="Trebuchet MS" color="#173940" font-size="16px" align="center">
            Click the button bellow to log in:
          </mj-text>
          <mj-button background-color="#F48668" border-radius="20px" font-size="16px" font-weight="600" href="http://www.pchol.fr">Join my app</mj-button>
        </mj-column>
      </mj-section>
    </mj-wrapper>
    <mj-wrapper border-radius="8px" padding="15px">
      <mj-section background-color="#FFFFFF" border-radius="8px">
        <mj-column>
          <mj-text font-family="Trebuchet MS" color="#173940" font-size="16px" align="center">
            Ran into a problem ? Do not hesitate to contact me at <a href="mailto:help@pchol.fr"><b>help@pchol.fr</b></a>
          </mj-text>
        </mj-column>
      </mj-section>
    </mj-wrapper>
  </mj-body>
</mjml>
```

- The `<mj-head>` part allows me to:
  - Specify my preferred color scheme (only light). [Advanced MJML tricks][advanced-mjml] include light and dark display compatibility. Check it out!
  - Write some css to further style my contact email address. I try to keep it minimal, as it won't be covered by MJML compatibility features and may not work on some clients.
- The `<mj-body>` part describes what will be displayed. It's not html but it remains fairly easy to understand. Check [the documentation][mjml-documentation] to learn more about this syntax!

By compiling your MJML template into html + css, you can use it in a SES template (or later in raw emails). To compile, either use the hands-on [live editor][mjml-live-editor], or go with npm:

```sh
npm run mjml myTemplate.mjml --output myTemplate.html
```

Let's have an look on the result! No more big differences between email clients. Except for the contact email address (I warned you 🤓), the result is just what was expected.

![MacOS Mail vs Gmail MJML](./assets/mjml-mac-gmail-difference.png 'MacOS Mail vs Gmail MJML')

### The power of raw emails: attachments, images, custom headers and more

Everything that was covered until now can be implemented using either templated or raw SES emails. But from now, you will be diving into the unique possibilities offered by raw emails, to push your messages to their limit.

Let's say I want to add a logo in my email's header. Using templated emails, it will be impossible to have it displayed on every mail client. The solution is to send a raw email with your image as attachment, and to reference it in the message's body, using contentID.

I modified my email's header MJML code to include an image:

```xml
<mj-section background-color="#F48668" border-radius="8px 8px 0 0" padding="0">
  <mj-column>
    <mj-image width="20px" src="cid:my-logo@pchol.fr" alt="My logo"/>
  </mj-column>
</mj-section>
```

Then, in the lambda I use to send my emails, I can use the Nodemailer library to translate my MJML template into a raw email, and to add attachments to my message.

```js
import * as aws from '@aws-sdk/client-ses';
import nodemailer from 'nodemailer';
import myHtmlTemplate from './myHtmlTemplate'; // imported as a string

export const handler = async () => {
  const sesClient = new aws.SESClient({});
  const transporter = nodemailer.createTransport({
    SES: { ses: sesClient, aws },
  });

  await transporter.sendEmail({
    from: 'notifications@pchol.fr', // my domain name
    to: '<recipient>',
    subject: 'This a raw email test',
    text: 'This a raw email test',
    html: myHtmlTemplate.replace('{{username}}', 'Pchol').replace('{{password}}', 'very_secret_password'), // manually replace parameters
    attachments: [
      {
        filename: 'pchol-logo.png',
        path: '/opt/pchol-logo.png', // image stored in a Lambda layer
        cid: 'my-logo@pchol.fr', // same as in the template
      },
    ],
  });

  return 'OK!';
};
```

The files that are attached to my mails come from a Lambda layer. These layers can be easily set on the different available frameworks.

Using additional parameters like `list` or `headers`, you can specify custom headers for your emails (which is impossible using templated emails). For instance, to add a `List-unsubscribe` header (the last obstacle to your mail-tester perfect score), you can add the following code to your handler:

```typescript
list: {
  unsubscribe: {
    url: 'http://www.pchol.fr',
    comment: 'Unsubscribe from this notifications',
  },
},
```

Here is the final result, containing the logo as an attachment and the `List-unsubscribe` header. The full mail, with html styles, attachment and custom headers obtains a 100% score on mail-tester when I send it from my custom domain based SES identity.

![Final email with attachment](./assets/email-final.png 'Final email with attachment')

Mail-tester is happy too !

![Mail tester list-unsubscribe](./assets/list-unsubscribe.png 'Mail tester list-unsubscribe')

SES raw emails allow you to send MIME emails, that can have multiple content-type sections (text/html, text/plain...). If you open the source of the email you received, you can see these different sections and their content. You can also the the List-unsubscribe header you just added!

```txt
...

List-Unsubscribe: <http://www.pchol.fr> (Unsubscribe)
Date: Wed, 4 Jan 2023 17:46:43 +0000
MIME-Version: 1.0

...

----_NmP-92cb611d778ad2fe-Part_1
Content-Type: text/plain; charset=utf-8
Content-Transfer-Encoding: 7bit

This a raw email test

...

----_NmP-92cb611d778ad2fe-Part_3
Content-Type: text/html; charset=utf-8
Content-Transfer-Encoding: quoted-printable

... my html ...
```

This is important, because if a client simply refuses to display your html content, it will fallback to the text/plain one.

## Conclusion

Here is a little check-list of everything I covered in this article, and that you should check when developing emails sending on AWS SES:

- ✅ Alarms monitoring my AWS sender reputation
- ✅ Configuration set level suppression lists
- ✅ Custom domain name SES identity
- ✅ DKIM on my domain
- ✅ DMARC on my domain
- ✅ Templated VS Raw emails argument
- ✅ Responsive email templates using MJML
- ✅ Raw emails allowing to send MIME emails (attachments, custom headers...)

For each of these problems, I tried to show you simple yet effective counter-measures, that are easy and quick to implement.

[mail-tester]: https://www.mail-tester.com/
[mjml]: https://mjml.io/
[advanced-mjml]: https://www.emailonacid.com/blog/article/email-development/advanced-mjml-coding/
[mjml-documentation]: https://documentation.mjml.io/#standard-body-components
[mjml-live-editor]: https://mjml.io/try-it-live
[caniemail]: https://www.caniemail.com/
[sls-mentor]: https://sls-mentor.dev
