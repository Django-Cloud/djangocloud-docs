---
title: "Connect your AWS account"
description: "DjangoCloud runs your app in your AWS account, on Amazon Lightsail, through a limited IAM role."
---

DjangoCloud runs your app in **your** AWS account, on Amazon Lightsail. To do that it needs access to your account. The recommended way is an **IAM role** that trusts DjangoCloud: nothing secret is stored, and you revoke access by deleting the role. An IAM access key is available as a fallback.

<div style="aspect-ratio:16/9;max-width:100%;margin:1.5rem 0"><iframe src="https://www.youtube.com/embed/OB9ktTUv66s" title="Connect your AWS account to Django Cloud" loading="lazy" style="width:100%;height:100%;border:0;border-radius:8px" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></div>

## Option 1: an IAM role (recommended)

Wondering whether this is safe? Read [Is my AWS account safe?](../../reference/is-my-aws-account-safe/), or the [security model](../../reference/aws-access-security/) for the technical details.

DjangoCloud assumes the role with `sts:AssumeRole` for short sessions (one hour at a time). The role's trust policy only lets DjangoCloud in when the request carries **your own ExternalId**, which is shown in **Settings → AWS account** and by `GET /api/v1/aws`. The ExternalId is not a secret, but it is unique to you; it stops anyone else from making DjangoCloud act in your account.

### With CloudFormation (easiest)

No AWS knowledge needed. In **Settings → AWS account**:

1. Click **Set up the role in AWS** and pick your region.
2. Click **Open AWS**. The AWS console opens with everything filled in. Sign in to the AWS account you want to deploy into if asked.
3. At the bottom of the AWS page tick **I acknowledge that AWS CloudFormation might create IAM resources with custom names** and click **Submit**.
4. That's it. When AWS finishes creating the role (about a minute), AWS notifies DjangoCloud, which checks the role in the background and connects automatically. You don't copy anything back, and you don't need to keep the page open: it updates when it's done, and **Settings → AWS account** shows the result if you come back later.

The stack creates a role named `djangocloud-deployer` with the trust policy and the permissions below. When it finishes it tells DjangoCloud the role's ARN together with your ExternalId, and DjangoCloud connects only if it can actually assume the role with that ExternalId. To remove DjangoCloud's access later, delete the stack.

If the page doesn't connect, open the stack's **Outputs** tab in AWS, copy `RoleArn` and paste it into **Role ARN** on the same page. The dialog also offers **download the template** if you prefer to upload it to CloudFormation yourself.

### By hand

1. In the AWS console open **IAM → Roles → Create role → Custom trust policy**.
2. Copy the trust policy from **Settings → AWS account**. It is already filled in with your own ExternalId, so paste it as is. It allows exactly one identity, `arn:aws:iam::710023142180:user/server`, to assume the role, and only when the request carries your ExternalId (`sts:ExternalId`). Without your ExternalId nobody, including another DjangoCloud customer, can use your role.

3. Attach the permissions policy below and name the role exactly `djangocloud-deployer`. DjangoCloud can only assume a role with that name. Copy its ARN.
4. Paste the ARN in **Settings → AWS account** and click **Verify and connect**.

DjangoCloud assumes the role and calls AWS before saving, and tells you on the same page what to fix if the trust policy or permissions are wrong. The check runs in the background and usually takes a few seconds.

### Permissions policy

This policy lets DjangoCloud run your app on Lightsail and set up a small build environment in your account (a build project, a private bucket for build inputs, a role and logs). It has no access to any other service:

