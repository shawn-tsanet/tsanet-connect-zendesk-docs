---
description: >-
  Step-by-step setup of the TSANet Connect integration for Zendesk: ZIS
  registration, the inbound webhook, event handling, the ZAF sidebar app, and
  pre-launch testing.
---

# Installation Guide

This guide walks a Zendesk administrator through installing the TSANet Connect
integration from start to finish. Plan for roughly one focused day of work.

{% hint style="info" %}
Contact membership@tsanet.org to obtain credentials for the Beta environment
before you begin.
{% endhint %}

## Before You Start

### From TSANet

* **A dedicated API user account.** This account belongs to the integration, not
  to any individual person. Email membership@tsanet.org with the subject
  "API Credentials Request: Zendesk Integration" and ask for Beta credentials.
* **Your partner's TSANet ID.** To send a collaboration request to a specific
  partner, you need their TSANet company ID and department ID.

### From Zendesk

* **Administrator access.** You must be a Zendesk administrator, not just an
  agent.
* **A paid Zendesk plan, Suite Professional or higher.** Trial accounts cannot
  be used. Zendesk Integration Services (ZIS) is required, and it is not
  available on lower tiers.
* **Five custom ticket fields.** These store TSANet data on each ticket. Create
  them in Admin Center &gt; Objects and rules &gt; Tickets &gt; Fields.

| Field name | Type | What it stores |
| --- | --- | --- |
| TSANet Token | Text | The unique ID linking this ticket to a TSANet case |
| TSANet Tokens (Multi) | Text | A list of IDs for tickets with multiple cases |
| TSANet Status | Dropdown | Current status: OPEN, ACCEPTED, INFORMATION, REJECTED, CLOSED |
| TSANet Partner | Text | The name of the partner company you are collaborating with |
| TSANet Respond By | Date | The deadline by which you must respond (calendar date) |

{% hint style="warning" %}
After creating each field, note its numeric field ID from the field's URL. You
will enter these IDs into the ZAF app settings in Step 4.
{% endhint %}

> 📸 **Screenshot placeholder:** Creating a custom ticket field in Admin Center
> &gt; Objects and rules &gt; Tickets &gt; Fields.

## The Five Steps at a Glance

1. **Register Zendesk with TSANet's event system.** Create a ZIS integration
   container to hold all TSANet resources.
2. **Create the inbound webhook.** Get the address TSANet uses to send you
   events.
3. **Configure inbound event handling.** Decide how each type of TSANet event
   updates a ticket.
4. **Install the ZAF sidebar app.** Add the panel agents use on every ticket.
5. **Test everything before going live.** Run the full scenario end to end.

## Step 1 &mdash; Register Zendesk with TSANet's Event System

Create a **ZIS integration container**, a named bucket inside Zendesk's ZIS
platform that all TSANet resources will live in.

Generate a Zendesk API token in Admin Center &gt; Apps and integrations &gt;
APIs &gt; Zendesk API &gt; Add API token.

> 📸 **Screenshot placeholder:** The Add API token screen in Admin Center.

Then create the integration:

```
POST https://{your-subdomain}.zendesk.com/api/services/zis/registry/tsanet_connect
```

A `200 OK` response confirms the container was created. Save the full response
immediately. It contains OAuth credentials that are shown only once.

{% hint style="info" %}
A `409 Conflict` means the integration already exists. That is fine, continue
to Step 2.
{% endhint %}

## Step 2 &mdash; Create the Inbound Webhook

This is the address TSANet uses to reach you whenever something happens on one
of your collaborations.

### Step 2A &mdash; Get a ZIS OAuth token

Using the OAuth Client ID from the Step 1 response, request a ZIS Bearer token.
The response contains `token.full_token`. Save it for this step and Step 3.

### Step 2B &mdash; Create the inbound webhook

Create the webhook resource inside the integration container. The response
returns three values TSANet needs: a webhook URL, a username, and a password.

{% hint style="warning" %}
The username and password are shown only once. Save all three values
immediately, then send them to TSANet so inbound events can be delivered.
{% endhint %}

## Step 3 &mdash; Configure Inbound Event Handling

TSANet sends four types of events. Each one reaches a ticket through a defined
path.

| Event | What it means | How it reaches your ticket |
| --- | --- | --- |
| `collaboration.status.changed` | Partner accepted, rejected, or closed | Updates ticket status and adds a comment |
| `collaboration.note.created` | Partner posted a note | Adds an internal comment with the note text |
| `collaboration.information.requested` | Partner needs more info before accepting | Flags the ticket as ACTION REQUIRED and emails the agent |
| `collaboration.sla.breach` | The response deadline has passed | Adds a breach warning and tags the ticket |

{% hint style="info" %}
The INFORMATION event is the one most likely to be missed. Create a Zendesk
trigger that emails the assigned agent as soon as the `tsanet_action_required`
tag is added. Build it in Admin Center &gt; Objects and rules &gt; Business
rules &gt; Triggers.
{% endhint %}

