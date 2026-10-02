# Reconstructing an OAuth Consent-Phishing Kill Chain Across Two App Registrations

> [!NOTE]
> Identifiers and lab answers are redacted throughout, both in the text and in screenshots. This preserves the integrity of the lab for others and follows standard practice for handling environment data.


---

## Scenario

Someone accessed the Mad Hat Labs tenant within the last 24 hours. No exploit was used and no alert fired. The logs show a series of ordinary, successful sign-ins. A legacy internal connector app registration had been flagged by the team as carrying configuration that was not there before. 

The question: reconstruct what the attacker did, stage by stage, using read-only access to directory applications, and identify what they would still hold after a standard containment.

## Environment

- Platform: Microsoft Azure / Microsoft Entra ID
- Services in scope: App registrations, service principals, Microsoft Graph permissions, OAuth consent
- Tools: Azure Portal
- Access level: Reader on directory apps (observe only, no changes made)
- Environment: live multi-user Azure training tenant
- Date: 2026-09-29

## Investigation

**Starting position:** I knew from the chapter that the attack had five stages before I opened anything, so this was a confirmation exercise rather than a blind discovery. What I was testing was whether I could locate each stage myself and explain why each one follows from the last.

### 1. Establish the access level I was working from

Before touching the flagged app I checked the Owned applications tab. The account owns nothing in the directory, which set the boundary for everything after: I could read app configuration but could not modify it.

<img width="1902" height="815" alt="2 7 2 Owned Apps" src="https://github.com/user-attachments/assets/e4070a16-da89-457a-8ea6-9640940a28ca" />

### 2. Entry

I opened App registrations and switched from the default Owned applications tab to All applications, which was the only way the flagged app became visible to me. Seven applications were registered in the tenant, with the legacy connector among them.

The incident team had recorded the entry method as free-text metadata in the Internal notes field under Branding and properties. The entry point was a phished human. The user completed MFA on a fake login page, and the attacker took the session token that was issued afterwards. Because that token already carried a claim saying MFA had been satisfied, Conditional Access evaluated the session and allowed it. Nothing failed and nothing alerted.

What made that user valuable was not their own access. Through years of drift they had been left as an Owner on the legacy connector app, which is what every later stage depends on.

<img width="1910" height="857" alt="2 7 3 All Apps" src="https://github.com/user-attachments/assets/21b75bbe-c6f0-4569-b368-39c2c2703eb2" />

<img width="1902" height="857" alt="2 7 5 Legacy app note flags" src="https://github.com/user-attachments/assets/717bfc46-d8ed-458d-a6b8-23072799ab55" />

### 3. Escalate

The API permissions blade is the answer to what the app can actually do. Application-type Graph permissions had been granted with admin consent, meaning they were live and tenant-wide. These were not granted by the attacker. They were granted when the connector was originally stood up, which means the escalation pre-dated the intrusion. The attacker only needed to become the app.


<img width="1917" height="817" alt="2 7 9 API_Permissions" src="https://github.com/user-attachments/assets/4015cc5f-d879-43f7-b043-0532c845fffc" />

### 4. Pivot

The Certificates and secrets blade held a single client secret with an expiry set to the end of the century. A secret authenticates the application itself through the client credentials flow, with no user involved, so from this point the attacker no longer needed a human session at all.

A credential alone is fragile, because rotation kills it. So the Owners blade is where the real persistence sits: a second app registration's service principal had been added as an owner of the legacy app. Ownership means the ability to mint fresh credentials indefinitely, so rotating the secret does not remove the attacker's access.

<img width="1916" height="865" alt="2 7 6 Legacy app Certs_and_secrets" src="https://github.com/user-attachments/assets/5cb8efc1-689e-426c-9083-bdad7f66d6cd" />

<img width="1917" height="825" alt="2 7 Owners list" src="https://github.com/user-attachments/assets/26a31896-9b7f-4d13-9b53-3b3d7a35bca2" />


### 5. Persist

The Expose an API blade turns the legacy app from a client into a resource, meaning other applications can request permission to call it. A custom scope had been published there. Unlike a secret or an ownership entry, a scope is configuration rather than a credential, so it survives both credential rotation and an owner review.

At this point the attacker holds no new access. A scope is a door, not a key. It only becomes access when a victim consents.

<img width="1917" height="872" alt="2 7 9 legacyapp_xposeAPI" src="https://github.com/user-attachments/assets/84299d20-0e6d-41f5-a9f8-d929a981542e" />

### 6. Loot

The rogue app's Authentication blade carried a redirect URI pointing at infrastructure outside the tenant, alongside a normal-looking development URI. The rogue app's client ID, the exposed scope, and that redirect URI together produce a working consent phishing URL. A victim already signed in on a corporate device clicks Accept, and the authorisation code is delivered to the attacker.

<img width="1917" height="882" alt="2 7 10 5TH FLAG" src="https://github.com/user-attachments/assets/5298dbd3-e560-48a9-9c63-3d4723c29ca0" />

### Why this path exists at all

