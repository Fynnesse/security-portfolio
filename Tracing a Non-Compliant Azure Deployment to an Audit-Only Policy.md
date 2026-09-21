# Tracing an Unauthorised Weekend Deployment to an Audit-Only Policy in Azure

**Date:** 2026-09-21
**Environment type:** Live multi-user Azure training tenant
**Time spent:** ~[X]h

---

## Scenario

An intern was given temporary Contributor access to build a test environment for an experiment. They deployed over a weekend without following the organisation's governance standards, then left. I picked it up Monday morning as the on-call engineer with Reader access. The question: what was deployed, who deployed it, and why didn't the existing governance controls stop it?

## Environment

- Platform: Microsoft Azure
- Services in scope: Resource groups, Storage accounts, Azure Resource Manager deployments, Azure Policy
- Tools: Azure Portal
- Access level: Reader (observe only, no changes made)
- Environment: live multi-user Azure training tenant

## Investigation

**Starting hypothesis:** resources created without proper tags or naming, plus a misunderstanding of Azure Policy and resource locks. Possibly oversized VMs or spend beyond what the experiment needed.

### 1. Locate the resource group that doesn't belong

I started at the subscription's resource group list and compared every name against the pattern the rest followed. All of them used a consistent prefix convention except one. There were few enough resource groups to spot it by eye, which is the point of a naming convention: it makes an anomaly visible at a glance instead of something you have to hunt for.

![Resource group list showing the naming outlier](screenshots/01-rg-list-naming-outlier.png)

### 2. Inspect what's inside and who owns it

The outlier held a single resource, a storage account. I opened it and went to its tags to understand what it was for. The tags carried an owner value, which put a person behind the case rather than just a resource.

![Resource group contents](screenshots/02-rg-contents.png)
![Storage account tags, values redacted](screenshots/03-storage-tags.png)

### 3. Trace the deployment

Every IaC deployment leaves a record in the resource group's Deployments blade. There was a single deployment. Its name pointed to who ran it, and the blade showed when it ran and that it succeeded. That gives the starting point for an incident timeline.

![Deployments blade, deployment name redacted](screenshots/04-deployments.png)

### 4. Find out why governance didn't stop it

Under the resource group's Policies, I checked the assignments applying at this scope. Alongside several security initiatives there was one standalone policy: a naming convention, assigned at subscription level and therefore inherited by this resource group. Its Parameters tab showed the effect set to **Audit**.

Audit logs a violation but lets the request through. So the policy was assigned, it covered this resource group, and it did evaluate the deployment. It just wasn't configured to stop it.

![Naming convention policy assignment, effect set to Audit](screenshots/05-policy-effect-audit.png)

## What broke / what surprised me

- **My hypothesis was wider than the evidence.** I went in expecting oversized VMs or cost overruns alongside the naming and tagging problems. The deployment turned out to be a single storage account, so the sizing and spend angle didn't hold.
- **Navigation cost more than I expected.** Deployments and Policies both sit under Settings in the resource group menu, not at the top level. A few seconds each to find. Small in a lab, but on an incident clock the time spent working out where things live adds up. Worth knowing the portal layout before you need it.
- [ADD: anything else real from the session]

## Findings and recommendations

**Findings**

- A resource group and storage account were created outside the organisation's naming convention.
- The storage account carried an owner tag, allowing attribution to the person who deployed it.
- A single deployment created it, and it completed successfully.
- A naming convention policy was assigned at subscription scope and covered this resource group, but its effect was set to Audit. The violation was recorded, not prevented.

**Recommendations**

1. Change the naming convention policy effect from Audit to Deny, so non-compliant requests are rejected before the resource is created.
2. Review the policy assignments across the subscription to confirm each one uses the right definition and the right effect for its purpose.
3. Brief anyone granted temporary access on the organisation's governance standards before that access begins.

## What I learned

- Every IaC deployment leaves a record in the resource group's Deployments blade, which makes it the natural starting point for reconstructing who created what, and when.
- A policy being assigned and active is not the same as a policy being enforced. The effect decides whether a violation is blocked or just written down.
- [ADD: one thing you'd do differently next time]
