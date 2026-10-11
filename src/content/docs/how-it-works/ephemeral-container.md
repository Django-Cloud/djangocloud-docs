---
title: "Persistent files and the ephemeral container"
description: "Your app runs in a container, and the container's filesystem is ephemeral."
---

Your app runs in a container, and **the container's filesystem is ephemeral**. Anything your app writes to disk is lost when the container restarts, when you deploy a new release, when you scale, and when AWS moves your app to another node.

:::danger
Never keep data you care about on the container's disk. That includes uploaded files, a SQLite database, generated reports and anything cached on disk that you can't recreate.
:::

## What this means for a Django app

| What | Do this instead |
| --- | --- |
| **Database** | Use a managed Postgres database. SQLite files are lost on every deploy. |
| **User uploads** (`MEDIA_ROOT`) | Use [file storage](../../guides/file-storage/): a private bucket, connected with `django-storages`. |
| **Static files** | `collectstatic` runs during the build, so they are baked into the image. Serve them with WhiteNoise or from object storage. |
| **Sessions and cache** | Use the database, or an external cache. Don't rely on files or in-process memory, especially with more than one instance. |
| **Background work** | Use a [Celery worker](../../guides/task-queues-and-workers/). A thread or process inside the web container stops whenever the container does. |
| **Scheduled work** | Not available yet. Trigger it from outside, see [background tasks](../../guides/background-tasks/). |

## More than one instance

If you scale to several instances, each has its own filesystem and its own memory. Anything that must be shared, such as sessions, caches and files, has to live outside the container.

## Temporary files are fine

Use the disk freely for temporary work within a single request, for example resizing an image before sending it to S3. Just don't expect it to be there later.
