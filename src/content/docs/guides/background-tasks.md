---
title: "Background tasks and scheduled jobs"
description: "Run Celery tasks with a queue and a worker. Scheduled jobs are not available yet."
---

Your deployment runs a web process: uvicorn (or gunicorn) serving your Django app. Anything that runs *inside* that process, such as a thread or a scheduler started at import time, stops when the container restarts and runs once per instance.

## Background tasks: Celery

DjangoCloud runs Celery for you. Add a **task queue** and turn on a **worker**, and your tasks run in a second container next to your app. See [Task queues and workers](../task-queues-and-workers/).

## Scheduled jobs

:::caution
Scheduled jobs (Celery beat and cron) are **not available yet**. A scheduler started inside each instance would run every job once per instance.
:::

What works instead:

* **Work triggered from outside.** For example a scheduled GitHub Action, or an external cron service calling a protected URL in your app that does the work.
* **Short work inside a request.** Anything that finishes while the user waits.

## Tips

* Make tasks and jobs **idempotent**. With more than one instance, or after a restart, the same one can run twice.
* Don't rely on local disk to pass data between a request and a task. See [the ephemeral container](../../how-it-works/ephemeral-container/), and use [file storage](../file-storage/) for files.