This path exists for a few reasons. Credential phishing is expensive and repeatable in the wrong way. The attacker pays for a fake login page that eventually gets reported and blocked, the stolen session token expires on its own, and the user might change their password, so they would have to do the whole thing again and the user might not fall for it the second time. Consent phishing sidesteps all of that. The victim is already signed in on their own trusted device, so there is no rogue device, no impossible travel, and no MFA prompt to fail. Nothing anomalous happens, so nothing alerts. And what the attacker ends up with survives containment, because the grant sits outside the identity containment playbook. Resetting the password, revoking sessions and enforcing MFA do not touch it.

## Attack chain summary

| Stage | Where the evidence sat | What the attacker gained |
|---|---|---|
| 1. Entry | Branding and properties, internal notes | A session token carrying an MFA-satisfied claim, from a user who was an owner on the legacy app |
| 2. Escalate | API permissions | Tenant-wide Graph capability that already existed on the app |
| 3. Pivot | Certificates and secrets, then Owners | Authentication as the app itself, plus the ability to mint fresh credentials after any rotation |
| 4. Persist | Expose an API | A callable scope that survives credential rotation and owner removal |
| 5. Loot | Authentication, rogue app | A working consent phishing URL pointed at attacker infrastructure |

## Naming the pattern: confused deputy

A confused deputy attack is one where a less privileged actor tricks a more privileged system into performing an action on their behalf that they could not perform directly. The privileged system is not compromised. It does exactly what it was built to do, for a request it should never have honoured.

Here the legacy connector is the deputy. It holds tenant-wide Graph permissions and it is trusted to use them. The attacker never acquired those permissions themselves. They acquired the ability to make the connector act, first through a stolen session belonging to an owner, then through a credential minted on the app. Every Graph call that followed was the connector doing its job.

## What broke / what surprised me

- **Owning an app registration is its own privilege path.** It is not a directory role, so it does not appear in any review of privileged roles, yet it carries the ability to mint credentials on an app that holds tenant-wide permissions.
- **The Owners list on every app registration is editable, and attackers target it.** I had not thought of an ownership list as an attack surface before this.
- **Any member user can register an application by default.** The attacker did not need elevated rights to create the rogue app. That capability comes with being a member of the tenant.
- **Expose an API changes what the app fundamentally is.** It converts the legacy app from something that calls other services into something other services can call. I could see that the scope existed when I walked the blade, but I could not explain what it did or why it mattered until I went back and worked out the client versus resource distinction.
- **You cannot clear logs here the way you would on an endpoint.** There is no delete operation and no equivalent of a log-cleared event. Retention expires on its own schedule instead, which means waiting achieves what log wiping achieves, with no action to detect.

## Findings and recommendations

**Findings**

- A legacy connector app held application-type Microsoft Graph permissions with admin consent, granting tenant-wide capability that pre-dated the intrusion.
- A client secret on that app was set to expire at the end of the century.
- A second app registration's service principal was present on the legacy app's Owners list, providing credential-minting rights independent of any single secret.
- A custom scope had been published on the legacy app, turning it into a callable resource.
- A redirect URI on the rogue app pointed at infrastructure outside the tenant.
- The rogue app holds admin-equivalent capability through a chain of ownership and permissions while holding no directory role, so a review of privileged role membership would not surface it.

**Recommendations**

Ordered by the sequence they should be carried out in, not by severity alone. Removing the rogue owner comes first because while that service principal remains on the Owners list, the attacker can mint a replacement secret the moment the current one is revoked.

| # | Action | Priority | Owner |
|---|---|---|---|
| 1 | Remove the rogue service principal from the legacy app's Owners list | Critical | IAM |
| 2 | Revoke and remove all unauthorised client secrets and certificates on the affected application | Critical | IAM |
| 3 | Revoke existing OAuth grants explicitly, as standard containment does not remove them | Critical | IAM / SOC |
| 4 | Reset the password for the compromised account and revoke its sessions | High | Service Desk / IAM |
| 5 | Remove the malicious API scope from the legacy app | High | Application owners |
| 6 | Remove the unauthorised redirect URI from the rogue app | High | Application owners |
| 7 | Review and reduce the Graph application permissions on the legacy app to what the connector actually requires | High | IAM / Governance |
| 8 | Restrict user application registration so that creating an app is not a default member capability | Medium | IAM |
| 9 | Implement continuous monitoring and alerting on new client secrets, new owners, new redirect URIs and new exposed scopes | High | SOC / Detection engineering |

Two of these carry a cost worth stating. Reducing the Graph permissions on a legacy connector risks breaking a production sync, because in most cases nobody still knows what the connector genuinely needs. Restricting user application registration removes a capability developers currently use without asking, so it needs either an approval route or a named group that keeps the right.

## What I learned

- User-scoped Conditional Access only evaluates users. It does not apply to service principals. Once the attacker authenticated as the application with a client secret, there was no user, no device and no sign-in prompt for any user policy to act on, so a policy requiring MFA for all users had nothing to enforce against.
- Next time I would stop and understand a mechanism properly at the point I hit it, rather than noting what was there and moving on. I walked the Expose an API blade and recorded the custom scope without being able to explain what a scope actually did, which meant I could describe the finding but not its significance until afterwards. Taking the extra time up front would have made the investigation itself sharper, not just the write-up.
