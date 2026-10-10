---
title: "Quickstart"
description: "From an existing Django project to a live URL in five steps."
---

From an existing Django project to a live URL in five steps.

<div style="aspect-ratio:16/9;max-width:100%;margin:1.5rem 0"><iframe src="https://www.youtube.com/embed/R51096ZB1sk" title="Deploy a brand-new Django project to AWS with Django Cloud" loading="lazy" style="width:100%;height:100%;border:0;border-radius:8px" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></div>

1. **Create your account** at [djangocloud.dev](https://djangocloud.dev) and add a card. Choose a plan: **Self-hosted** runs the app in your own AWS account. **Fully managed** is run by us in a dedicated AWS account, so there is no AWS account to connect.
2. **[Connect your AWS account](../connect-aws/)** (Self-hosted only). Create a limited IAM role that trusts DjangoCloud (one CloudFormation stack) and paste its ARN into the dashboard. An IAM access key also works. On Fully managed, skip this step.
3. **Install the CLI** ([`djangocloud-cli` on PyPI](https://pypi.org/project/djangocloud-cli/)) in your project's environment:

   ```bash
   pip install djangocloud-cli
   ```

4. **Sign in** from your terminal. The CLI shows a short code that you approve in the browser, so no password is typed into the terminal:

   ```bash
   djangocloud login
   ```

5. **Deploy:**

   ```bash
   djangocloud deploy
   ```

   On the first run the CLI asks which project this folder deploys to, or creates one, and shows the server size before it continues. Nothing is created until you confirm.

When the release is live the CLI prints its URL. Every later `deploy` skips the questions.

The commands work as soon as the package is installed, with nothing to add to your project. If you prefer `python manage.py djangocloud ...`, add `"djangocloud_cli"` to `INSTALLED_APPS` first.

## What you need

* Python 3.10 or newer and Django 4.2 or newer.
* An ASGI or WSGI application, for example `config.asgi:application` or `config.wsgi:application`. The CLI detects it, and you can [change it in the build settings](../../how-it-works/build-settings/).
* Your dependencies in `requirements.txt`, or in `pyproject.toml` and `uv.lock` if you use uv.

## Next

* Set [environment variables](../../how-it-works/environment-variables/) in the dashboard. Your `.env` file is never uploaded.
* Read about the [ephemeral container](../../how-it-works/ephemeral-container/) before you store uploads on disk.
* Connect [GitHub](../github/) to deploy on every push.
* Add your own address with [custom domains](../../guides/custom-domains/).
* Watch your app in [logs and metrics](../../how-it-works/logs-and-metrics/).
