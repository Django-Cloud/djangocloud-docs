---
title: "Server size and scaling"
description: "Each deployment runs on one or more identical instances. You choose the size of an instance and how many you run."
---

Each deployment runs on one or more identical **instances**. You choose the size of an instance and how many you run.

## Sizes

| Size | vCPU | RAM |
| --- | --- | --- |
| Nano | 0.25 (shared) | 512 MB |
| Micro | 0.25 (shared) | 1 GB |
| Small | 0.5 (shared) | 1 GB |
| Medium | 1 | 2 GB |
| Large | 2 | 4 GB |
| XLarge | 4 | 8 GB |

## Changing size or instance count

Open your deployment, go to **Settings**, and choose a size and a number of instances between **1 and 20**. The change is recorded straight away and then applied to the running service in the background, so you don't need to deploy again. On Fully managed the new size is billed first. If applying it fails, the deployment keeps its current size and the reason is shown.

### From the command line

```bash
djangocloud scale                          # pick a size from a list, then the number of instances
djangocloud scale --size small --instances 3
```

The CLI shows the old and the new size, asks to confirm (`-y` skips that), and then waits until the change is applied. `--no-wait` returns as soon as it is queued, and `djangocloud status` shows the size at any time. Leave out `--size` or `--instances` to keep the current value of that one. Both are limited to the sizes above and 1 to 20 instances, and only one change can run at a time. In [CI](../../getting-started/ci/) give both flags, with `--no-input` in front: `djangocloud --no-input scale --size small --instances 3 -y`. See the [CLI reference](../../reference/cli/) for the flags.

### Autoscaling

Autoscaling adds and removes instances for you, between a minimum and a maximum you choose.

```bash
djangocloud autoscale on --min 2 --max 6
djangocloud autoscale off          # keeps the range for next time
djangocloud autoscale              # show it
```

You can also set it under **Settings** in the dashboard. It is deliberately slow and careful:

* **Out:** one more instance when the CPU averaged over 5 minutes is above 70% (or memory above 85%), at most every 5 minutes.
* **In:** one fewer instance when the CPU stayed below 30% for the whole last 30 minutes, and the remaining instances would still sit under 60% carrying the same load, at most every 15 minutes.
* It never goes outside your minimum and maximum, and never while a size change is already running.
* It reads the same samples as the [Metrics](../logs-and-metrics/) tab, which arrive every few minutes, so it reacts in minutes, not seconds.

If a scaling change fails, for example because a card is declined, autoscaling switches itself **off** and says why, instead of retrying and billing in a loop.

## Usage alerts

We email you when your app's CPU or memory stays high, and optionally when the server stops answering.

```bash
djangocloud alerts on --cpu 80 --memory 85 --downtime
djangocloud alerts off
djangocloud alerts                 # show the settings
```

* The **average over the last 10 minutes** has to be over your limit, so one busy minute is not an alert.
* You get one email when it starts, a reminder at most every 6 hours while it lasts, and the alert resets once usage is back under the limits.
* With `--downtime` you are also emailed when the server stops answering.

On Self-hosted, AWS bills you for the servers directly in your own AWS account. DjangoCloud does not charge for them.

## Choosing a size

* Start with **Nano** for a small app or a test.
* uvicorn (or gunicorn, for a WSGI-only app) runs a fixed number of workers (default 2, set with `workers` in the [build settings](../build-settings/)). Each worker uses memory, so a memory-hungry app needs a bigger size or fewer workers.
* If a release keeps restarting or being killed, it may be running out of memory. Try the next size up.

## More than one instance

Instances don't share a filesystem or memory. Sessions, caches and uploaded files have to live outside the container. See [Persistent files and the ephemeral container](../ephemeral-container/).
