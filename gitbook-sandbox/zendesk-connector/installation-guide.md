---
description: >-
  Step-by-step setup of the TSANet Connect integration for Zendesk: ZIS
  registration, the OAuth connection to TSANet, the ZIS flow bundle, the ZAF
  sidebar app, and pre-launch testing.
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
  This is what the ZAF app uses in Step 4.
* **A TSANet-issued Microsoft Entra client.** A separate credential from the API
  user account above: a client ID and secret, plus service principal
  onboarding — contact TSANet with your ZIS service principal's object ID. This
  is what ZIS uses in Step 2 to call the TSANet API on its own.
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
| TSANet Tokens (Multi) | Text | A list of IDs for tickets with multiple cases (used by the ZAF app only) |
| TSANet Status | Dropdown | Current status: OPEN, ACCEPTED, INFORMATION, REJECTED, CLOSED |
| TSANet Partner | Text | The name of the partner company you are collaborating with |
| TSANet Respond By | Date | The deadline by which you must respond (calendar date) |

{% hint style="warning" %}
After creating each field, note its numeric field ID from the field's URL. You
will substitute four of these (Token, Status, Partner, Respond By) into the ZIS
flow bundle in Step 3, and enter all five into the ZAF app settings in Step 4.
{% endhint %}

> 📸 **Screenshot placeholder:** Creating a custom ticket field in Admin Center
> &gt; Objects and rules &gt; Tickets &gt; Fields.

## The Five Steps at a Glance

1. **Register Zendesk with ZIS.** Create the API token, OAuth client, and
   integration container ZIS needs to manage itself.
2. **Connect ZIS to TSANet.** Register the Entra OAuth client-credentials
   connection that lets ZIS call the TSANet API without handling
   authentication itself.
3. **Deploy the ZIS flow bundle.** This is what actually creates and updates
   Zendesk tickets — steps 1 and 2 only set up the plumbing it runs on.
4. **Install the ZAF sidebar app.** Add the panel agents use on every ticket.
5. **Test everything before going live.** Run the full scenario end to end.

## Step 1 &mdash; Register Zendesk with ZIS

Three pieces of one-time Zendesk-side setup.

### 1a. Create a Zendesk API token

Admin Center &gt; Apps and integrations &gt; APIs &gt; Zendesk API &gt; Add API
token. Copy it immediately — you won't see it again. This authenticates the
`curl` commands throughout this guide.

> 📸 **Screenshot placeholder:** The Add API token screen in Admin Center.

### 1b. Create a ZIS OAuth client

ZIS needs its own OAuth client to issue the short-lived tokens it uses to
manage its own connections and flows — a separate credential from the API
token above.

Admin Center &gt; Apps and integrations &gt; APIs &gt; OAuth clients &gt; Add
OAuth client. Fill in:

* **Client name:** `tsanet_zis_client`
* **Company:** TSANet
* **Redirect URLs:** `https://yoursubdomain.zendesk.com` (placeholder — not used)

Click **Save** and copy the **Client ID** shown.

### 1c. Create the ZIS integration container

A named bucket inside Zendesk's ZIS platform that all TSANet resources —
connections, the flow bundle, webhooks — will live in.

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/registry/tsanet_connect" \
  -u "YOUR_EMAIL/token:YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"description": "TSANet Connect integration"}'
```

A `200 OK` response confirms the container was created.

{% hint style="info" %}
A `409 Conflict` means the integration already exists. That is fine, continue.
{% endhint %}

Finally, request a ZIS OAuth token — you'll use it to authenticate every ZIS
management call in Steps 2 and 3:

```bash
ZIS_TOKEN=$(curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/v2/oauth/tokens" \
  -u "YOUR_EMAIL/token:YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"token":{"client_id":"YOUR_ZIS_CLIENT_ID","scopes":["read","write"]}}' \
  | jq -r '.token.full_token')
