---
title: "Known limitations"
description: "What to know before you build on DjangoCloud. We list these plainly so you aren't surprised later."
---

What to know before you build on DjangoCloud. We list these plainly so you aren't surprised later.

## The container is ephemeral

Files written to disk are lost on every restart and deploy. Use a managed database and object storage. See [Persistent files and the ephemeral container](../../how-it-works/ephemeral-container/).

## How your app is served

Your app is started with **uvicorn** when it has an ASGI app (`asgi_module`), and with gunicorn when it only has a WSGI app. Both use the `workers` from your [build settings](../../how-it-works/build-settings/). Websockets work with uvicorn. Each deployment is a single small container, so the number of concurrent connections is limited by its size, and a restart or deploy closes open connections. You can replace the command with `start_command`, but that is your own responsibility to test.

## Always on, and no background workers yet

* Your app runs around the clock. There is no scale-to-zero, which also means no cold starts.
* Each deployment runs **one web process**. Background workers (Celery) and scheduled jobs (cron) are **coming soon** and are not available yet. See [Background tasks and scheduled jobs](../../guides/background-tasks/) for what works today.

## Build and source limits

* The source upload is limited to **100 MB**.
* A build that runs for around 9 to 10 minutes is stopped. Keep images lean, and avoid compiling large dependencies from source during the build.
* Python 3.10 to 3.13 are supported, with pip or uv.
* Only a folder with `manage.py` is deployed. If your project lives in a subfolder, set `root` in the build settings.

## The first build in a brand-new AWS account can take hours

A new AWS account starts with no build capacity. DjangoCloud asks AWS to enable it automatically, and the deployment page says so while it waits. This happens once per account.

## Rollbacks restore code, not data

A rollback does not reverse database migrations. DjangoCloud blocks a rollback that would run old code against a newer schema unless you confirm it. See [Releases and rollbacks](../../how-it-works/releases-and-rollbacks/).

## What we keep, and for how long

* **Releases you can roll back to** (their images and uploaded source): the newest 3.
* **Logs:** 30 days.
* **CPU and memory graphs:** 30 days.

 Older images and uploads are deleted automatically after a new release goes live, and a release whose image was deleted can't be rolled back to. The live release is never deleted.

When you **delete a deployment**, its server and images are deleted from AWS too. A database is deleted after we save a final snapshot of it in your AWS account, so you can still restore it.

## Environment variables

* Up to 200 per deployment.
* Changes apply on the next deploy, not to the running release.

## Regions

You can deploy to the [regions listed here](../regions/), where Lightsail container services exist. A deployment stays in the region it was created in.

## Metrics and logs

Logs and metrics are collected every few minutes, so the newest lines can be a little behind.

## Self-hosted: you run your own cloud

On Self-hosted, your app runs in your AWS account, so AWS limits and quotas on your account apply. On Fully managed we run it in a dedicated AWS account for you, so there is no AWS account to look after.
