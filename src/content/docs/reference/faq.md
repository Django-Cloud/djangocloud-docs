---
title: "FAQ"
description: "On Self-hosted, yes: your app runs in your account and AWS bills you directly. On Fully managed, no: we run it for you."
---

## Do I need an AWS account?

On **Self-hosted**, yes. Your app runs in your AWS account and AWS bills you directly. On **Fully managed**, no: we run it for you in a dedicated AWS account that we set up and look after.

## Can I use SQLite?

Not in production. The container's disk is wiped on every deploy and restart. Use Postgres. See [Databases](../../guides/databases/).

## Where do my uploaded files go?

Not on local disk. Use object storage such as Amazon S3. See [the ephemeral container](../../how-it-works/ephemeral-container/).

## Can I deploy on every push?

Yes. [Connect GitHub](../../getting-started/github/) and pick a branch.

## Can I deploy from CI?

Yes, with an API token. See [Deploy from CI](../../getting-started/ci/).

## How do I undo a bad deploy?

Roll back to an earlier release. Code rolls back, but database migrations don't. See [Releases and rollbacks](../../how-it-works/releases-and-rollbacks/).

## Does it support websockets or ASGI?

Yes. If your project has an ASGI app (every `startproject` does), it is started with uvicorn, and websockets work with Channels or your own consumers. Open connections are closed on every deploy and restart. See [Build settings](../../how-it-works/build-settings/).

## Can I run Celery workers or cron jobs?

Not yet: they are coming soon. Today each deployment runs one web process. See [Background tasks and scheduled jobs](../../guides/background-tasks/) for what works in the meantime.

## Which Python and Django versions?

Python 3.10 to 3.13. The CLI needs Django 4.2 or newer.

## Where is my data?

Your app and its data live in the AWS region you choose, in your account. See [Regions](../regions/).

## Something isn't working

See [Troubleshooting failed deploys](../../troubleshooting/failed-deploys/), or ask in [Slack](../../community-and-support/support/).