```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "lightsail:*",
           "sts:GetCallerIdentity"
         ],
         "Resource": "*"
       },
       {
         "Effect": "Allow",
         "Action": [
           "servicequotas:GetServiceQuota",
           "servicequotas:GetAWSDefaultServiceQuota",
           "servicequotas:ListServiceQuotas",
           "servicequotas:RequestServiceQuotaIncrease",
           "servicequotas:GetRequestedServiceQuotaChange",
           "servicequotas:ListRequestedServiceQuotaChangeHistory",
           "servicequotas:ListRequestedServiceQuotaChangeHistoryByQuota"
         ],
         "Resource": "*"
       },
       {
         "Effect": "Allow",
         "Action": [
           "kms:CreateGrant",
           "kms:DescribeKey",
           "kms:ListAliases",
           "kms:ListGrants",
           "kms:Decrypt",
           "kms:GenerateDataKey",
           "kms:GenerateDataKeyWithoutPlaintext",
           "kms:ReEncrypt*"
         ],
         "Resource": "*"
       },
       {
         "Effect": "Allow",
         "Action": "iam:CreateServiceLinkedRole",
         "Resource": "arn:aws:iam::*:role/aws-service-role/servicequotas.amazonaws.com/*",
         "Condition": {
           "StringEquals": {
             "iam:AWSServiceName": "servicequotas.amazonaws.com"
           }
         }
       },
       {
         "Effect": "Allow",
         "Action": "codebuild:*",
         "Resource": "arn:aws:codebuild:*:*:project/djangocloud-build*"
       },
       {
         "Effect": "Allow",
         "Action": [
           "logs:CreateLogGroup",
           "logs:PutRetentionPolicy",
           "logs:GetLogEvents",
           "logs:DescribeLogStreams"
         ],
         "Resource": [
           "arn:aws:logs:*:*:log-group:/aws/codebuild/djangocloud-build",
           "arn:aws:logs:*:*:log-group:/aws/codebuild/djangocloud-build:*"
         ]
       },
       {
         "Effect": "Allow",
         "Action": "s3:*",
         "Resource": [
           "arn:aws:s3:::djangocloud-build-*-*",
           "arn:aws:s3:::djangocloud-build-*-*/*"
         ]
       },
       {
         "Effect": "Allow",
         "Action": [
           "iam:CreateRole",
           "iam:GetRole",
           "iam:PutRolePolicy",
           "iam:TagRole"
         ],
         "Resource": "arn:aws:iam::*:role/djangocloud-build-*"
       },
       {
         "Effect": "Allow",
         "Action": "iam:PassRole",
         "Resource": "arn:aws:iam::*:role/djangocloud-build-*",
         "Condition": {
           "StringEquals": {
             "iam:PassedToService": "codebuild.amazonaws.com"
           }
         }
       }
     ]
   }
   ```

## Option 2: an IAM access key (fallback)

:::caution
Create a dedicated IAM user for this. Never use your root account keys.
:::

1. In the AWS console open **IAM → Users → Create user**, for example `djangocloud`.
2. Attach the permissions policy above to the user.
3. Under **Security credentials**, create an access key.
4. In **Settings → AWS account** open **Or use an access key instead**, choose a [region](../../reference/regions/), and paste the access key ID and secret access key.

DjangoCloud checks the keys with AWS before saving them. You can switch an existing key connection to a role at any time from the same page; the stored key is then deleted.

## New AWS accounts

:::note
If your AWS account is brand new, AWS may need some hours to enable builds for it the first time. DjangoCloud asks for it automatically and shows "Waiting for AWS to enable builds" on your deployment page, then continues on its own.
:::

## How access is handled

* **Role:** no secret is stored. DjangoCloud keeps the role ARN and your ExternalId, and assumes the role for short sessions. Delete the role (or the CloudFormation stack) in AWS to revoke access immediately.
* **Access key:** the secret is encrypted at rest, only used to deploy your projects, and never shown again after you save it. Revoke the key in IAM at any time.
* You can also **disconnect** your account in the dashboard. Disconnecting does not stop anything: existing deployments keep running on AWS.

## Choosing a region

Pick the region closest to your users. All deployments created while a region is connected go there. See the [list of supported regions](../../reference/regions/).
