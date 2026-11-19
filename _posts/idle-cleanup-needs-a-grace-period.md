---
layout: post
title: "Your idle resource cleanup needs a grace period, not a better query"
date: 2026-11-19
tags: [cost, finops, azure, python, platform-engineering]
---

Write an idle cleanup job that deletes the same day it decides something is idle, and the first thing it'll kill is a Function App that runs once a month. Month-end reconciliation, something like that. Zero executions for three weeks, so by the query it's idle. The team that owns it finds out on the one day it matters.

At a bank I worked at we had a lot of Azure Functions nobody remembered creating. Demo apps, one-off migrations, someone's proof of concept from two reorgs ago. Each one cheap. All of them together, not cheap. Plus every one of them is something with credentials and network access that no one is patching.

You can't fix that with a smarter idle query. The fix is not deleting anything the same day you decide it's idle. Ours waited 48 hours.

## the rule

Pretty simple once we settled on it:

- No executions in the lookback window, no `keep` tag: tag it `idle-since=<now>` and tell the owner.
- Still idle 48 hours after that tag: remove it.
- Gets traffic during the grace period: drop the tag, start over.
- Owner adds `keep=<reason>`: leave it alone. Forever, or until someone audits the keep tags.

The tag is the state. No database, no second system to keep in sync. Anyone can look at the resource in the portal and see why it's about to go away and when.

## the sweeper

Rough shape of it. Runs on a schedule per subscription.

```python
#!/usr/bin/env python3
# idle function app sweeper. tag first, delete after 48h. DRY_RUN until you trust it.
import os
from datetime import datetime, timedelta, timezone

from azure.identity import DefaultAzureCredential
from azure.mgmt.resource import ResourceManagementClient
from azure.mgmt.resource.resources.models import Tags, TagsPatchResource
from azure.mgmt.web import WebSiteManagementClient
from azure.monitor.query import MetricAggregationType, MetricsQueryClient

SUB = os.environ["AZURE_SUBSCRIPTION_ID"]
DRY_RUN = os.environ.get("DRY_RUN", "true") == "true"
LOOKBACK = timedelta(days=int(os.environ.get("LOOKBACK_DAYS", "14")))
GRACE = timedelta(hours=48)

cred = DefaultAzureCredential()
web = WebSiteManagementClient(cred, SUB)
res = ResourceManagementClient(cred, SUB)
metrics = MetricsQueryClient(cred)


def executions(site_id):
    # FunctionExecutionCount is what consumption plans report. check what your plan emits.
    r = metrics.query_resource(
        site_id,
        metric_names=["FunctionExecutionCount"],
        timespan=LOOKBACK,
        granularity=timedelta(days=1),
        aggregations=[MetricAggregationType.TOTAL],
    )
    return sum(
        p.total or 0
        for m in r.metrics
        for ts in m.timeseries
        for p in ts.data
    )


def patch_tags(site_id, op, tags):
    # older azure-mgmt-resource versions call this update_at_scope (not an LRO)
    res.tags.begin_update_at_scope(
        site_id, TagsPatchResource(operation=op, properties=Tags(tags=tags))
    ).result()


def notify(owner, msg):
    print(f"[notify {owner}] {msg}")  # ours went to slack, use whatever people read


now = datetime.now(timezone.utc)

for site in web.web_apps.list():
    if "functionapp" not in (site.kind or ""):
        continue
    tags = site.tags or {}
    if "keep" in tags:
        continue

    owner = tags.get("owner", "#cloud-cost")  # no owner tag, goes to the channel
    idle = executions(site.id) == 0
    since = tags.get("idle-since")

    if not idle:
        if since:
            print(f"{site.name}: traffic again, clearing idle-since")
            if not DRY_RUN:
                patch_tags(site.id, "Delete", {"idle-since": since})
        continue

    if not since:
        print(f"{site.name}: idle, starting grace period")
        if not DRY_RUN:
            patch_tags(site.id, "Merge", {"idle-since": now.isoformat()})
            notify(owner, f"{site.name} has had no executions in {LOOKBACK.days}d. "
                          f"Removing in 48h. Tag it keep=<reason> to stop this.")
        continue

    if now - datetime.fromisoformat(since) >= GRACE:
        print(f"{site.name}: grace period over, deleting")
        if not DRY_RUN:
            web.web_apps.delete(site.resource_group, site.name)
            notify(owner, f"{site.name} removed. Idle since {since}.")
```

Couple of things in there on purpose.

**Re-check at delete time.** It doesn't trust the tag alone. Every run recomputes executions, so if the thing woke up on hour 47, the tag gets cleared instead of the app getting deleted. The tag says "idle as of then," not "idle now."

**DRY_RUN defaults to true.** You flip it per subscription. First week in a new subscription it only prints. You read the list, you find the month-end job, you tag it `keep` yourself before anyone gets a scary message.

**No owner tag goes to a channel, not to nobody.** Plenty of apps had no owner tag. Those are exactly the ones you want a human to see before they disappear.

## stuff that bit us

**The lookback window decides who gets hurt.** Two weeks catches the obvious junk and misses anything monthly or quarterly. Make it long and you clean up almost nothing. I'd keep it shortish and lean on the grace period and `keep` tags to catch the periodic stuff, which is the whole point of having the grace period.

**48 hours over a weekend isn't 48 hours.** Tag on Friday afternoon, delete on Sunday, owner sees both messages Monday morning. Either don't start new grace periods late in the week or count business hours. Pick one.

**Executions isn't the only signal.** An app with a broken trigger also shows zero executions. Deleting it is probably still right, but the message to the owner should say "zero executions," not "unused." Sometimes it's broken and they didn't know.

**Deletes need to be boring to undo.** If the app was deployed from a pipeline, someone can redeploy it. If it was clicked together in the portal, it's gone. That's a reason to push people toward pipelines, not a reason to stop cleaning up. But know which one you're deleting.

**Watch the keep tags.** Give people a free "don't touch" tag and some of them will put it on everything. We did a pass now and then looking at `keep` values. `keep=yes` isn't a reason.

The code took an afternoon. Getting teams to trust it took the grace period. Nobody minds an automated cleanup that warns them first. Everybody minds the one that didn't.
