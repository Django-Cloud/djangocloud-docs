---
title: "How your app is run"
description: "DjangoCloud runs your normal Django project in a container. You don't write a Dockerfile or a server config: we generate both from your build settings."
---

DjangoCloud runs your normal Django project in a container. You don't write a Dockerfile or a server config: we generate both from your [build settings](../build-settings/). This page shows exactly what we generate, what we add to your settings, and how your app is started, so nothing about your production environment is a mystery.

## The short version

| | |
| --- | --- |
| **Web server** | **uvicorn** when your project has an ASGI app (every `startproject` project does). **gunicorn** when it only has a WSGI app. |
| **Workers** | 2 by default (`workers`). |
| **Port** | 8000 by default (`port`). |
| **HTTPS** | Ends at our load balancer. Your app receives plain HTTP plus an `X-Forwarded-Proto` header, and is set up to read it. |
| **Static files** | `collectstatic` runs during the build, and **WhiteNoise** serves the files unless something else already does. |
| **Database migrations** | `release_command` (default `python manage.py migrate --noinput`) runs every time a container starts. |
| **Health check** | `GET` on `healthcheck_path` (default `/`), every 10 seconds. |
| **Files you write to disk** | Lost on every restart and deploy. See [the ephemeral container](../ephemeral-container/). |

## 1. The image

Your image is built from this recipe. This is the real output for a project with an ASGI app, a `requirements.txt` and one system package. It is shortened for reading: the wrapper's source and the long `collectstatic` command are abbreviated.

```dockerfile
FROM python:3.13-slim
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 PIP_NO_CACHE_DIR=1
ENV DJANGOCLOUD_BASE_SETTINGS=config.settings
ENV DJANGOCLOUD_STATIC_FILES=auto
ENV DJANGO_SETTINGS_MODULE=djangocloud_settings
RUN apt-get update && apt-get install -y --no-install-recommends libpq-dev && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY ./requirements.txt ./requirements.txt
RUN pip install -r requirements.txt "uvicorn[standard]" whitenoise
COPY ./ ./
COPY <<'PYSHIM' /app/djangocloud_settings.py    # the settings wrapper, see below
RUN DJANGOCLOUD_BUILD_NOTES=1 DJANGO_SECRET_KEY=build-only SECRET_KEY=build-only sh -c '... python manage.py collectstatic --noinput ...'
EXPOSE 8000
CMD ["sh", "-c", "python manage.py migrate --noinput && exec uvicorn config.asgi:application --host 0.0.0.0 --port 8000 --workers 2 --proxy-headers --forwarded-allow-ips '*'"]
```

In order:

1. **Base image.** The official `python:<version>-slim` image. Pick the version with `python_version` (3.10 to 3.13).
2. **System packages.** Anything in `system_packages` is installed with `apt-get`, for example `libpq-dev` or `ffmpeg`.
3. **Your dependencies.** With **pip**, from `requirements_file`. With **uv**, from `pyproject.toml` and `uv.lock` (`uv sync --frozen --no-dev`).
4. **The server, installed for you.** `uvicorn[standard]` or `gunicorn`, depending on which one starts your app, so it does not need to be in your requirements. `whitenoise` is installed too unless you turned `static_files` off.
5. **Your code**, copied in, plus the small settings wrapper described next.
6. **`collectstatic`.** Runs during the build with a throwaway secret key (`build-only`) that exists only in that build layer. Your settings therefore have to import without real secrets. If your project doesn't use `django.contrib.staticfiles` (an API-only app) this step is skipped. Any other failure, such as settings that won't import, fails the build and the log says why.

## 2. The settings wrapper

The container doesn't use your settings module directly. It sets `DJANGO_SETTINGS_MODULE=djangocloud_settings`, a tiny module we add that **imports your own settings first** and then adjusts four things. It makes each decision when the settings load, by looking at what you configured, and prints the reason to your build log with a `[djangocloud]` prefix.

If anything about your settings surprises it, it leaves them exactly as you wrote them: the wrapper is never the reason an app doesn't start.

### Static files

When nothing else serves your static files, WhiteNoise is added to `MIDDLEWARE` right after `SecurityMiddleware`. If you haven't set them, `STATIC_URL` becomes `/static/` and `STATIC_ROOT` becomes `staticfiles/` in your app folder, and files are served with WhiteNoise's compressed storage.

It does **not** add WhiteNoise when:

* `static_files` is `"off"` in your build settings,
* `django.contrib.staticfiles` is not in `INSTALLED_APPS`,
* WhiteNoise is already in your `MIDDLEWARE`,
* `STATIC_URL` points at another host, such as a CDN, or
* you use another static files storage, such as S3.

### Hosts and the health check