```

## Step 2 &mdash; Connect ZIS to TSANet

This registers the connection that lets ZIS flows call the TSANet API on their
own — the method is **OAuth client credentials (Microsoft Entra)**: ZIS stores
the long-lived client credential TSANet issued you and mints/renews its own
short-lived tokens. Nothing scheduled, no refresh job to maintain
([tsanetgit/Zendesk\_App#1](https://github.com/tsanetgit/Zendesk_App/issues/1)).

### 2a. Register the OAuth client

The TSANet-issued `TENANT_ID`, `AUDIENCE`, and client credentials go here (the
API scope goes in `default_scopes`):

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/connections/oauth/clients/tsanet_connect" \
  -H "Authorization: Bearer $ZIS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "tsanet_entra",
    "grant_type": "client_credentials",
    "client_id": "YOUR_ENTRA_CLIENT_ID",
    "client_secret": "YOUR_ENTRA_CLIENT_SECRET",
    "token_url": "https://login.microsoftonline.com/TENANT_ID/oauth2/v2.0/token",
    "default_scopes": "api://AUDIENCE/.default"
  }'
```

{% hint style="warning" %}
Paste the client secret **verbatim**. Entra secrets can begin with punctuation,
and trimming it breaks auth with `AADSTS7000215`.
{% endhint %}

### 2b. Create the connection

No browser or admin-consent step is needed for client credentials:

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/connections/oauth/start/tsanet_connect" \
  -H "Authorization: Bearer $ZIS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"oauth_client_name": "tsanet_entra", "name": "tsanet_oauth"}'
```

The response contains a `redirect_url` with a `verification_code`. **GET that
URL** (with the same `$ZIS_TOKEN` bearer) to complete creation — this step is
required even for client credentials.

### 2c. Verify

The connection should hold a live `access_token` and a `token_expiry` about an
hour out:

```bash
curl -s -H "Authorization: Bearer $ZIS_TOKEN" \
  "https://{your-subdomain}.zendesk.com/api/services/zis/connections/tsanet_connect?name=tsanet_oauth"
```

ZIS renews the token automatically when it expires. Continue to Step 3 to
deploy the flow bundle that puts this connection to work.

> Gotcha: to change the stored credential later, the endpoint is **`PATCH`**
> `/api/services/zis/connections/oauth/clients/tsanet_connect/{uuid}` —
> `PUT` returns 405.

## Step 3 &mdash; Deploy the ZIS Flow Bundle

The connection from Step 2 does nothing by itself. The flows that actually
create and update Zendesk tickets live in the **ZIS flow bundle**, a single
JSON file maintained in the TSANet Connect source repository.

* Download the bundle:
  [`zis/tsanet_connect_bundle.json`](https://github.com/tsanetgit/Zendesk_App/blob/main/zis/tsanet_connect_bundle.json)
* Full technical reference, every gotcha, and the comment-forwarding add-on
  below: [`zis/README.md`](https://github.com/tsanetgit/Zendesk_App/blob/main/zis/README.md)

### 3a. Create the `zendesk` connection

The bundle's Zendesk-side actions (creating and updating tickets) don't
authenticate themselves — they need a basic-auth connection named `zendesk`:

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/integrations/tsanet_connect/connections/basic_auth" \
  -H "Authorization: Bearer $ZIS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"zendesk","username":"YOUR_EMAIL/token","password":"YOUR_API_TOKEN","allowed_domain":"{your-subdomain}.zendesk.com"}'
```

### 3b. Edit the bundle before uploading

Open the downloaded JSON file and substitute:

| What | Where | Note |
| --- | --- | --- |
| Custom field IDs | ticket-create/search/update actions | The **TSANet Token / Status / Partner / Respond By** field IDs from "Before You Start" |
| API host | every TSANet API action | Ships pointed at Production (`connect2.tsanet.org`) — use `connect2.tsanet.net` for Beta. Substitute **all** occurrences, not just the first, or some calls hit the wrong environment |
| `engineerEmail` | the Accept action | Your TSANet API user's email (from "Before You Start"). It must be on your member-registered domain — TSANet's Accept endpoint rejects any other domain |
| OAuth connection name | every TSANet API action | Ships as `tsanet_oauth`. Only change this if you named the Step 2 connection something else |

