---
layout: post
title: "Auto-delete idle cloud stuff, but give people 48 hours"
date: 2026-11-08
tags: [azure, finops, cost, platform-engineering, python]
---

Pull the bill. Find a Function App on a Premium plan with zero executions in weeks. Ask in the channel who owns it. Nobody. Ask again. Somebody says it might be from a POC. Nobody wants to be the one who deletes it, so it keeps billing.

Multiply that by every subscription and that's where cloud waste actually lives. Not in some badly sized cluster. In stuff nobody remembers making.

At a bank I worked at I built cost governance for Azure Functions that cleaned this up automatically. Find idle apps, tell the owner, wait 48 hours, delete. The 48 hours is the whole design. Everything else is plumbing.

## why not just delete

The "delete anything idle" version lasts about one incident. The first time it kills something that runs once a month for a month-end report, the tool gets turned off and nobody's allowed to mention it again.

Flag-then-delete is different. The owner gets a message, they have two days to say "no, keep it," and if they don't, it goes. Nobody can say they weren't told. And most of the time the answer is silence, because the thing really was dead.

## what idle means

Careful here. Consumption plan apps with no traffic cost basically nothing. If you're chasing those you're optimizing the wrong thing. The money is in Premium and Dedicated plans, where you pay for instances whether anything runs or not. So the check is per app, but the reason you care is the plan underneath.

I use zero `FunctionExecutionCount` over a window. Pick the window off your longest legit schedule. If you have month-end batch stuff, two weeks isn't enough.

Two things that make an app look busy when it isn't:

**Timers.** A timer trigger fires on schedule whether or not it does anything useful. A function that wakes up every five minutes to check an empty queue will never look idle. You won't catch those with execution count. Those need a human, or a second check on whatever the function writes to.

**Health pingers.** Same thing for HTTP apps behind some uptime check. The pinger counts as traffic. Exclude it or you'll never flag anything.

## the reaper

Runs once a day. Report-only by default, because the first run is always a surprise.

```python
#!/usr/bin/env python3
# idle function app reaper. flag first, delete 48h later if nobody objects.
import os
from datetime import datetime, timedelta, timezone

from azure.identity import DefaultAzureCredential
from azure.mgmt.resource import ResourceManagementClient
from azure.mgmt.resource.resources.models import Tags, TagsPatchResource
from azure.mgmt.web import WebSiteManagementClient
from azure.monitor.query import MetricAggregationType, MetricsQueryClient

SUB = os.environ["AZURE_SUBSCRIPTION_ID"]
IDLE_DAYS = int(os.environ.get("IDLE_DAYS", "14"))
GRACE = timedelta(hours=48)
DRY_RUN = os.environ.get("DRY_RUN", "true") == "true"  # safe unless you say otherwise
FLAG = "idle-flagged-at"

cred = DefaultAzureCredential()
web = WebSiteManagementClient(cred, SUB)
arm = ResourceManagementClient(cred, SUB)
metrics = MetricsQueryClient(cred)
now = datetime.now(timezone.utc)


def executions(site_id):
    r = metrics.query_resource(
        site_id,
        metric_names=["FunctionExecutionCount"],
        timespan=timedelta(days=IDLE_DAYS),
        granularity=timedelta(days=1),
        aggregations=[MetricAggregationType.TOTAL],
    )
    return sum(p.total or 0 for m in r.metrics for ts in m.timeseries for p in ts.data)


def set_tags(site_id, op, tags):
    if DRY_RUN:
        print(f"  [dry run] tags {op} {tags}")
        return
    arm.tags.begin_update_at_scope(
        site_id, TagsPatchResource(operation=op, properties=Tags(tags=tags))
    ).result()


def notify(owner, msg):
    print(f"  -> {owner}: {msg}")
    if not DRY_RUN:
        send(owner, msg)  # slack, teams, email, whatever people read


for site in web.web_apps.list():
    if "functionapp" not in (site.kind or ""):
        continue
    tags = site.tags or {}

    # opt out, with an expiry. "keep forever" is how you got here.
    keep = tags.get("keep-until")
    if keep and datetime.fromisoformat(keep).replace(tzinfo=timezone.utc) > now:
        continue

    owner = tags.get("owner", "#cloud-cost")  # no owner tag, the channel hears about it
    flagged = tags.get(FLAG)

    if executions(site.id) > 0:
        if flagged:
            set_tags(site.id, "Delete", {FLAG: flagged})  # came back to life
        continue

    if not flagged:
        set_tags(site.id, "Merge", {FLAG: now.isoformat()})
        notify(owner, f"{site.name}: no executions in {IDLE_DAYS}d. "
                      f"deleting in 48h. set keep-until=YYYY-MM-DD to stop it.")
    elif now - datetime.fromisoformat(flagged) >= GRACE:
        notify(owner, f"{site.name}: deleted after 48h grace")
        if not DRY_RUN:
            web.web_apps.delete(site.resource_group, site.name)
```

Heads up, the Azure Python SDKs move around. The tags call was `update_at_scope` in older `azure-mgmt-resource`, and the metrics client has been getting split into its own package. Check what you've pinned.

The state lives on the resource as a tag. No database, no state file. If the reaper's identity can't write tags, it can't flag, so it can't delete. That's a nice property to have by accident.

## then the plan

Deleting the app doesn't delete the App Service plan. An empty Premium plan keeps billing. That's the part people miss, and it's the part that actually saves money.

```python
# second pass: plans with nothing on them
for plan in web.app_service_plans.list():
    if (plan.number_of_sites or 0) == 0:
        print(f"empty plan: {plan.name} ({plan.sku.tier})")
        # same flag/48h/delete dance as above
```

Same flag, same grace, same message. Don't skip the grace just because it's empty. Someone might be mid-redeploy.

## stuff I'd do the same again

**Report-only first.** Run it for a cycle just printing. You'll find apps with no owner tag, owner tags pointing at people who left, and at least one thing everyone swears is critical that hasn't run in a month. Sort that out before anything gets deleted.

**Expiring exemptions.** `keep-until` with a date, never a boolean. A boolean opt-out turns into a permanent one and you're back where you started.

**Can it come back from git?** Before turning deletes on, I wanted every flagged app to be redeployable from its pipeline. If deleting it means losing it, that's not a cost problem, that's a much worse problem you just found.

**Tell people what happened.** The delete message goes to the same place as the warning. Nobody should find out from a 404.

Cleanup tooling lives or dies on trust. Give people a warning and a way to say no, and they'll let it delete things. Skip that and they'll make you turn it off.
