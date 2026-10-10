---
title: "Deleting a deployment and disconnecting AWS"
description: "You can delete a deployment from the dashboard."
---

## Delete a deployment

You can delete a deployment from the dashboard. This removes it from DjangoCloud, together with its release history and settings, and then deletes what it left in AWS:

* its **container service** and **images**, so nothing keeps running,
* its **database**, if it has one, after we save a **final snapshot** of it in your AWS account (named `final-<database name>-<date>`),
* the **source uploads** we stored for it.

:::caution
Deleting is final. The database snapshot is the only thing kept, and it stays in your AWS account until you delete it there.
:::

Removal runs in the background. If your AWS account is disconnected or its environment was already removed, we delete only what we hold ourselves.

## Disconnect your AWS account

Under **Settings → AWS account** in the dashboard you can disconnect your account. DjangoCloud forgets your role (or access key). **Existing deployments keep running on AWS**, but you can no longer deploy new releases until you connect again.

To fully revoke access, also delete the IAM role (or deactivate the access key) in the AWS console.

## Revoke CLI tokens

Tokens for the CLI and CI are listed under **Settings → Developer**. Revoke any you no longer use.

## Delete your DjangoCloud account

Open **Settings → Account** and use **Close account**. It cancels your subscription and deletes everything: your deployments, their servers, images and databases (a final database snapshot is taken first), your AWS environment if we host it, and your account. It cannot be undone, and you confirm by typing your email address.