> 📸 **Screenshot placeholder:** The Zendesk trigger that emails the assigned
> agent when an INFORMATION event arrives.

### Forward public replies to the partner (optional)

By default, partner notes flow **into** Zendesk. You can also send agent replies
back **out** to the partner automatically: when an agent posts a **public
reply** on a TSANet ticket, it is forwarded to the partner as a note. Internal
comments are never forwarded.

Set this up with one Zendesk webhook and one Zendesk trigger. The trigger fires
on a public comment (on tickets tagged `tsanet_inbound` or `tsanet_outbound`)
and sends the reply to a second ZIS inbound webhook, which posts it to the
partner. Only replies authored by an **agent or admin** are forwarded — an end
customer's public reply is not.

{% hint style="info" %}
With this in place, the ZAF app's **Add Note → Public** simply posts a public
reply and lets the trigger deliver it, so the partner receives each note once.
{% endhint %}

<details>

<summary>Server-side maintenance: the GitHub Actions workflow</summary>

The integration's scheduled work runs on a clock, independent of any agent's
browser:

* **TSANet authentication.** Current setups use a ZIS **OAuth client-credentials
  connection** (Microsoft Entra): ZIS mints and renews its own short-lived
  tokens, so no token-refresh job is needed. (Older setups stored a TSANet token
  in the ZIS connection and refreshed it every 60 minutes with a scheduled job —
  that pattern is now legacy.)
* **SLA monitoring.** A scheduled job pulls all open TSANet cases, finds any
  whose response deadline has passed, and tags the matching Zendesk ticket.

The reference implementation runs both as a GitHub Actions workflow on a cron
schedule. One-time setup is about 30 minutes: create a repository, add the
required repository secrets, drop in the workflow file, and run it once
manually to confirm both jobs succeed. After that it runs unattended.

</details>

## Step 4 &mdash; Install the ZAF Sidebar App

The ZAF sidebar app is the panel agents use on every ticket. It is distributed
as a pre-built ZIP and installed privately. There is no Zendesk Marketplace
listing.

* Visit the TSANet Connect releases page to get the latest version:
  [https://github.com/tsanetgit/Zendesk\_App/releases](https://github.com/tsanetgit/Zendesk_App/releases)
* Download the `tsanet-connect-vX.Y.Z.zip` asset from the latest release.
* In Admin Center &gt; Apps and integrations &gt; Zendesk Support apps, choose
  to upload a private app and select the ZIP.

> 📸 **Screenshot placeholder:** Uploading the private app ZIP in Admin Center
> &gt; Apps and integrations &gt; Zendesk Support apps.

* When prompted, enter the app settings:

| Setting | Value |
| --- | --- |
| TSANet API username | The dedicated API user account from "Before You Start" |
| TSANet API password | The password for that account |
| TSANet environment | `BETA` for setup and testing, `PRODUCTION` when live |
| Custom field IDs | The five numeric field IDs noted when you created the fields |

> 📸 **Screenshot placeholder:** The ZAF app settings screen with the TSANet
> credentials, environment, and custom field IDs filled in.

{% hint style="success" %}
Updating the app later is the same process: download the new ZIP and upload it
over the existing app. Settings are preserved across updates.
{% endhint %}

## Step 5 &mdash; Test Everything Before Going Live

Before real partner engineers receive your requests and real SLA clocks start,
run the full scenario as a fire drill.

<details>

<summary>The 7-test sequence</summary>

Run these in order. Each test builds on the previous one.

1. **Can you log in to TSANet?** Call the Beta login endpoint with your API
   credentials and confirm you receive a token.
2. **Can you find a partner?** Search for a test partner by name and confirm
   you get back a partner ID and department ID.
3. **Can you create a collaboration request?** Submit a test request via the
   API and confirm you receive a unique token.
4. **Does the ZIS inbound path work?** Send a simulated TSANet event to your
   webhook URL and confirm the correct ticket is updated.
5. **Does the INFORMATION alert work?** Send a simulated INFORMATION event and
   confirm the tag is added and the agent is emailed.
6. **Does the full agent flow work?** Have a real agent open a ticket, click
   New Collaboration, search for a partner, and submit.
7. **Does error handling work?** Submit with an incorrect partner ID and
   confirm the agent sees a clear, helpful error.

</details>

When all seven tests pass, contact TSANet to request Production credentials,
then update the app settings to use them.

## Authentication Summary

There are four authentication contexts in this integration.

| Context | Method | Where it is stored |
| --- | --- | --- |
| ZIS setup (Steps 1 to 3) | Zendesk Basic Auth | Used once, not stored in the app |
| ZAF app to Zendesk (runtime) | Inherited agent session via the ZAF SDK | Automatic, no credential needed |
| ZAF app to TSANet API (runtime) | JWT Bearer token from login | Zendesk app settings (encrypted) |
| ZIS to TSANet API (runtime) | OAuth client credentials (Microsoft Entra) | ZIS connection — minted and renewed automatically |

## Need Help

For credentials, environment access, or installation questions, contact
membership@tsanet.org.
