---
title: "Welcome"
description: "DjangoCloud deploys your Django project to AWS with one command."
template: splash
hero:
  title: "DjangoCloud documentation"
  tagline: "Deploy your Django project to AWS with one command, and keep every release so you can roll back."
  actions:
    - text: Quickstart
      link: ./getting-started/quickstart/
      icon: right-arrow
    - text: CLI reference
      link: ./reference/cli/
      variant: minimal
    - text: djangocloud-cli on PyPI
      link: https://pypi.org/project/djangocloud-cli/
      icon: external
      variant: minimal
---

DjangoCloud deploys your Django project to AWS with one command. You skip the Dockerfiles, load balancers, certificates and server setup, and keep every release saved so you can roll back.

```bash
pip install djangocloud-cli
djangocloud login
djangocloud deploy
```

The CLI is [`djangocloud-cli` on PyPI](https://pypi.org/project/djangocloud-cli/). Nothing to change in your project. Your static files, HTTPS and database migrations are taken care of.

## Where to start

* **New here?** Follow the [Quickstart](getting-started/quickstart/).
* **Curious how your app is run?** Read [How your app is run](how-it-works/how-your-app-runs/): uvicorn or gunicorn, static files, health checks and what we add to your settings.
* **Want deploys on every push?** [Connect GitHub](getting-started/github/).
* **Wondering what is and isn't supported?** Read the [known limitations](reference/limitations/) before you build on it.
* **Stuck?** Come to [Slack](community-and-support/support/).
