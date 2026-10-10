---
title: "Install the CLI and deploy"
description: "The CLI is a normal Python package, djangocloud-cli on PyPI:"
---

The CLI is a normal Python package, [`djangocloud-cli` on PyPI](https://pypi.org/project/djangocloud-cli/):

<div style="aspect-ratio:16/9;max-width:100%;margin:1.5rem 0"><iframe src="https://www.youtube.com/embed/GPaCFVB2Sx0" title="Deploy an existing Django project with Django Cloud" loading="lazy" style="width:100%;height:100%;border:0;border-radius:8px" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></div>

```bash
pip install djangocloud-cli
djangocloud login
djangocloud deploy
```

There is nothing to change in your project. `djangocloud` works as soon as the package is installed, and still works when your settings won't load.

Prefer `python manage.py djangocloud ...`? Add `"djangocloud_cli"` to `INSTALLED_APPS` first, because Django only finds a management command in an installed app. See [CLI commands](../../reference/cli/).

## What a first deploy looks like

```text
$ djangocloud deploy
You're not signed in yet.
Open https://djangocloud.dev/dashboard/cli/?code=ABCD-EFGH and check that the code is ABCD-EFGH.
✓ Signed in as you@example.com
? Which project is this?  Create a new project
? Server size  Nano  0.25 vCPU, 0.5 GB RAM
? Create this server in your own AWS account? Yes
✓ Linked to my-shop (.djangocloud/config.json)
Deploying my-shop
✓ Wrote build settings to .djangocloud/config.json
  found wsgi_module = config.wsgi:application
  found asgi_module = config.asgi:application (started with uvicorn)
✓ Packed 148 files (212 KB). .env and .git are never uploaded.
✓ Uploaded. Release v1 started.
  Building v1
  v1 is live
✓ v1 is live.
```

What happens after the upload is described in [From code to a live URL](../../how-it-works/overview/), and exactly how your app is run is in [How your app is run](../../how-it-works/how-your-app-runs/).

## What gets uploaded

The folder is packed into one `.tar.gz`. In a git repository that is what git sees, so your `.gitignore` is respected. Whatever your `.gitignore` says, these are **never** uploaded:

* `.env` and `.env.*` (except `.env.example`)
* `.git`, virtualenvs and `node_modules`
* `*.sqlite3`, `*.pem` and `*.key`
* caches

The upload is limited to 100 MB. Set secrets as [environment variables](../../how-it-works/environment-variables/) in the dashboard, not in the upload.

## The `.djangocloud` folder

`.djangocloud/config.json` records which project the folder deploys to and holds your [build settings](../../how-it-works/build-settings/). It contains no secrets, so commit it: a CI checkout then knows its project (`djangocloud --no-input deploy` needs no flags), and if you [deploy from GitHub](../github/) the build can read your build settings. Versions up to 0.1.12 hid the folder with a `.gitignore` inside it; newer ones remove that file if the CLI wrote it.

## Signing in safely

Login uses a code you approve in the browser. The token is stored in `~/.config/djangocloud/credentials.json`, readable only by you. See and revoke tokens under **Settings → Developer** in the dashboard.
