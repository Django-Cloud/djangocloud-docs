---
title: "Backups and restoring your database"
description: "This page covers databases that DjangoCloud creates for you. They are managed PostgreSQL databases on Amazon Lightsail, in your AWS account."
---

This page covers databases that DjangoCloud creates for you. They are managed PostgreSQL databases on Amazon Lightsail, in your AWS account. If you bring your own database, backups are up to your database provider. See [Databases](../databases/).

## What is backed up

Lightsail takes **automatic backups** of your database. They let you restore to **any moment in the last 7 days**, in steps of 5 minutes. You can never restore to a moment before the database existed.

Your database page shows the exact window ("restorable to any moment up to ..."). A brand-new database has no backups for its first few minutes. The page tells you when there is nothing to restore from yet.

Backups live in the same AWS account as the database. They are not a copy somewhere else.

## Restoring to an earlier moment

A restore never changes your current database. It **creates a new database** from the backups, at the moment you choose.

1. Open your database page and choose **Restore to a new database**. Pick the moment (UTC).
2. Lightsail builds the new database. This takes up to about 15 minutes. **Your current database keeps running, and your app keeps using it.**
3. When the new database is ready, your connection variables (`DATABASE_URL` and the `DJANGOCLOUD_HOSTED_DB_*` variables) are updated to point at it. If you set your own `DATABASE_URL`, we leave it alone and only update the `DJANGOCLOUD_HOSTED_DB_*` variables.
4. Your app starts using the restored database **from your next deploy**.
5. The old database stays, untouched, until you delete it.

Your settings do not change. They read the same variables, which now point at the restored database.

### What you lose

A restore takes your data back to the moment you chose. **Anything written after that moment is not in the restored database.** The old database still has it until you delete it, so nothing is lost yet, but your app will not see it after the switch.

Your app keeps using the old database until you deploy, so anything written in the meantime also stays behind. Deploy soon after the restored database is ready.

### Choosing the moment

* To undo a bad migration or a bad deploy, choose a moment **just before it ran**. Restoring to the latest moment includes the damage.
* Times are in UTC. Check your release history for when the problem started.
* Your next deploy runs your release command (`python manage.py migrate --noinput` by default) against the restored database. If you restored to a time before a migration, that migration runs again.

### After the restore

Check that your app works on the restored database. Only then delete the old one. Deleting it asks you to type its name. It is **deleted immediately, with no final snapshot**, and there is no way to get it back.

## When is a final snapshot taken?

| What you do | Final snapshot? |
| --- | --- |
| Delete the database from its page | **No.** The data is gone for good. |
| Delete the previous database after a restore | **No.** The data is gone for good. |
| Delete the whole deployment | **Yes.** It is kept in your AWS account until you delete it. See [Deleting a deployment](../../how-it-works/deleting-and-disconnecting/). |

## Migrations and your data

* The old version of your app keeps serving **while a migration runs**, so a migration has to work with both the old and the new code. Add a column in one deploy and remove the old one in a later deploy.
* A migration that holds a lock on a busy table can make your app unresponsive until it finishes. DjangoCloud does not set a lock timeout for you.
* A rollback does **not** undo your data. We block a rollback to a release older than migrations that were already applied, unless you confirm. See [Releases and rollbacks](../../how-it-works/releases-and-rollbacks/).
* Before a risky migration, take a manual snapshot of the database in the Lightsail console in your AWS account, so you have a known restore point.
