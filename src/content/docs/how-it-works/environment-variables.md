---
title: "Environment variables"
description: "Set secrets and configuration in the dashboard, under your deployment's Settings. Your .env file is never uploaded."
---

Set secrets and configuration in the dashboard, under your deployment's **Settings**. Your `.env` file is never uploaded.

## Adding variables

Paste a block of `KEY=value` lines. DjangoCloud checks the whole block first, and if any line is invalid it saves nothing and tells you which, so a typo can't half-apply. You can choose to replace all existing variables with the pasted block instead of adding to them.

* You can store up to 200 variables per deployment.
* Values are encrypted at rest and can't be read back from the dashboard once saved.

## From the command line

```bash
djangocloud env push production.env
djangocloud env list
```

`env push` reads a `.env` file and saves the variables on the project, encrypted. It prints names only, never values.

* **It adds and overwrites; it does not remove.** Variables you don't send are left alone. Add `--prune` to remove everything you didn't send. It asks first, and it never removes the variables DjangoCloud sets itself, such as `DJANGOCLOUD_HOSTED_DB_*` for a database it created, `DJANGOCLOUD_QUEUE_*` for a task queue and `DJANGOCLOUD_STORAGE_*` for file storage.
* **All or nothing.** If a line is invalid, nothing is sent and the error names the line.
* **Preview it** with `--dry-run`: it lists which names are new and which would be overwritten, and changes nothing.
* **Read from standard input** with `-` instead of a file name.

The file is read like a `.env` file: comments, `export KEY=value`, single and double quotes and quoted values that span several lines (a PEM key) all work. `$VAR` is not expanded.

`env remove NAME [NAME ...]` removes variables by name. Names that aren't set are ignored, and the variables DjangoCloud manages itself (`DJANGOCLOUD_HOSTED_DB_*`, `DJANGOCLOUD_QUEUE_*`, `DJANGOCLOUD_STORAGE_*`) are never removed. Like every change, it applies from the next deploy.

`env list` shows the names that are set. Values are never shown, by the CLI or by the API.

Both commands act on the linked project, or on `--project <slug>`. See [Deploy from CI](../../getting-started/ci/) for keeping the values in your CI secrets instead of a file.

## When changes take effect

Environment variables are part of a release. A change applies to the **next** release you deploy, not to the one that's running. After editing variables, deploy again.

Because each release keeps the variables it was built with, a [rollback](../releases-and-rollbacks/) restores the values from that moment.

## Typical settings

```text
DJANGO_SECRET_KEY=...
DJANGO_ALLOWED_HOSTS=myapp.example.com
DATABASE_URL=postgres://...
```

Your settings module has to read these from the environment. The build step runs `collectstatic` with a throwaway secret key, so your settings must import without real secrets present.
