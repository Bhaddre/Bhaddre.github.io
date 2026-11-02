---
layout: post
title: "Don't enforce a VPC Service Controls perimeter you haven't run in dry run"
date: 2026-10-29
tags: [gcp, vpc-service-controls, security, terraform, platform-engineering]
---

Enforce a VPC Service Controls perimeter cold and the first thing that breaks is usually your own Terraform. The state bucket is in GCS, GCS is now a restricted service, and the runner lives in a CI project that isn't in the perimeter. Plan fails with a 403 that says `Request is prohibited by organization's policy` and a `vpcServiceControlsUniqueIdentifier`. Nothing else. Good luck.

Then the next GKE node that comes up can't pull images, because Artifact Registry sits in a shared project outside the perimeter too. Now it's an incident.

I own the GCP org at my day job. We moved ~350 microservices from AWS to GCP across 6+ GKE projects, and every one of those projects ends up behind a perimeter. I wasn't going to learn what breaks by enforcing and waiting for Slack to tell me. Dry run first. Every time.

## what dry run actually does

A perimeter has two configs. `status` is what's enforced. `spec` is the dry-run config. When they differ, VPC SC evaluates every request against both, enforces `status`, and logs anything `spec` *would* have blocked. Nobody gets a 403 from the dry-run side. You just get a log line.

So the move is: put the real perimeter in `spec`, leave `status` empty, and read logs for a while.

```hcl
resource "google_access_context_manager_service_perimeter" "prod" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/servicePerimeters/prod"
  title  = "prod"

  # no status block yet = nothing enforced
  use_explicit_dry_run_spec = true

  spec {
    resources = [for n in var.prod_project_numbers : "projects/${n}"]

    restricted_services = [
      "storage.googleapis.com",
      "bigquery.googleapis.com",
      "artifactregistry.googleapis.com",
      "secretmanager.googleapis.com",
    ]

    # VMs and pods inside can only call restricted services, through the restricted VIP
    vpc_accessible_services {
      enable_restriction = true
      allowed_services   = ["RESTRICTED-SERVICES"]
    }

    # terraform runner in the CI project needs the state bucket
    ingress_policies {
      ingress_from {
        identities = ["serviceAccount:${var.tf_runner_sa}"]
        sources {
          resource = "projects/${var.ci_project_number}"
        }
      }
      ingress_to {
        resources = ["projects/${var.tf_state_project_number}"]
        operations {
          service_name = "storage.googleapis.com"
          method_selectors {
            method = "*"
          }
        }
      }
    }

    # nodes pull images from the shared registry project
    egress_policies {
      egress_from {
        identities = [for sa in var.gke_node_sas : "serviceAccount:${sa}"]
      }
      egress_to {
        resources = ["projects/${var.registry_project_number}"]
        operations {
          service_name = "artifactregistry.googleapis.com"
          method_selectors {
            method = "*"
          }
        }
      }
    }
  }
}
```

Resources go in as project *numbers*, not IDs. That one gets everybody once. Field names here are from the google provider I was on, double check against yours, the ingress/egress blocks have changed shape before.

Also heads up on `vpc_accessible_services`. Turning it on means stuff inside the perimeter can only reach restricted services. If your pods call some Google API you didn't list, that breaks too. Which is the point, but it's a second blast radius and it'll show up in the same logs.

## reading the violations

Dry-run violations land in the Policy Denied audit log, in the project that got called. With 6+ projects that's useless one at a time, so an org-level aggregated sink sends `cloudaudit.googleapis.com/policy` to one log bucket. Then:

```bash
gcloud logging read '
  log_id("cloudaudit.googleapis.com/policy")
  protoPayload.metadata.dryRun=true
  timestamp>="2026-10-01T00:00:00Z"
' --project=sec-logging --bucket=vpcsc --location=global --view=_AllLogs \
  --format=json --limit=50000 > dryrun.json
```

Raw, this is thousands of lines that all look the same. What you want is the unique set of who called what and why it would have failed:

```python
#!/usr/bin/env python3
# group dry-run VPC SC violations so a human can actually read them
import json
import sys
from collections import Counter

rows = Counter()
for e in json.load(open(sys.argv[1])):
    p = e.get("protoPayload", {})
    md = p.get("metadata", {})
    who = p.get("authenticationInfo", {}).get("principalEmail", "?")
    rows[(
        p.get("serviceName", "?"),
        p.get("methodName", "?"),
        who,
        md.get("violationReason", "?"),
    )] += 1

for (svc, method, who, reason), n in rows.most_common():
    print(f"{n:>7}  {reason:<40} {svc:<32} {method:<50} {who}")
```

That list is the work. Every row is either a rule you need to write or a thing that shouldn't be happening. Both are worth knowing before you enforce.

The common reasons: `RESOURCES_NOT_IN_SAME_SERVICE_PERIMETER` (cross-project call, you need egress/ingress or a bigger perimeter) and `NO_MATCHING_ACCESS_LEVEL` (something from outside with no ingress rule). Principal email is sometimes redacted depending on who's calling, so you'll be guessing from the method on a few.

## stuff the logs turned up

Not a complete list. Just the ones I'd check first next time.

**Terraform state and CI.** Covered above. Fix it before anything else or you can't ship the fix for anything else.

**Image pulls.** Node service accounts calling Artifact Registry across the perimeter. Easy to miss because pulls only happen when a node or pod is new. If your dry-run window has no node churn, you won't see it. Roll a node pool on purpose during dry run.

**Log sinks and BigQuery exports.** Sinks write as their own writer identity, not as you. That identity needs an ingress rule on the destination side. Shows up as a weird service account email you've never seen.

**Google-managed agents.** Cloud Build, Dataflow, the various `service-<number>@...` accounts. They act on your behalf from Google's side and some of them need explicit ingress. Read the per-product VPC SC docs for anything that does this.

**Humans on laptops.** Console and `gcloud` from outside count as outside. Either an access level for the corp network/devices or people lose the console for anything restricted. Decide that up front, don't find out from a VP.

## when to flip it

I don't enforce until a full business cycle of dry run comes back clean. Week minimum, longer if there's month-end batch stuff. Then I copy `spec` into `status` in the same PR, so what's enforced is byte for byte what was tested.

After that, any change to the perimeter goes back through dry run first. Keeping both configs in Terraform makes that a normal PR instead of a special event.

Enforcing VPC SC isn't the hard part. Knowing every cross-project call your platform makes is, and dry run is the only cheap way to find out.
