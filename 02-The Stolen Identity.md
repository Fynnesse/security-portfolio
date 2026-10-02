# Reconstructing an OAuth Consent-Phishing Kill Chain Across Two App Registrations

> [!NOTE]
> Identifiers and lab answers are redacted throughout, both in the text and in screenshots. This preserves the integrity of the lab for others and follows standard practice for handling environment data.

**Date:** 2026-09-29


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

*[screenshot: owned applications, empty]*

### 2. Entry

I opened App registrations and switched from the default Owned applications tab to All applications, which was the only way the flagged app became visible to me. Seven applications were registered in the tenant, with the legacy connector among them.

The incident team had recorded the entry method as free-text metadata in the Internal notes field under Branding and properties. The entry point was a phished human. The user completed MFA on a fake login page, and the attacker took the session token that was issued afterwards. Because that token already carried a claim saying MFA had been satisfied, Conditional Access evaluated the session and allowed it. Nothing failed and nothing alerted.

What made that user valuable was not their own access. Through years of drift they had been left as an Owner on the legacy connector app, which is what every later stage depends on.

*[screenshot: all applications list]*
*[screenshot: branding and properties, notes value redacted]*

### 3. Escalate

The API permissions blade is the answer to what the app can actually do. Application-type Graph permissions had been granted with admin consent, meaning they were live and tenant-wide. These were not granted by the attacker. They were granted when the connector was originally stood up, which means the escalation pre-dated the intrusion. The attacker only needed to become the app.

*[screenshot: API permissions, type and consent status visible]*

### 4. Pivot

The Certificates and secrets blade held a single client secret with an expiry set to the end of the century. A secret authenticates the application itself through the client credentials flow, with no user involved, so from this point the attacker no longer needed a human session at all.

A credential alone is fragile, because rotation kills it. So the Owners blade is where the real persistence sits: a second app registration's service principal had been added as an owner of the legacy app. Ownership means the ability to mint fresh credentials indefinitely, so rotating the secret does not remove the attacker's access.

*[screenshot: certificates and secrets, expiry column]*
*[screenshot: owners list showing the rogue app]*

### 5. Persist

The Expose an API blade turns the legacy app from a client into a resource, meaning other applications can request permission to call it. A custom scope had been published there. Unlike a secret or an ownership entry, a scope is configuration rather than a credential, so it survives both credential rotation and an owner review.

At this point the attacker holds no new access. A scope is a door, not a key. It only becomes access when a victim consents.

*[screenshot: expose an API, user consent display name cropped]*

### 6. Loot

The rogue app's Authentication blade carried a redirect URI pointing at infrastructure outside the tenant, alongside a normal-looking development URI. The rogue app's client ID, the exposed scope, and that redirect URI together produce a working consent phishing URL. A victim already signed in on a corporate device clicks Accept, and the authorisation code is delivered to the attacker.

*[screenshot: authentication blade, attacker URI redacted]*

### Why this path exists at all

This path exists for a few reasons. Credential phishing is expensive and repeatable in the wrong way. The attacker pays for a fake login page that eventually gets reported and blocked, the stolen session token expires on its own, and the user might change their password, so they would have to do the whole thing again and the user might not fall for it the second time. Consent phishing sidesteps all of that. The victim is already signed in on their own trusted device, so there is no rogue device, no impossible travel, and no MFA prompt to fail. Nothing anomalous happens, so nothing alerts. And what the attacker ends up with survives containment, because the grant sits outside the identity containment playbook. Resetting the password, revoking sessions and enforcing MFA do not touch it.

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

1. Remove the rogue service principal from the legacy app's Owners list.
2. Revoke and remove all unauthorised client secrets and certificates on the affected application.
3. Reset the password for the compromised account and revoke its sessions.
4. Revoke existing OAuth grants explicitly. Standard containment does not remove them.
5. Remove the malicious API scope from the legacy app.
6. Remove the unauthorised redirect URI from the rogue app.
7. Review and reduce the Graph application permissions on the legacy app to what the connector actually requires.
8. Restrict user application registration so that creating an app is not a default member capability.
9. Implement continuous monitoring and alerting on new client secrets, new owners, new redirect URIs, and new exposed scopes.

## What I learned

- User-scoped Conditional Access only evaluates users. It does not apply to service principals. Once the attacker authenticated as the application with a client secret, there was no user, no device and no sign-in prompt for any user policy to act on, so a policy requiring MFA for all users had nothing to enforce against.
- Next time I would stop and understand a mechanism properly at the point I hit it, rather than noting what was there and moving on. I walked the Expose an API blade and recorded the custom scope without being able to explain what a scope actually did, which meant I could describe the finding but not its significance until afterwards. Taking the extra time up front would have made the investigation itself sharper, not just the write-up.
