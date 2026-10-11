---
title: "File storage"
description: "A private bucket for the files your users upload, which survive every deploy."
---

Your app's disk is reset on every deploy, so uploaded files must live somewhere else. File storage is a **private bucket** (S3-compatible, on AWS Lightsail) in your AWS environment, with a key that opens only that bucket. You paste one snippet into `settings.py`; DjangoCloud never edits your code.

## Sizes

Choose one when you add the storage:

| Size | Data transfer out per month |
| --- | --- |
| 5 GB | 25 GB |
| 100 GB | 250 GB |
| 250 GB | 500 GB |

Storage or transfer beyond the size you chose is billed by AWS at its overage rates. A bucket cannot be resized yet, so pick the size you need.

## Set it up

### 1. Add storage

**Storage**, then **Add storage**, then pick the deployment and a size.

### 2. Paste this into `settings.py`

On your computer it changes nothing. On DjangoCloud the variables exist, so uploads go to the bucket. Your other `STORAGES` entries (for example WhiteNoise) are kept:

```python
import os

# On your computer nothing changes: files stay where your settings put them (MEDIA_ROOT).
# On DjangoCloud they go to the private bucket we created for you. Your other STORAGES entries are kept.
if "DJANGOCLOUD_STORAGE_BUCKET" in os.environ:
    STORAGES = {
        "staticfiles": {"BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage"},
        **globals().get("STORAGES", {}),  # whatever you already had (for example WhiteNoise) wins
        "default": {
            "BACKEND": "storages.backends.s3.S3Storage",
            "OPTIONS": {
                "bucket_name": os.environ["DJANGOCLOUD_STORAGE_BUCKET"],
                "access_key": os.environ["DJANGOCLOUD_STORAGE_ACCESS_KEY_ID"],
                "secret_key": os.environ["DJANGOCLOUD_STORAGE_SECRET_ACCESS_KEY"],
                "region_name": os.environ["DJANGOCLOUD_STORAGE_REGION"],
                "querystring_auth": True,  # the bucket is private: links to files are signed and expire
                "file_overwrite": False,
            },
        },
    }
```

This needs Django 4.2 or newer and `django-storages` 1.14 or newer.

### 3. Deploy

:::note
**You do not install `django-storages`.** When DjangoCloud builds your image it adds `django-storages[s3]` after your own requirements. If you use your own Dockerfile, add it there.
:::

## The variables DjangoCloud sets

They are saved encrypted and given to your app every time it deploys: `DJANGOCLOUD_STORAGE_BUCKET`, `DJANGOCLOUD_STORAGE_ACCESS_KEY_ID`, `DJANGOCLOUD_STORAGE_SECRET_ACCESS_KEY` and `DJANGOCLOUD_STORAGE_REGION`.

Like every variable, they are saved into each release, so storage you add reaches your app on its **next deploy**. See [Environment variables](../../how-it-works/environment-variables/).

## The bucket is private

Nothing is public. Links to files are **signed and expire**, so use `file.url` in your templates and do not store the links. The key can read and write this one bucket and nothing else.

## Seeing what is happening

**Storage**, then your deployment:

* **Overview:** how full the bucket is against its size, how many files it holds, and how long until it is full at the current growth. At 80% a banner asks you to plan ahead. At 100% it warns that further uploads are billed at overage rates.
* **Metrics:** charts of the storage used and the number of files over 7 to 30 days.

AWS reports the size of a bucket **once a day**, so the numbers lag behind. Data transfer used is not shown, because Lightsail does not report it per bucket.

## Removing it

**Storage**, then your deployment, then **Settings**, then **Delete storage and files**. **Every file is deleted for good**, with no backup, and you must type the bucket's name to confirm. Deleting the deployment deletes its bucket too.

## Troubleshooting

* **Uploads still go to the server's disk.** The snippet is missing from `settings.py`, or you have not deployed since adding storage.
* **`ModuleNotFoundError: storages` with your own Dockerfile.** Install `django-storages[s3]` in it.
* **Images stop showing after a while.** Signed links expire. Generate them when the page renders with `file.url` instead of saving them.