### 3c. Upload the bundle

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/registry/tsanet_connect/bundles" \
  -u "YOUR_EMAIL/token:YOUR_API_TOKEN" -H "Content-Type: application/json" \
  -d @tsanet_connect_bundle.json
```

{% hint style="info" %}
This call uses your Zendesk API token (Basic auth), not the ZIS bearer token.
{% endhint %}

### 3d. Create the inbound webhook

This is the address TSANet uses to reach you whenever something happens on one
of your collaborations:

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/inbound_webhooks/generic/tsanet_connect" \
  -H "Authorization: Bearer $ZIS_TOKEN" -H "Content-Type: application/json" \
  -d '{"source_system":"tsanet","event_type":"collaboration_event"}'
```

{% hint style="warning" %}
The response returns an ingest URL and Basic-auth credentials, shown only
once. Save all three values immediately, then send them to TSANet so inbound
events can be delivered (`callbackUrl` = the ingest URL, `callbackAuth` type
`BASIC`).
{% endhint %}

### 3e. Install the job spec

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/registry/job_specs/install?job_spec_name=zis:tsanet_connect:job_spec:jobspec_handle_ping" \
  -H "Authorization: Bearer $ZIS_TOKEN"
```

{% hint style="warning" %}
**Reinstall the job spec after every future bundle upload.** Re-uploading the
bundle orphans the existing install and the flow silently stops firing.
{% endhint %}

### What happens on each inbound event

One flow handles every TSANet event the same way: TSANet pings the webhook
with an event type and case token, ZIS pulls the full case from the TSANet
API, then either creates a new ticket (first event on that case) or updates
the existing one — syncing the TSANet Status, Partner, and Respond By fields
and adding a comment. The ticket is found by searching for the token, so no
per-event routing logic is needed.

{% hint style="info" %}
The INFORMATION status is the one most likely to be missed, since it needs a
reply before the SLA clock resumes. Build a Zendesk trigger that emails the
assigned agent as soon as **TSANet Status** changes to **Information** — the
flow above keeps that field current on every sync. Build it in Admin Center
&gt; Objects and rules &gt; Business rules &gt; Triggers.
{% endhint %}

> 📸 **Screenshot placeholder:** The Zendesk trigger that emails the assigned
> agent when the TSANet Status field changes to Information.

### Forward public replies to the partner (optional, recommended)

By default, partner notes flow **into** Zendesk. You can also send agent replies
back **out** to the partner automatically: when an agent posts a **public
reply** on a TSANet ticket, it is forwarded to the partner as a note. Internal
comments are never forwarded.

First, deploy the second piece of the bundle (in addition to 3d/3e above):

```bash
# Create the comment-forwarding inbound webhook (its own ingest URL + Basic creds)
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/inbound_webhooks/generic/tsanet_connect" \
  -H "Authorization: Bearer $ZIS_TOKEN" -H "Content-Type: application/json" \
  -d '{"source_system":"zendesk","event_type":"public_comment"}'

# Install its job spec (reinstall after every future bundle upload, like Step 3e)
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/registry/job_specs/install?job_spec_name=zis:tsanet_connect:job_spec:jobspec_forward_comment" \
  -H "Authorization: Bearer $ZIS_TOKEN"
