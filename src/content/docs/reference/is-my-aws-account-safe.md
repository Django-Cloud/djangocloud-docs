---
title: "Is my AWS account safe?"
description: "What DjangoCloud can and can't do in your AWS account, in plain language: no password or key to hand over, access that expires within the hour, and a switch you control."
---

Short answer: yes, and you stay in control. This page explains why without assuming you know AWS. If you want the technical details, read [AWS access: the security model](../aws-access-security/).

## The idea in one picture

Handing over an **access key** is like making a copy of your house key and mailing it to someone. It works until you remember to change the locks.

Connecting with an **IAM role**, which is what DjangoCloud recommends, is like a hotel key card. You tell AWS: "Let DjangoCloud in, but only to these rooms, only for an hour at a time, and only when they show the code that belongs to me." The card stops working by itself, you can cancel it in one click, and the hotel (AWS) keeps a log of every door it opened.

## What you set up

1. In **Settings → AWS account** you click **Set up the role in AWS**.
2. AWS opens with everything filled in. You tick one box and click **Submit**. That creates a role called `djangocloud-deployer` in *your* account.
3. DjangoCloud connects by itself. You don't copy or paste any password or key.

## What DjangoCloud stores

* The **role address** (called an ARN), the **AWS account number** and the **region** you picked.
* Your personal **ExternalId**, a long random code that only works with your role. It is not a password.

DjangoCloud does **not** store an AWS password or access key for you. There is nothing secret of yours in our database to steal.

## What DjangoCloud can and can't do

**It can** run your Django app on Amazon Lightsail, build your code in a small build setup (AWS CodeBuild) and create the resources that needs, such as the container service, a database if you ask for one, and logs.

**It can't** touch the rest of your AWS account. The permissions are limited to Lightsail, a few quota settings and DjangoCloud's own build resources, and every one of them is listed on [Connect your AWS account](../../getting-started/connect-aws/). It has no access to your other databases, storage buckets, servers or billing.

## How long does access last?

DjangoCloud asks AWS for temporary credentials that expire after **one hour**. After that it must ask again, and AWS checks the rules every time. If you delete the role, the next request is refused.

## How can I see what happened?

Every time DjangoCloud uses the role, AWS writes it to **CloudTrail**, a log that lives in your account and that we cannot edit. Entries show the name `djangocloud-` and a number so you can recognise them. See [how to look](../aws-access-security/#seeing-what-happened) if you want to check.

## How do I cut access off?

Delete the stack named `djangocloud-<region>` (for example `djangocloud-eu-west-1`) in **CloudFormation** (or delete the role `djangocloud-deployer` in **IAM**). Access ends immediately. Your running app keeps running, but DjangoCloud can no longer change it. You can also click **Disconnect** in **Settings → AWS account**; see [Deleting and disconnecting](../../how-it-works/deleting-and-disconnecting/).

## Things worth doing for extra safety

* **Use a separate AWS account** for what DjangoCloud runs. It is free to create one inside the same AWS Organization, and it keeps everything else away from the role.
* **Set an AWS billing alert** so you notice unexpected spending early.
* Keep **two-step sign-in** on your DjangoCloud and AWS accounts.

## Where the limits are

No tool is risk free, and we would rather tell you. While the role exists, DjangoCloud can change what runs in your Lightsail and build setup. That is the job you gave it. A separate AWS account keeps the damage from any mistake, ours or yours, inside that account. The technical page describes exactly what is and isn't covered: [AWS access: the security model](../aws-access-security/).

## Prefer an access key?

It still works as a fallback. The secret is stored encrypted, shown only once and never again, and you can revoke it in IAM at any time. A role is safer because nothing secret has to be kept anywhere.
