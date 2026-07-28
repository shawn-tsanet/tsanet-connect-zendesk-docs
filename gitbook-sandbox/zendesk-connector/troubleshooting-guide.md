---
description: >-
  Diagnose and fix common issues with the TSANet Connect integration for
  Zendesk: the sidebar app, inbound events, SLA alerts, ZIS, and the scheduled
  maintenance jobs. Organized by who hits the problem and what they see.
---

# Troubleshooting Guide

This guide helps you resolve the most common issues with the TSANet Connect integration. It is organized by audience so you can jump straight to your situation:

* [**For Agents**](#for-agents-using-the-sidebar) — you use the TSANet panel on tickets and something is not working.
* [**For Administrators — App & Settings**](#for-administrators-app-and-settings) — install, credentials, and custom fields.
* [**For Administrators — Inbound, SLA & Maintenance**](#for-administrators-inbound-sla-and-maintenance) — incoming requests, SLA alerts, ZIS, and the scheduled jobs.

{% hint style="info" %}
Most issues fall into one of three buckets: **credentials** (wrong or expired), **configuration** (a field ID or setting that does not match), or **the registered email domain** (the most common single cause). Check those three first.
{% endhint %}

## For Agents — Using the Sidebar

### The TSANet panel is missing or shows only a "+ New" bar

This is normal on tickets that have no TSANet collaboration yet — the panel collapses to a compact bar with a single **New Collaboration** button to stay out of your way. It expands automatically once a collaboration exists on the ticket.

If the panel does not appear at all on any ticket:

* Refresh the ticket. The app loads when the ticket view opens.
* Confirm the app is still installed and enabled in **Admin Center → Apps and integrations → Zendesk Support apps**.
* If it is installed but blank, the credentials may not be set — see the next item.

### "Credentials not configured"

The app cannot reach TSANet because the API username or password is missing or wrong in the app settings.

{% hint style="warning" %}
Only an administrator can fix this. Ask them to re-check the **TSANet API username** and **TSANet API password** in the app settings (Admin Center → Apps and integrations → Zendesk Support apps → TSANet Connect → app settings).
{% endhint %}

### New Collaboration search returns no partners

* The partner may not be a TSANet member, or may be listed under a slightly different company name. Try a shorter search term.
* A company with multiple departments returns multiple results — pick the right department.
* If no partner ever returns, confirm with your administrator that the app is pointed at the correct **environment** (`BETA` for testing, `PRODUCTION` when live) and that credentials are valid.

### A button (Accept, Reject, Request More Info, Add Note) seems to do nothing

The action buttons open a dialog inside the panel. If nothing happens:

* Wait a moment — the panel talks to TSANet over the network and can take a few seconds.
* Refresh the ticket and try again.
* If it still fails, note the exact button and case status and report it (see [Need Help](#need-help)).

### "Error processing request" when you Accept a case

This almost always means the **engineer email does not match your company's registered TSANet domain**.

{% hint style="danger" %}
TSANet requires the responding engineer's email to be on **your company's** registered domain (for example `you@yourcompany.com`) — never the customer's email and never a personal address. The app submits a configured company email on your behalf; if you see this error, your administrator needs to confirm the **TSANet API username** in the app settings is a valid address on your registered domain.
{% endhint %}

### The SLA countdown is missing on an open case

* The countdown only applies to the **first response**. Once a case is **Accepted**, **Rejected**, or moved to **Information**, the SLA clock stops by design — a missing countdown there is expected.
* On a case still in **Open** status, a missing countdown usually means TSANet has not set a response deadline, which depends on your group's SLA configuration in TSANet. Contact membership@tsanet.org if open cases never show a deadline.

### Partner notes are not showing up in the ticket

Notes posted by the partner are mirrored into the Zendesk ticket as internal comments, and the panel refreshes automatically about once a minute. If a note is missing:

* Give it a few minutes, then refresh the ticket — or click **Sync Now** in the panel to pull the latest immediately.
* Each note is mirrored only once, so you will not see duplicates; if you believe a note was sent but never arrived, report it.

### The Close button is missing on a case

Only the **submitting** company can close a collaboration. If you received the request (an **inbound** case), the Close button will not appear — the partner who opened it closes it when the work is done. This is expected, not a bug.

## For Administrators — App & Settings

### The app upload or update

The ZAF app is distributed as a pre-built ZIP and installed privately — there is no Zendesk Marketplace listing. Always download the latest `tsanet-connect-vX.Y.Z.zip` from the [releases page](https://github.com/tsanetgit/Zendesk_App/releases) and upload it under **Admin Center → Apps and integrations → Zendesk Support apps**.

{% hint style="success" %}
Updating is the same as installing: upload the new ZIP over the existing app. Your settings (credentials, environment, field IDs) are preserved across updates.
{% endhint %}

### Domain / "engineer email" validation errors

This is the single most common configuration problem.

* The **TSANet API username** in the app settings must be an account on **your company's** TSANet-registered domain.
* It must not be a customer's email and must not be a personal address.
* If agents report "Error processing request" on Accept, this setting is almost always the cause.

### Custom field issues (status not updating, deadline blank)

The app reads and writes five custom ticket fields. Each must exist, and its **numeric field ID** must be recorded in the app settings.

You do not enter these by hand. Open **TSANet Connect** from the left nav bar and click **Detect field IDs**: it reads the fields out of this instance, shows you what it matched, and writes the IDs into the app's settings when you click **Apply**. If a field was renamed, retyped, or duplicated, re-running Detect is the fix. It reports ambiguous or wrong-typed matches rather than guessing at them.

| If you see… | Check |
| --- | --- |
| Status never changes on the ticket | The **TSANet Status** field ID in settings matches the actual dropdown field, and the dropdown values exist |
| "Respond By" date never populates | The **TSANet Respond By** field is a **Date** field (not Date/time). Zendesk Date fields only accept a calendar date, and the app sets it accordingly |
| Token-based lookups fail | The **TSANet Token** field ID in settings matches the field, and the same ID is used by the maintenance jobs (see below) |

{% hint style="info" %}
Detect covers **every** ticket field the integration uses, including the two optional field-action ones (**TSANet Action** and **TSANet Action Text**). A field that does not exist on this instance is reported as not found and skipped, which is fine for the optional ones: the feature they belong to simply stays off. Create the field, re-run Detect, and click **Apply**.

The one setting Detect cannot fill is the **TSANet engineer email**. It is not a ticket field, so there is nothing to detect. Enter it by hand, and make sure it is on your member-registered domain, because TSANet's Accept endpoint rejects any other domain.
{% endhint %}

## For Administrators — Inbound, SLA & Maintenance

### Inbound requests are not creating or updating tickets

Incoming TSANet events reach Zendesk through the **inbound webhook** you created during installation. If inbound cases never appear:

* Confirm TSANet has the correct webhook **URL, username, and password** you generated during setup (these are shown only once — if they were lost, regenerate and resend them).
* Inbound cases are created **server-side** by the webhook push, so they appear even when no agent has Zendesk open. The ZAF app's background poller is a **fallback** that fills in if push is unavailable; it defers to push so no duplicate ticket is created. Check the browser console for the `[TSANet BG]` log prefix to confirm the fallback is running and credentials are set.

### Public replies are not reaching the partner

If agents post **public replies** but the partner does not receive them as notes:

* Confirm the **forwarding trigger and webhook** are configured (see the Installation Guide, "Forward public replies to the partner"). Without them, public replies stay in Zendesk.
* The trigger only forwards replies authored by an **agent or admin** — an end customer's public reply is not forwarded, by design.
* The ticket must carry the `tsanet_inbound` or `tsanet_outbound` tag for the trigger to fire.
* **Internal** comments are never forwarded — only public replies and **Add Note → Public** reach the partner.

### Agents miss "Information requested" events

The **Information** event (a partner needs more detail before accepting) is the one most often missed because it does not change ticket status on its own.

* Create a Zendesk trigger that emails the assigned agent when the `tsanet_action_required` tag is added (**Admin Center → Objects and rules → Business rules → Triggers**).
* Because inbound volume is low, also notify whoever owns inbound intake so requests are not missed.

### SLA breach emails fire repeatedly, or never fire

The breach alert is a Zendesk trigger that watches for the `tsanet_sla_breached` tag.

| Symptom | Fix |
| --- | --- |
| Trigger never fires | Confirm the ZAF background poller is running (an agent must have Zendesk open) and that its `TSANet Token` field ID matches the field the app uses |
| Trigger fires over and over for the same ticket | The trigger condition must use **Current tags** (`current_tags`), not **Tags**. The wrong field name causes a silent mismatch |
| Breaches reported on accepted cases | Expected: the TSANet SLA tracks the first response only. Once a case is accepted, rejected, or moved to Information, breach detection stops |

### The ZIS connection / token problems

Current setups use a ZIS **OAuth client-credentials connection** (Microsoft Entra) that mints and renews its own short-lived tokens — no refresh job is involved. (Older setups stored a 60-minute TSANet token in the ZIS connection and replaced it on a schedule.)

<details>

<summary>You cannot see the integration in Admin Center</summary>

ZIS custom integrations are **API-only** and do not appear in the Admin Center UI. Verify it with a direct API call:
`GET /api/services/zis/integrations/tsanet_connect/connections`. A 401/403/404 from a standard API token is expected (e.g. `connections/all` returns 403 "API token is not supported") — ZIS management endpoints require a ZIS OAuth token.

</details>

<details>

<summary>"Invalid client secret" (AADSTS7000215) through ZIS, but the same secret works directly</summary>

The stored secret is corrupted — usually a paste artifact such as a trimmed leading character or a merged line. Re-send it verbatim with `PATCH` (not `PUT`, which returns 405) to the OAuth client endpoint.

</details>

### Zendesk View columns for TSANet fields are missing

When you create a View to monitor active TSANet cases, the Zendesk Views API silently ignores custom-field columns. Add the TSANet columns **manually** in **Admin Center → Workspaces → Views** after the View is created. Outbound and inbound cases are tagged `tsanet_outbound` and `tsanet_inbound`, so you can also filter Views by those tags.

## Quick Diagnostic Checklist

When something is wrong and you are not sure where to start, work through these in order:

1. **Credentials** — are the TSANet username/password in the app settings correct, and is the **environment** (`BETA`/`PRODUCTION`) right?
2. **Domain** — is the TSANet API username on your company's registered domain?
3. **Field IDs** — do the five custom field IDs in the app settings match the actual fields? Re-run **Detect field IDs** to check and repair them.
4. **Webhook** — does TSANet have the correct inbound webhook URL, username, and password?
5. **Triggers** — do tag-based triggers use **Current tags**?

## Need Help

If you have worked through the relevant section above and the issue persists:

* For credentials, environment access, or partner/membership questions, contact **membership@tsanet.org**.
* To report a bug or request an enhancement in the app itself, open an issue at [https://github.com/tsanetgit/Zendesk\_App/issues](https://github.com/tsanetgit/Zendesk_App/issues).

When reporting, include: what you were doing, the exact error text or missing behavior, the case status and direction (inbound/outbound), and whether you are on `BETA` or `PRODUCTION`.
