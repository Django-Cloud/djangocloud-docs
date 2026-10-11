---
title: "Task queues and workers"
description: "Run background tasks with Celery: a task queue (Amazon SQS) and a worker that runs next to your app."
---

Run background tasks with Celery. You add a **task queue** and turn on a **worker**. DjangoCloud creates the queue in your AWS environment, adds its login to your deployment, and runs the worker next to your app. You paste one snippet into `settings.py`; DjangoCloud never edits your code.

## What you get

* **A queue** named `dc-<your-slug>-celery`, encrypted at rest. This is where `.delay()` puts tasks and where the worker takes them from.
* **A dead-letter queue** named `dc-<your-slug>-dead`. A task that fails 5 times in a row is moved there and kept for 14 days, so a failing task is never lost silently.
* **A key that can use only those queues.** It belongs to an IAM user named `djangocloud-queue-<your-slug>`, which can send, receive and delete messages on this app's queues and nothing else. It cannot create or delete queues.
* **A worker** (optional): your app's image, started a second time as `celery -A <your app> worker`. See [the worker](#the-worker).

## Set it up

The dashboard shows each step with your own folder names filled in (**Workers**, then your deployment, then **Setup**).

### 1. Add the queue

**Queues**, then **Add queue**, then pick the deployment. It takes a few seconds. On Self-hosted you may be asked to update your DjangoCloud CloudFormation stack the first time: its role needs permission to create queues. The page says so, and you retry once the stack is updated.

### 2. Create `celery.py`

Skip this if you already have a Celery app. Create a file called `celery.py` in the same folder as your `settings.py`:

```python
import os

from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
app = Celery("config")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

and add these lines to the `__init__.py` in that same folder, so Celery loads when Django starts:

```python
from .celery import app as celery_app

__all__ = ("celery_app",)
```

Replace `config` with the name of the folder that holds your `settings.py`. The dashboard fills in the right name once you have deployed.

### 3. Paste this into `settings.py`

On your computer it changes nothing, because the variables only exist on DjangoCloud:

```python
import os

if "DJANGOCLOUD_QUEUE_BROKER_URL" in os.environ:
    CELERY_BROKER_URL = os.environ["DJANGOCLOUD_QUEUE_BROKER_URL"]
    CELERY_BROKER_TRANSPORT_OPTIONS = {
        "region": os.environ["DJANGOCLOUD_QUEUE_REGION"],
        "queue_name_prefix": os.environ["DJANGOCLOUD_QUEUE_PREFIX"],
        "visibility_timeout": 300,
        "wait_time_seconds": 20,  # long polling: an idle worker asks SQS once every 20 seconds (AWS's maximum)
    }
    # Every request to SQS costs a little, so the worker only does the work:
    CELERY_WORKER_ENABLE_REMOTE_CONTROL = False  # no control queue to poll (so no `celery inspect`)
    CELERY_WORKER_SEND_TASK_EVENTS = False  # no event stream for monitors like Flower
    CELERY_WORKER_PREFETCH_MULTIPLIER = 1  # take one task at a time, so a busy worker doesn't hoard
    CELERY_BROKER_CONNECTION_RETRY_ON_STARTUP = True
    # SQS keeps no task results. If you use a result backend, keep it pointed at something else (your database).