* `ALLOWED_HOSTS` is extended with the addresses your app is served on: its public URL and any [custom domains](../../guides/custom-domains/). You don't have to add them.
* Our load balancer calls the container by its **private IP address** for health checks, which Django would normally refuse with a 400. A small middleware answers those as if they were addressed to your app, so the health check passes and your log isn't filled with errors. Requests for any other host are still checked against `ALLOWED_HOSTS`.
* If your `ALLOWED_HOSTS` already contains `"*"`, nothing is changed.

### CSRF

`CSRF_TRUSTED_ORIGINS` is extended with `https://` plus each of your app's addresses, because Django 4 and later refuses form posts from an HTTPS page whose origin isn't trusted.

### HTTPS behind the load balancer

If you haven't set it, `SECURE_PROXY_SSL_HEADER` becomes `("HTTP_X_FORWARDED_PROTO", "https")`. Without it Django thinks every request is plain HTTP, builds `http://` links, and `SECURE_SSL_REDIRECT` redirects forever.

:::note
Read your build log to see each decision. For example: `[djangocloud] WhiteNoise added: your app now serves its own static files from /app/staticfiles.` or `[djangocloud] Not adding WhiteNoise: WhiteNoise is already in your MIDDLEWARE.`
:::

## 3. Starting your app

The container runs your **release command** and then your **server**, in one shell command:

```text
sh -c "<release_command> && exec <server command>"
```

### uvicorn (ASGI)

```text
uvicorn config.asgi:application --host 0.0.0.0 --port 8000 --workers 2 --proxy-headers --forwarded-allow-ips '*'
```

Websockets, async views and Channels work. `--proxy-headers` and `--forwarded-allow-ips '*'` make uvicorn trust the `X-Forwarded-*` headers from the load balancer. That is safe because the load balancer is the only thing that can reach your container.

### gunicorn (WSGI)

```text
gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 2
```

Gunicorn's own defaults apply, including its 30 second worker timeout.

### Which one is used

* With `server` set to `auto` (the default), **uvicorn** is used when `asgi_module` is set, and gunicorn otherwise.
* The CLI detects your modules. It keeps **gunicorn** when your `wsgi.py` does more than `application = get_wsgi_application()`, for example wraps the app in WhiteNoise or Sentry, because ASGI would skip that wrapper. It tells you when it does.
* Set `server` to `uvicorn` or `gunicorn` to choose yourself, or set `start_command` to run something else entirely, such as Daphne or Hypercorn. Its server must then be in your requirements, and testing it is up to you.

### The release command runs on every start

The release command is part of the container's start command, so it runs **every time a container starts**: on each deploy, after a restart, and on every instance when you run more than one.

* Keep it **safe to repeat**. `migrate` is. A command that sends emails or imports data is not.
* With several instances, each runs `migrate` at the same moment on a new deploy. Concurrent migrations can collide on a busy schema. If that worries you, set `release_command` to an empty string and run migrations as part of a deploy step you control.
* If the release command fails, the container exits and never becomes healthy. The deployment is marked **failed** and the previous release keeps serving. The last lines of the output appear in the **Logs** tab.

## 4. Health checks and rollouts

The load balancer requests `healthcheck_path` on your container:

| | |
| --- | --- |
| Interval | every 10 seconds |
| Timeout | 5 seconds |
| Counts as healthy | any response from 200 to 399 |
| Becomes healthy after | 2 successes in a row |
| Becomes unhealthy after | 5 failures in a row |

A new release starts alongside the old one. Traffic only moves once the new one is healthy, so if it never is (a crash, a failing migration, a bad health check path) the old release keeps serving. After AWS reports the new deployment active, DjangoCloud also checks that your **public URL** answers before it marks the release active.

Pick a `healthcheck_path` that returns quickly and doesn't need a login or a database round trip you'd regret.

## 5. What's in your container's environment

* Your own [environment variables](../environment-variables/), decrypted at deploy time.
* `DJANGO_SETTINGS_MODULE=djangocloud_settings`, `DJANGOCLOUD_BASE_SETTINGS` (your settings module), `DJANGOCLOUD_STATIC_FILES` and `DJANGOCLOUD_ALLOWED_HOSTS` (your app's addresses). The last one is refreshed when you add or remove a domain, without a rebuild.
* `PYTHONUNBUFFERED=1` so log lines appear right away.

:::caution
Don't set `DJANGO_SETTINGS_MODULE` yourself as an environment variable. Yours would replace ours, the wrapper would be skipped, and the host, CSRF, HTTPS and static file help above would switch off. Use the `django_settings_module` [build setting](../build-settings/) instead.
:::

## 6. Logs

Everything your app writes to standard output and standard error is collected into the **Logs** tab and `djangocloud logs`. Use a console log handler. Anything written only to a file inside the container is lost on restart and never appears there. See [Logs and metrics](../logs-and-metrics/).

## What isn't there

* No SSH or shell access to the container.
* No persistent disk and no shared filesystem between instances.
* No built-in cron. A [Celery worker](../../guides/task-queues-and-workers/) is available, see [background tasks](../../guides/background-tasks/).
