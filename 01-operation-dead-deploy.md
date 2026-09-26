# Tracing a Non-Compliant Azure Deployment to an Audit-Only Policy

> [!NOTE]
> Identifiers and lab answers are redacted throughout, both in the text and in screenshots. This preserves the integrity of the lab for others and follows standard practice for handling environment data.
> 
## Scenario

An intern was given temporary Contributor access to build a test environment for an experiment. They deployed over a weekend without following the organisation's governance standards, then left. I picked it up Monday morning as the on-call engineer with Reader access.

The questions of interest: what was deployed, who deployed it, and why didn't the existing governance controls stop it?

## Environment

- Platform: Microsoft Azure
- Services in scope: Resource groups, Storage accounts, Azure Resource Manager deployments, Azure Policy
- Tools: Azure Portal
- Access level: Reader (observe only, no changes made)
- Environment: live multi-user Azure training tenant
- Date: 2026-09-21

## Investigation

**Starting hypothesis:** resources created without proper tags or naming, plus a misunderstanding of Azure Policy and resource locks. Possibly oversized VMs or spend beyond what the experiment needed.

### 1. Locate the resource group that doesn't belong

I started at the subscription's resource group list and compared every name against the pattern the rest followed. All of them used a consistent prefix convention except one. There were few enough resource groups to spot it by eye, which is the point of a naming convention: it makes an anomaly visible at a glance instead of something you have to hunt for.


<img width="1917" height="862" alt="1" src="https://github.com/user-attachments/assets/8f60734a-f3a0-43bb-b41d-a200b35e3388" />

<img width="1915" height="870" alt="2  Wrong Naming Convention" src="https://github.com/user-attachments/assets/6bb17341-624b-438c-9fee-d2d2295c93dc" />

*Resource group list showing the naming outlier*

### 2. Inspect what's inside and who owns it

The outlier held a single resource, a storage account. I opened it and went to its tags to understand what it was for. The tags carried an owner value, which put a person behind the case rather than just a resource.

<img width="1912" height="867" alt="3  Intern RG resources" src="https://github.com/user-attachments/assets/dfedf77b-4b79-4433-bcab-3445ffee7fdb" />

*Resource group contents*

<img width="1917" height="862" alt="5  Interns Resource Tags" src="https://github.com/user-attachments/assets/f674e9ae-46f1-44fc-995e-d32ef295338f" />

*Storage account tags*

### 3. Trace the deployment

Every IaC deployment leaves a record in the resource group's Deployments blade. There was a single deployment. Its name pointed to who ran it, and the blade showed when it ran and that it succeeded. That gives the starting point for an incident timeline.

<img width="1917" height="877" alt="6  RG Deployment" src="https://github.com/user-attachments/assets/f02f9f59-82c4-46a1-a4c1-92643334d984" />


<img width="1917" height="862" alt="7  RG Deployment overview" src="https://github.com/user-attachments/assets/b49d4b9c-807d-4ef1-b2ac-2141dbd385c9" />

*Deployments blade, deployment name*

### 4. Find out why governance didn't stop it

Under the resource group's Policies, I checked the assignments applying at this scope. Alongside several security initiatives there was one standalone policy: a naming convention, assigned at subscription level and therefore inherited by this resource group. Its Parameters tab showed the effect set to **Audit**.

Audit logs a violation but lets the request through. So the policy was assigned, it covered this resource group, and it did evaluate the deployment. It just wasn't configured to stop it.

<img width="1917" height="877" alt="8  RG Policies" src="https://github.com/user-attachments/assets/386dcd87-3850-4963-9e33-e7e51312636a" />


<img width="1916" height="867" alt="9  RG Naming Convention Policy" src="https://github.com/user-attachments/assets/765d883e-9e22-476b-88bd-203f441a989a" />

*Naming convention policy assignment, effect set to Audit*

## What broke / what surprised me

- **My hypothesis was wider than the evidence.** I went in expecting oversized VMs or cost overruns alongside the naming and tagging problems. The deployment turned out to be a single storage account, so the sizing and spend angle didn't hold.
- **Navigation cost more than I expected.** Deployments and Policies both sit under Settings in the resource group menu, not at the top level. A few seconds each to find. Small in a lab, but on an incident clock the time spent working out where things live adds up. Worth knowing the portal layout before you need it.

## Findings and recommendations

**Findings**

1. A resource group and storage account were created outside the organisation's naming convention.
2. The storage account carried an owner tag, allowing attribution to the person who deployed it.
3. A single deployment created it, and it completed successfully.
4. A naming convention policy was assigned at subscription scope and covered this resource group, but its effect was set to Audit. The violation was recorded, not prevented.

**Recommendations**
1. Determine if the naming policy effect was intentionally set to audit mode or if it was unintentional.
2. Change the naming convention policy effect from Audit to Deny, so non-compliant requests are rejected before the resource is created.
3. Review the policy assignments across the subscription to confirm each one uses the right definition and the right effect for its purpose.
4. Brief anyone granted temporary access on the organisation's governance standards before that access begins.

## What I learned

1. Every IaC deployment leaves a record in the resource group's Deployments blade, which makes it the natural starting point for reconstructing who created what, and when.
2. A policy being assigned and active is not the same as a policy being enforced. The effect decides whether a violation is blocked or just written down.
3. Tags and metadata of any resource,resource group, management group or subscriptions always tell a story.
