---
title: "Logs and metrics"
description: "See what your app is doing without opening the AWS console."
---

See what your app is doing without opening the AWS console.

## Logs

The **Logs** tab of a deployment shows these kinds of lines:

| Source | What it is |
| --- | --- |
| **App** | Everything your app writes to standard output and standard error. Use a console log handler. |
| **Build** | The output of building your image, including the `[djangocloud]` notes about [your settings](../how-your-app-runs/). |
| **Release** | What DjangoCloud did to roll your release out: deploying, health checks, and why a release failed. |
| **Worker** | What your [Celery worker](../../guides/task-queues-and-workers/) writes, when you have one. |

### Searching

* **Search** shows lines that contain your text. Separate several terms with commas or new lines, up to 10.
* **Exclude** hides lines that contain any of its terms, which is handy for silencing a noisy health check.
* **Time range:** the last 15 minutes, 1 hour, 6 hours, 24 hours, 7 days, or all, or a **custom range** (in UTC).
* **Level:** all levels, warnings and errors, or errors only. The level is read from the line's own text.
* **Source:** app, build, release or worker.
* **Live** keeps the view up to date every few seconds. Switch it off to read without it moving.

The page shows the newest 300 matching lines and tells you when there are more, so narrow the range or add a search term to get to older ones. Lines keep the time your app logged them.

### From the terminal

```bash
djangocloud logs                       # the latest 100 lines
djangocloud logs -f                    # keep streaming (Ctrl-C to stop)
djangocloud logs -n 500 --source app   # only your app's output
djangocloud logs --since 2h            # 90s, 30m, 2h or 7d
```

### How long logs are kept

Your app's log is collected every couple of minutes, so the newest lines can be a little behind. Lines are kept for **30 days**, and at most 20,000 lines per deployment, so one very noisy app can't fill the database. See [Known limitations](../../reference/limitations/).

## Metrics

The **Metrics** tab shows two charts: **CPU** and **Memory**, each as a percentage of what your [server size](../size-and-scaling/) allows. 100% means the server is fully used.

* **Ranges:** 1 hour, 6 hours, 24 hours, 7 days or 30 days.
* **Underneath each chart:** the latest value, and the average, peak and low for the range you chose.
* **Hover** over a chart to read it. A guide line, a dot on the line and a callout show the **time** (in your own timezone) and the **value**, so you can see exactly when the load was high. On a touch screen, touch and drag. With a keyboard, focus a chart and use the left and right arrow keys.
* **Averages on long ranges.** A chart draws at most 160 points. When a range has more samples than that, each point is the **average** of several, and the callout says so. The **Peak** under the chart always comes from the real samples, so it can be higher than any point you see.

Samples are taken **every 5 minutes** while your app is running and kept for **30 days**. A new deployment shows "No samples in this time range yet" until the first one arrives.

### In the deployments list

Each row of the **Deployments** list has two small graphs, CPU and Memory, for the last 24 hours. They have the same hover callout. The list refreshes every 10 seconds, and pauses while your pointer is on a graph so the callout doesn't disappear while you read it.

### Metrics from the terminal

```bash
djangocloud metrics              # the last hour
djangocloud metrics --since 6h
djangocloud metrics --json       # the samples, for scripts
```

It draws CPU and memory as small charts with the latest value, the peak and the average for the range. The samples are the same ones the Metrics tab uses: every 5 minutes, kept for 30 days, so `--since` can reach back at most 30 days.

### Status from the terminal

```bash
djangocloud status          # is it live and answering, its URL, size and latest releases
djangocloud status --json   # the same as JSON, for scripts
```