```

Then, in Zendesk Admin Center, create a **webhook** (Basic auth = the
credentials from the call above, endpoint = its ingest URL, JSON) and a
**trigger** — conditions: comment is public AND current tags include
`tsanet_inbound` or `tsanet_outbound`; action: notify that webhook with the
ticket's token, latest public comment, ticket ID, and `{{current_user.role}}`.
Full request-body detail is in
[`zis/README.md`](https://github.com/tsanetgit/Zendesk_App/blob/main/zis/README.md).
Only replies authored by an **agent or admin** are forwarded — an end
customer's public reply is not.

{% hint style="info" %}
With this in place, the ZAF app's **Add Note → Public** simply posts a public
reply and lets the trigger deliver it, so the partner receives each note once.
{% endhint %}

## Optional: Native Field Actions and One-Click Macros

Teams that do not install the sidebar app, or that want a no-app fallback, can
drive the inbound lifecycle from two native Zendesk fields. An agent sets a
dropdown and the integration performs the action against TSANet. The flow that
does this (`flow_field_action`) is already included in the same bundle you
deployed in Step 3 — setting it up has three parts.

### Create the two fields

In Admin Center &gt; Objects and rules &gt; Tickets &gt; Fields, create:

* **TSANet Action**, a dropdown with these options (the tag values must match
  exactly):
  * Accept (tag `tsanet_action_accept`)
  * Reject (tag `tsanet_action_reject`)
  * Request Info (tag `tsanet_action_request_info`)
  * Add Note (tag `tsanet_action_add_note`)
* **TSANet Action Text**, a text field for the supporting text (a reject reason,
  an information question, or a note body).

Note the Field ID of each, shown in the URL when you open the field.

### Wire the two field IDs into the bundle

Substitute both field IDs into the bundle JSON alongside the Step 3b
substitutions, then re-upload it (Step 3c) and reinstall **all three** job
specs — `jobspec_handle_ping`, `jobspec_forward_comment` (if deployed), and:

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/registry/job_specs/install?job_spec_name=zis:tsanet_connect:job_spec:jobspec_field_action" \
  -H "Authorization: Bearer $ZIS_TOKEN"
```

If you're setting this up for the first time together with Step 3, just make
these substitutions before your first upload and skip the re-upload.

### Create the optional macros

Macros make the field actions one-click: instead of opening the dropdown, an
agent applies a macro that sets the TSANet Action field. Create one macro per
action in Admin Center &gt; Workspaces &gt; Macros (or via the API). Each macro
sets the TSANet Action field to the matching value:

| Macro | Sets TSANet Action to |
| --- | --- |
| TSANet: Accept | `tsanet_action_accept` |
| TSANet: Request Info | `tsanet_action_request_info` |
| TSANet: Reject | `tsanet_action_reject` |
| TSANet: Send partner-only note | `tsanet_action_add_note` |

The agent still types any needed text into TSANet Action Text before applying
the macro. The **TSANet: Send partner-only note** macro is how an agent sends a
note the partner sees but the end customer does not, without the sidebar app.

Create a macro via the API by substituting your TSANet Action field ID for
`FIELD_ID`:

```bash
curl -X POST "https://{your-subdomain}.zendesk.com/api/v2/macros.json" \
  -u "you@example.com/token:YOUR_API_TOKEN" -H "Content-Type: application/json" \
  -d '{"macro":{"title":"TSANet: Send partner-only note","actions":[{"field":"custom_fields_FIELD_ID","value":"tsanet_action_add_note"}]}}'
```

{% hint style="info" %}
Macros are per-instance Zendesk configuration and cannot be bundled with the
integration, so each Zendesk account creates its own. The field actions work
without macros (set the dropdown by hand); the macros are purely a one-click
convenience.
{% endhint %}

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
   confirm the TSANet Status field updates and the agent is emailed.
6. **Does the full agent flow work?** Have a real agent open a ticket, click
   New Collaboration, search for a partner, and submit.
7. **Does error handling work?** Submit with an incorrect partner ID and
   confirm the agent sees a clear, helpful error.

</details>

When all seven tests pass, contact TSANet to request Production credentials,
then update the app settings to use them.

## Authentication Summary

There are five authentication contexts in this integration.

| Context | Method | Where it is stored |
| --- | --- | --- |
| ZIS management calls (Steps 1c to 3e) | ZIS OAuth bearer (`$ZIS_TOKEN`) | Used per-session, not stored in the app |
| ZIS to TSANet API (runtime) | OAuth client credentials (Microsoft Entra) | ZIS connection — minted and renewed automatically |
| ZIS to Zendesk API (runtime) | Basic auth (`zendesk` connection) | ZIS connection, created in Step 3a |
| ZAF app to Zendesk (runtime) | Inherited agent session via the ZAF SDK | Automatic, no credential needed |
| ZAF app to TSANet API (runtime) | JWT Bearer token from login | Zendesk app settings (encrypted) |

## Need Help

For credentials, environment access, or installation questions, contact
membership@tsanet.org.