```

This assumes Celery reads its settings with the `CELERY_` prefix, as in the `celery.py` above (the standard Django setup from the Celery documentation).

### 4. Turn the worker on, and deploy

**Workers**, then your deployment, and turn it on. Then deploy.

You do not have to say where your Celery app lives: when DjangoCloud builds your image it looks through your code for `app = Celery(...)` (normally the `celery.py` from step 2) and starts the worker with it. If you have more than one, or want to be explicit, enter the place yourself, for example `config.celery:app`. That always wins over what we find.

:::note
**You do not install Celery's SQS support.** When DjangoCloud builds your image it adds `celery[sqs]` (and `django-storages[s3]` if you use [file storage](../file-storage/)) after your own requirements, so anything you pinned is kept. If you use [your own Dockerfile](../../how-it-works/build-settings/), it is not touched: add `celery[sqs]` there.
:::

## The variables DjangoCloud sets

They are saved encrypted on your deployment and given to your app **and to its worker** every time it deploys:

* `DJANGOCLOUD_QUEUE_BROKER_URL`, `DJANGOCLOUD_QUEUE_ACCESS_KEY_ID`, `DJANGOCLOUD_QUEUE_SECRET_ACCESS_KEY`
* `DJANGOCLOUD_QUEUE_REGION`, `DJANGOCLOUD_QUEUE_PREFIX`
* `CELERY_BROKER_URL`, unless you already set your own. We never replace yours.

Like every variable, they are saved into each release. **Adding a queue does not change a release that is already live**: the variables reach your app on its next deploy, and a rollback restores the variables that release had. See [Environment variables](../../how-it-works/environment-variables/).

### Variables for the worker or the web app only

Both containers get the same variables, which is usually what you want. To give one of them a variable the other should not have, start its name with `WORKER_ONLY__` or `WEB_ONLY__`. The container that matches gets it **without the prefix**, and the other one does not get it at all:

```
WORKER_ONLY__STRIPE_SECRET_KEY=rk_live_...   # the worker sees STRIPE_SECRET_KEY, the web app has none
WEB_ONLY__SESSION_COOKIE_DOMAIN=example.com  # the web app sees SESSION_COOKIE_DOMAIN
```

A prefixed variable beats a plain one of the same name in its own container. Use it for secrets that only background tasks need.

## The worker

* It is a second container in the same deployment, built from the **same image** as your app but started with `celery -A <your app> worker -l info --without-gossip --without-mingle --without-heartbeat` instead of your web server.
* It gets **the same environment variables** as your app, including your database.
* It **shares the server's CPU and memory** with your app and **runs once on every instance**. With 3 instances you have 3 workers. A busy worker slows your web app down: if tasks wait too long, give the server more CPU or run more instances. See [Size and scaling](../../how-it-works/size-and-scaling/).
* Every deploy replaces the worker together with the app, with the new image. A task that was running when the old worker stopped is picked up again after the queue's 5-minute visibility timeout, so tasks must be safe to run twice.
* Its output appears in your deployment's **Logs** with the source **Worker**. See [Logs and metrics](../../how-it-works/logs-and-metrics/).
* Turning it off removes the container on the next deploy.

## Keeping the requests low

An idle worker should cost almost nothing, so the setup does only the work:

* **Long polling.** An idle worker asks SQS once every 20 seconds (the maximum AWS allows), and the queues long-poll by default. Celery's own default asks every 10 seconds.
* **No heartbeats, gossip or mingle.** The worker starts with `--without-heartbeat --without-gossip --without-mingle`.
* **No control queue.** With `CELERY_WORKER_ENABLE_REMOTE_CONTROL = False` there is no extra queue to poll. The trade-off: `celery inspect` and remote control commands do not work.
* **No task events.** With `CELERY_WORKER_SEND_TASK_EVENTS = False`, tools that need the event stream, like Flower, do not work.
* **Logging at the task level.** The worker logs at `info`: a line when a task is received and when it succeeds or fails. There is no per-poll or per-heartbeat logging.

## Seeing what is happening

**Queues**, then your deployment:

* **Overview:** what is waiting and being worked on, how long the oldest task has waited, how many tasks finished and were queued in the last hour, and how many **failed for good**. A banner warns when tasks have waited over 5 minutes (your worker is not keeping up) or when tasks are in the dead-letter queue. A checklist shows what is still missing: queue created, login saved, the snippet found in your code, the live release has the queue, worker on, worker connected.
* **Metrics:** charts of tasks waiting, in flight, the age of the oldest task, tasks queued and finished per 5 minutes, empty polls, and the dead-letter queue, for 1 hour to 7 days.
* **Logs:** what your worker says about each task: when it was received, whether it succeeded or failed, and how long it took. Nothing shows what is inside a task.

The numbers come from AWS and are refreshed about every 5 minutes.

## Limits worth knowing

* **Delivery is at least once.** A task can run twice, for example if a worker is stopped part-way, so make tasks safe to repeat.
* **Long tasks.** A task being worked on is hidden from other workers for 5 minutes. A task that runs longer would be picked up a second time. Raise `visibility_timeout` in the snippet if you need longer tasks.
* **Delays.** SQS can delay a message by at most 15 minutes. For a longer `countdown` or `eta`, Celery holds the task in the worker's memory, which is lost if the worker restarts. Avoid long `eta` tasks.
* **No result backend.** SQS does not store task results. If you need them, keep your own `CELERY_RESULT_BACKEND`, for example `django-db`. DjangoCloud never changes it.
* **Scheduled jobs (Celery beat) are not available.** A second copy would run on every instance. See [Background tasks and scheduled jobs](../background-tasks/).
* **Order is not guaranteed.**

## Removing it

**Queues**, then your deployment, then **Settings**, then **Remove queue**. The queue, the dead-letter queue and the IAM user are deleted. **Tasks still waiting are lost.** The `DJANGOCLOUD_QUEUE_*` variables are removed, and `CELERY_BROKER_URL` too if it is still the one DjangoCloud set. Deleting the deployment removes them as well.

## Troubleshooting

* **Nothing runs.** Open the queue's **Overview** and read the checklist. The usual causes are the snippet missing from `settings.py`, or no deploy since you added the queue.
* **"DjangoCloud isn't allowed to create a task queue."** On Self-hosted, update your DjangoCloud CloudFormation stack (**Settings**, then **AWS account**) and retry.
* **"We couldn't find your Celery app in your code."** Add the `celery.py` from step 2 (it must contain `app = Celery(...)`), or enter the place of your app on the Workers page, and deploy again. Files in folders called `tests`, `migrations` or `docs` are not searched.
* **Tasks wait a long time.** The worker is slow or busy. Check the Logs, then give the server more CPU or run more instances.
* **A task failed for good.** Search the Logs for the task name. Tasks in the dead-letter queue stay for 14 days.
