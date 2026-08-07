---
description: >-
  Step-by-step setup of the TSANet Connect integration for Zendesk: ZIS
  registration, the OAuth connection to TSANet, the ZAF sidebar app, the ZIS
  flow bundle, and pre-launch testing.
---

# Installation Guide

This guide walks a Zendesk administrator through installing the TSANet Connect
integration from start to finish. Plan for roughly one focused day of work.

There is **one download**. The ZIS flow bundle ships inside the sidebar app's
ZIP, and the app is what deploys it, so you install the app before you deploy
the bundle. There is no separate ZIS package to fetch and no Zendesk API token
to create anywhere in this process.

{% hint style="info" %}
Contact membership@tsanet.org to obtain credentials for the Beta environment
before you begin.
{% endhint %}

## Before You Start

### From TSANet

* **A dedicated API user account.** This account belongs to the integration, not
  to any individual person. Email membership@tsanet.org with the subject
  "API Credentials Request: Zendesk Integration" and ask for Beta credentials.
  This is what the sidebar app uses in Step 3.
* **A TSANet-issued Microsoft Entra client.** A separate credential from the API
  user account above: a client ID and secret, plus service principal
  onboarding. Contact TSANet with your ZIS service principal's object ID. This
  is what ZIS uses in Step 2 to call the TSANet API on its own.
* **Your partner's TSANet ID.** To send a collaboration request to a specific
  partner, you need their TSANet company ID and department ID.

The two credentials do two different jobs and are not interchangeable. The
Entra client cannot be used by the sidebar app, and the API user is not used by
ZIS. You need both.

### From Zendesk

* **Administrator access.** You must be a Zendesk administrator, not just an
  agent.
* **A paid Zendesk plan, Suite Professional or higher.** Trial accounts cannot
  be used. Zendesk Integration Services (ZIS) is required, and it is not
  available on lower tiers.
* **Four custom ticket fields**, plus one optional fifth. These store TSANet
  data on each ticket. Create them in Admin Center &gt; Objects and rules &gt;
  Tickets &gt; Fields.

| Field name | Type | What it stores |
| --- | --- | --- |
| TSANet Token | Text | The unique ID linking this ticket to a TSANet case |
| TSANet Status | Dropdown | Current status: OPEN, ACCEPTED, INFORMATION, REJECTED, CLOSED |
| TSANet Partner | Text | The name of the partner company you are collaborating with |
| TSANet Respond By | Date | The deadline by which you must respond (calendar date) |
| TSANet Tokens (Multi) | Text | *Optional.* A list of IDs for tickets carrying more than one case |

{% hint style="success" %}
**You do not need to write down the field IDs.** Create the fields and move on.
In Step 3 the app reads them out of your instance and writes them into its own
settings, and in Step 4 it substitutes them into the flow bundle for you. The
ID settings are deliberately blank when you install.
{% endhint %}

> 📸 **Screenshot placeholder:** Creating a custom ticket field in Admin Center
> &gt; Objects and rules &gt; Tickets &gt; Fields.

## The Five Steps at a Glance

1. **Register Zendesk with ZIS.** Create the integration's OAuth client and the
   integration container ZIS needs to manage itself. ZIS creates its own OAuth
   client for you as part of that.
2. **Connect ZIS to TSANet.** Register the Entra OAuth client-credentials
   connection that lets ZIS call the TSANet API without handling
   authentication itself.
3. **Install the ZAF sidebar app.** The panel agents use on every ticket, and
   the tool that deploys the flow bundle in Step 4.
4. **Deploy the ZIS flow bundle.** This is what actually creates and updates
   Zendesk tickets. Steps 1 and 2 only set up the plumbing it runs on.
5. **Test everything before going live.** Run the full scenario end to end.

## Step 1: Register Zendesk with ZIS

Two pieces of one-time Zendesk-side setup. You create one OAuth client; ZIS
creates the second one for you.

### 1a. Create the integration's OAuth client + setup token

Everything in this guide authenticates with a short-lived **setup token**
minted from a confidential OAuth client. This same client later backs the
`zendesk` connection in Step 4a, so you create it exactly once. No Zendesk API
token is used anywhere. Zendesk is retiring API tokens (new accounts cannot
create them after July 28, 2026; all tokens stop working April 30, 2027), and
the integration neither stores nor requires one.

Sign in to Admin Center **as the dedicated service user** the integration
should act as (tokens from this client act as that user), then go to
**Apps and integrations &gt; APIs &gt; OAuth clients &gt; Add OAuth client**
and fill in:

* **Client name:** `TSANet Connect Integration`
* **Identifier:** `tsanet_zendesk`
* **Client kind:** **Confidential.** Required, because the client_credentials
  grant rejects public clients
* **Redirect URLs:** `https://{your-subdomain}.zendesk.com` (a placeholder, not used)

Click **Save**, then copy the **Secret** field that appears. The full secret
is shown **only once**. If you lose it, regenerating displays it truncated, so
delete and recreate the client instead.

Now mint a setup token (about a 30-minute lifetime, so re-run this if setup
takes longer):

```bash
SETUP_TOKEN=$(curl -s -X POST "https://{your-subdomain}.zendesk.com/oauth/tokens" \
  -H "Content-Type: application/json" \
  -d '{"grant_type":"client_credentials","client_id":"tsanet_zendesk","client_secret":"YOUR_CLIENT_SECRET","scope":"read write"}' \
  | jq -r '.access_token')
```

> 📸 **Screenshot placeholder:** The Add OAuth client screen in Admin Center,
> with Client kind set to Confidential.

### 1b. Create the ZIS integration container

A named bucket inside Zendesk's ZIS platform that all TSANet resources will
live in: connections, the flow bundle, and webhooks.

```bash
curl -s -w "\nHTTP %{http_code}\n" -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/registry/tsanet_connect" \
  -H "Authorization: Bearer $SETUP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"description": "TSANet Connect integration"}'
```

An `HTTP 200` response confirms the container was created. The integration name
is case-sensitive.

{% hint style="info" %}
An `HTTP 409` means the integration already exists. That is fine, continue.
{% endhint %}

{% hint style="danger" %}
**If you get `400 the integration: tsanet_connect is not available for upsert by this account`,**
that is expected and is not a fault on your side. Zendesk requires ZIS integration
names to be **globally unique across every Zendesk account**, so a name registered
by anyone is unavailable to everyone else, and `tsanet_connect` is already taken.

Register your own instead, for example `yourcompany_tsanet_connect`. Lowercase
letters, digits, underscore and hyphen, 1 to 64 characters. Then use **your** name
everywhere `tsanet_connect` appears in this guide, including the connection calls in
Steps 2 and 4a and the webhook call in Step 4d, and set **TSANet integration name**
in the app's settings in Step 3c so the flow bundle is built for your container.

Requires app **v1.0.61 or later**. Two things to know if you take this path:
connections do not move between integrations, so `tsanet_oauth` and `zendesk` both
have to be created under your name; and the credential-rotation scripts take
`--integration <your-name>`.
{% endhint %}

{% hint style="warning" %}
**Do not continue until you see 200 or 409.** This is the one step whose failure
surfaces later and in a form that points somewhere else: with no container,
Step 2b returns `401 Authorization failed due to integration mismatch`, which
reads like a bad Entra credential rather than a missing container.

The usual cause is permissions. ZIS registry endpoints are admin-only, and a
`client_credentials` token acts as the user its OAuth client was created under.
A non-admin therefore mints tokens successfully in Step 1a and is refused only
here. If that happens, create the Step 1a OAuth client again while signed in as
an administrator, then mint the setup token again.
{% endhint %}

Creating the container also creates an OAuth client for it, named
`zis_tsanet_connect`. The `200` response carries it as `zendesk_oauth_client`:

```json
"zendesk_oauth_client": { "id": 1234567890123, "identifier": "zis_tsanet_connect", "secret": "..." }
```

{% hint style="danger" %}
**This is the only client that can mint your ZIS token.** A ZIS token is bound to
exactly one integration, and the binding comes from the client that minted it. A
token from a client you created yourself is refused on every ZIS management
endpoint with `401 Authorization failed due to integration mismatch`, no matter
how much access that client has. Only `zis_<integration-name>` works.
{% endhint %}

Finally, request a ZIS OAuth token. You will use it to authenticate every ZIS
management call in Steps 2 and 4. Pass the numeric `id` from
`zendesk_oauth_client` above, not the identifier string, and not the client from
Step 1a:

```bash
ZIS_TOKEN=$(curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/v2/oauth/tokens" \
  -H "Authorization: Bearer $SETUP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"token":{"client_id":ZIS_CLIENT_NUMERIC_ID,"scopes":["read","write"]}}' \
  | jq -r '.token.full_token')
```

The mint needs only the numeric id, not the client's secret, so if you did not
keep the create response you can look the id up at any time. Find the entry whose
`identifier` is `zis_tsanet_connect`:

```bash
curl -s -H "Authorization: Bearer $SETUP_TOKEN" \
  "https://{your-subdomain}.zendesk.com/api/v2/oauth/clients.json" \
  | jq '.clients[] | {id, identifier}'
```

## Step 2: Connect ZIS to TSANet

This registers the connection that lets ZIS flows call the TSANet API on their
own. The method is **OAuth client credentials (Microsoft Entra)**: ZIS stores
the long-lived client credential TSANet issued you and mints and renews its own
short-lived tokens. Nothing scheduled, no refresh job to maintain
([tsanetgit/Zendesk\_App#1](https://github.com/tsanetgit/Zendesk_App/issues/1)).

### 2a. Register the OAuth client

The TSANet-issued `TENANT_ID`, `AUDIENCE`, and client credentials go here. The
API scope goes in `default_scopes`:

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
    "default_scopes": "AUDIENCE/.default"
  }'
```

{% hint style="warning" %}
**Scope format.** Use the bare Connect-app client ID GUID in `default_scopes`,
that is `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/.default`, using the `AUDIENCE`
value TSANet gives you. Do **not** prefix it with `api://`. That form fails
with `AADSTS500011: resource principal not found`, because the Connect app's
Application ID URI is not published. The same applies when testing the token
request in Postman or curl.
{% endhint %}

{% hint style="warning" %}
Paste the client secret **verbatim**. Entra secrets can begin with punctuation,
and trimming a leading character breaks auth with `AADSTS7000215`. The secret is
the opaque **Value**, not a GUID. If you received a GUID, that is the Secret ID,
so ask TSANet for the Value.
{% endhint %}

`TENANT_ID` and `AUDIENCE` are provided by TSANet at onboarding and differ
between Beta and Production, so do not reuse Beta values against Production.

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
URL** (with the same `$ZIS_TOKEN` bearer) to complete creation. This step is
required even for client credentials.

### 2c. Verify

The connection should hold a live `access_token` and a `token_expiry` about an
hour out:

```bash
curl -s -H "Authorization: Bearer $ZIS_TOKEN" \
  "https://{your-subdomain}.zendesk.com/api/services/zis/connections/tsanet_connect?name=tsanet_oauth"
```

ZIS renews the token automatically when it expires.

> Gotcha: to change the stored credential later, the endpoint is **`PATCH`**
> `/api/services/zis/connections/oauth/clients/tsanet_connect/{uuid}`.
> `PUT` returns 405.

## Step 3: Install the ZAF Sidebar App

The sidebar app is the panel agents use on every ticket. It is also what
deploys the flow bundle in Step 4, which is why it goes in before the bundle.
It is distributed as a pre-built ZIP and installed privately. There is no
Zendesk Marketplace listing.

{% hint style="warning" %}
**Install v1.0.53 or later.** That is the release where the app reads your
field IDs out of the instance instead of asking you to record them by hand.
Earlier releases either could not open the deploy screen at all
([tsanetgit/Zendesk\_App#131](https://github.com/tsanetgit/Zendesk_App/issues/131))
or blocked fresh installs
([tsanetgit/Zendesk\_App#134](https://github.com/tsanetgit/Zendesk_App/issues/134)).
{% endhint %}

* Visit the releases page and take the newest version:
  [https://github.com/tsanetgit/Zendesk\_App/releases](https://github.com/tsanetgit/Zendesk_App/releases)
* Download the `tsanet-connect-vX.Y.Z.zip` asset.
* Verify the download before installing it (see below).
* In Admin Center &gt; Apps and integrations &gt; Zendesk Support apps, choose
  to upload a private app, name it `TSANet Connect`, and select the ZIP.

### Verify the package before you upload it

Every release carries a build-provenance attestation and a SHA-256 checksum,
both published on the same release page, so you can confirm the ZIP is the one
TSANet built rather than a file altered in transit or substituted elsewhere.

{% code overflow="wrap" %}
```bash
# Provenance (requires the GitHub CLI)
gh attestation verify tsanet-connect-v<version>.zip --repo tsanetgit/Zendesk_App

# Or checksum only, run where the ZIP and checksums.txt both sit
shasum -a 256 -c checksums.txt
```
{% endcode %}

Expect a line confirming the attestation was verified against
`tsanetgit/Zendesk_App`, or `tsanet-connect-v<version>.zip: OK` from the
checksum check.

{% hint style="danger" %}
If verification fails, do not install the package. A failure means the file
does not match what TSANet published. Re-download it from the release page and
try again; if it still fails, contact membership@tsanet.org before proceeding.
A mismatch is worth reporting even if a re-download then succeeds.
{% endhint %}

> 📸 **Screenshot placeholder:** Uploading the private app ZIP in Admin Center
> &gt; Apps and integrations &gt; Zendesk Support apps.

### 3a. Enter the app settings

When prompted:

| Setting | Value |
| --- | --- |
| TSANet API username | The dedicated API user account from "Before You Start" |
| TSANet API password | The password for that account |
| TSANet environment | `BETA` for setup and testing, `PRODUCTION` when live |
| All field ID settings | **Leave blank.** Step 3b fills them in |
| Allowed action roles | *Optional.* Comma-separated Zendesk role names permitted to click the TSANet action buttons. Empty means all agents |
| TSANet integration name | Leave as `tsanet_connect`. Change it only if Step 1b forced you onto a different name, in which case enter the exact name you registered |
| Shared author user id | *Optional (v1.0.64+).* Numeric Zendesk user id of a dedicated user, for example "IBM via TSANet", that authors everything the integration writes: inbound case descriptions, receipts and mirrored partner notes. Create the user first. An end user costs no seat and a light agent also works; either way its email must be on **your own** domain, never the partner's, so notifications cannot route outside your org. Empty means those messages appear under the admin who authorised the Zendesk connection. The bundle deploy pre-flight resolves the id and shows the user's name before anything ships |

`BETA` maps to `connect2.tsanet.net` and `PRODUCTION` to `connect2.tsanet.org`.
Set it to match where your account is provisioned.

The **TSANet API password** is a Zendesk secure setting: it is stored
encrypted, never reaches the front end, and requests using it are proxied
server-side by Zendesk.

{% hint style="info" %}
**Allowed action roles is a convenience gate, not a security boundary.** It
controls which roles see and can click the TSANet action buttons. The TSANet
Connect API and its credential remain the real authorization control.
{% endhint %}

> 📸 **Screenshot placeholder:** The ZAF app settings screen with the TSANet
> credentials and environment filled in, and the field ID settings left blank.

### 3b. Detect the field IDs

Open **TSANet Connect** from the left nav bar in Zendesk Support and click
**Detect field IDs**.

It reads this instance's ticket fields, matches them by name, and shows you
exactly what it found (name, ID, and type) before anything is saved. Click
**Apply** to write them into the app's settings.

It refuses rather than guesses:

* Two fields sharing a name are reported as ambiguous, and it names both IDs.
* A name match with the wrong type is refused. **TSANet Status** must be a
  dropdown and **TSANet Respond By** must be a date.
* A missing required field is called out, with where to create it. A missing
  optional one is fine.

Until this is done, the sidebar tells any agent who opens a ticket that TSANet
Connect is not configured yet, rather than appearing to work and quietly doing
nothing.

### 3c. Confirm the sidebar renders

Open a ticket with **no** TSANet collaboration on it. The TSANet Connect panel
should appear collapsed to a slim bar reading "No active TSANet cases" with a
**+ New** button. Click **+ New** and the panel should expand and open the New
Collaboration search dialog.

If the sidebar shows an error instead of the compact bar, re-check the API
credentials and that the environment setting matches where your account is
provisioned.

{% hint style="success" %}
Updating the app later is the same process: download the new ZIP and upload it
over the existing app. Settings are preserved across updates, so there is no
need to re-enter credentials or re-run Detect.
{% endhint %}

## Step 4: Deploy the ZIS Flow Bundle

The connection from Step 2 does nothing by itself. The flows that actually
create and update Zendesk tickets live in the **ZIS flow bundle**, a single
JSON file that ships **inside the app ZIP you just installed**. You do not
download it or edit it separately.

Full technical reference, every flow, and every gotcha:
[`zis/README.md`](https://github.com/tsanetgit/Zendesk_App/blob/main/zis/README.md).

### 4a. Create the `zendesk` connection

The bundle's Zendesk-side actions (creating, searching, and updating tickets)
do not authenticate themselves. They need an **OAuth connection named
`zendesk`**. It stores no long-lived secret: ZIS mints short-lived tokens from
an OAuth client on your own instance and renews them itself.

The client is the one you already created in **Step 1a** (`tsanet_zendesk`,
confidential, owned by the dedicated service user), the same identifier and
secret. Register a ZIS OAuth client pointing at your **own** instance's token
endpoint, and create the connection from it:

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/connections/oauth/clients/tsanet_connect" \
  -H "Authorization: Bearer $ZIS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "zendesk_self",
    "grant_type": "client_credentials",
    "client_id": "tsanet_zendesk",
    "client_secret": "YOUR_CLIENT_SECRET",
    "token_url": "https://{your-subdomain}.zendesk.com/oauth/tokens",
    "default_scopes": "read tickets:write"
  }'

curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/connections/oauth/start/tsanet_connect" \
  -H "Authorization: Bearer $ZIS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"oauth_client_name": "zendesk_self", "name": "zendesk"}'
```

GET the returned `redirect_url` (with the same `$ZIS_TOKEN` bearer) to
complete creation, the same verification step as the TSANet connection in
Step 2b. The connection then holds a live token which ZIS renews
automatically.

{% hint style="info" %}
`read tickets:write` is the minimal scope: ticket search breaks under
`tickets:read` alone, and the ticket create and update actions need
`tickets:write`. **Migrating an existing install:** connection names are
unique across types, so delete the old basic-auth `zendesk` connection first,
then create this one under the same name. The bundle needs no changes.
{% endhint %}

### 4b. Check your per-instance values

You do not edit the bundle JSON. The app fills these in from its own settings
when you deploy, and refuses to upload if any of them is still a placeholder.
Use this table to confirm the app's settings are right before you deploy, and
to know what to look at if a flow later misbehaves.

| What | Where | Note |
| --- | --- | --- |
| Custom field IDs | ticket create / search / update actions | Filled in by **Detect field IDs** in Step 3b |
| API host | every TSANet API action | Ships pointed at Production (`connect2.tsanet.org`); Beta is `connect2.tsanet.net`. It appears in **all** the TSANet API actions, not just the first, and they move together |
| `engineerEmail` | the Accept action | Your TSANet API user's email (from "Before You Start"). It must be on your member-registered domain, because TSANet's Accept endpoint rejects any other domain |
| OAuth connection name | every TSANet API action | Ships as `tsanet_oauth`. This only differs if you named the Step 2b connection something else |

### 4c. Deploy the bundle

1. In Zendesk Support, open **TSANet Connect** from the left nav bar.
2. Read the **Current state** card at the top of the screen.
3. Check the **Pre-flight** results. All three must pass before the button enables.
4. Click **Deploy bundle**.

The app substitutes your per-instance values, uploads the bundle, **installs
every job spec in it**, then reads the registry back so you can see what is
actually installed. Success is judged by that read-back, not by the upload
response.

{% hint style="info" %}
**The Current state card tells you whether a deploy is needed at all.**
Requires app **v1.0.63 or later**.

| Row | What it tells you |
| --- | --- |
| Bundle | Whether the bundle registered on this instance matches what this app would deploy |
| App version | Which version of the app is installed here |
| Latest release | Whether a newer version has been published |

Only the Bundle row bears on the decision, and it compares the bundle's
**actual contents**, never version numbers. That is deliberate: between
v1.0.54 and v1.0.60 there were six releases in which the ZIS bundle did not
change once, so a version comparison would have asked you to redeploy six
times for no functional change. The two version rows are reference only and
can never disable the Deploy button.

The Bundle row gives three different answers because the remedy differs.
*Older than the one this app ships* means a different bundle generation, so
deploy. *App settings changed since it was deployed* means the right
generation built with different values, so deploy to pick them up. *Not
deployed yet* means there is nothing registered to compare.
{% endhint %}

{% hint style="warning" %}
**"Matches what this app would deploy" is not the same as "no deploy
needed."** It establishes what is registered and nothing about whether the
job specs are installed. A deploy interrupted between the upload and the
installs leaves a matching registry on an integration that processes no
events. Installed state is what Pre-flight and the post-deploy read-back
are for.
{% endhint %}

{% hint style="success" %}
**Job specs install themselves now.** Earlier revisions of this guide had you
`POST .../job_specs/install` by hand and repeat it after every upload. The app
installs every job spec the bundle declares on each deploy and confirms the
result, so there is nothing to run manually. Ignore any older instructions
that ask you to.
{% endhint %}

{% hint style="info" %}
**Why the app and not a curl command.** The bundles endpoint rejects OAuth
outright (401 "Authorization failed due to OAuth being disabled for this API
request", verified July 2026), so neither `$SETUP_TOKEN` nor `$ZIS_TOKEN` works
on it. The only credentials it accepts are a Zendesk API token and your own
signed-in admin session. Zendesk is withdrawing API tokens: none for accounts
created on or after **28 July 2026**, none for any account after **27 October
2026**, and all existing tokens stop working on **30 April 2027**. Password
access was removed from remaining accounts starting January 2026. The app runs
on your admin session, so there is no credential for you to create, store, or
rotate for this step.
{% endhint %}

{% hint style="warning" %}
**You must be a Zendesk administrator**, and deploying replaces the installed
bundle. An upload orphans the job specs from the previous one; the app
reinstalls them immediately and verifies, but the integration is briefly
inactive in between, so do not close the tab mid-run. If a step fails, the
screen names it and offers **Retry all**.

If the read-back reports job specs that are installed but absent from the
current bundle, those are stale orphans from an older generation and they
still intercept events. Uninstall them with
`DELETE /api/services/zis/registry/job_specs/install?job_spec_name=zis:tsanet_connect:job_spec:<name>`.
{% endhint %}

### 4d. Create the inbound webhook, then subscribe TSANet to it

This is **two API calls, and you make both of them.** Nothing here is an email
to TSANet, and nobody at TSANet does anything on your behalf.

1. **Zendesk** creates the address TSANet will post to, and gives you a
   credential that guards it.
2. **TSANet** is told to post to that address, using that credential.

The second call just carries across what the first one returned.

#### Call 1: create the ingest URL, on Zendesk

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/inbound_webhooks/generic/tsanet_connect" \
  -H "Authorization: Bearer $ZIS_TOKEN" -H "Content-Type: application/json" \
  -d '{"source_system":"tsanet","event_type":"collaboration_event"}'
```

{% hint style="warning" %}
The response returns an ingest URL, Basic-auth credentials, and a `uuid`, shown
only once. Save all of them immediately. There is no list API, so a lost `uuid`
is unrecoverable and you will not be able to rotate the credential later.
{% endhint %}

#### Call 2: register the subscription, on TSANet

Authenticate with your TSANet API account, the same one the sidebar app uses.
Get a token from `POST /v1/login`, then:

```bash
curl -s -X POST "https://connect2.tsanet.net/v1/webhooks" \
  -H "Authorization: Bearer YOUR_TSANET_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "callbackUrl": "THE_INGEST_URL_FROM_CALL_1",
    "callbackAuth": {
      "type": "BASIC",
      "username": "THE_BASIC_USERNAME_FROM_CALL_1",
      "password": "THE_BASIC_PASSWORD_FROM_CALL_1"
    }
  }'
```

`connect2.tsanet.net` is Beta; Production is `connect2.tsanet.org`. Leave
`eventTypes` out: omitted means both `collaboration-request.created` and
`note.created`, which is what the flow bundle expects.

Save the `id` from the response, which is what you need to inspect or remove the
subscription later. The response also carries `secret`, the HMAC signing key,
returned only on creation.

{% hint style="danger" %}
**Use `/v1/webhooks`, not `/v2/webhooks`.** Both endpoints exist and both will
accept your subscription, which makes this easy to get wrong and hard to notice.
`/v2/` delivers CloudEvents, whose event type strings are prefixed
(`org.tsanet.connect.collaboration-request.created`). The flow bundle matches the
unprefixed V1 name, so on a `/v2/` subscription every delivery is accepted,
returns 200, and then falls through to a no-op: **no ticket is ever created and
nothing reports an error.**

`/v1/webhooks` is deprecated with a sunset of 2027-01-01. Migrating the flow and
existing subscriptions to CloudEvents is tracked work and will ship with a future
app release. Until then, `/v1/` is the correct choice.
{% endhint %}

### What happens on each inbound event

One flow handles every TSANet event the same way: TSANet pings the webhook
with an event type and case token, ZIS pulls the full case from the TSANet
API, then either creates a new ticket (first event on that case) or updates
the existing one, syncing the TSANet Status, Partner, and Respond By fields
and adding a comment. The ticket is found by searching for the token, so no
per-event routing logic is needed. Only the creation event may create a
ticket, which is what stops a later note racing ahead and producing a
duplicate.

{% hint style="info" %}
The INFORMATION status is the one most likely to be missed, since it needs a
reply before the SLA clock resumes. Build a Zendesk trigger that emails the
assigned agent as soon as **TSANet Status** changes to **Information**. The
flow above keeps that field current on every sync. Build it in Admin Center
&gt; Objects and rules &gt; Business rules &gt; Triggers.
{% endhint %}

> 📸 **Screenshot placeholder:** The Zendesk trigger that emails the assigned
> agent when the TSANet Status field changes to Information.

### 4e. Forward public replies to the partner (optional, recommended)

By default, partner notes flow **into** Zendesk. You can also send agent replies
back **out** to the partner automatically: when an agent posts a **public
reply** on a TSANet ticket, it is forwarded to the partner as a note. Internal
comments are never forwarded, and only replies authored by an **agent or
admin** are sent, never an end customer's.

The flow and its job spec are already deployed, because Step 4c installs
everything in the bundle. What is left is the per-instance wiring. First create
the second inbound webhook, which gets its own ingest URL and Basic
credentials:

```bash
curl -s -X POST \
  "https://{your-subdomain}.zendesk.com/api/services/zis/inbound_webhooks/generic/tsanet_connect" \
  -H "Authorization: Bearer $ZIS_TOKEN" -H "Content-Type: application/json" \
  -d '{"source_system":"zendesk","event_type":"public_comment"}'
```

Then, in Zendesk Admin Center, create a **webhook** (Basic auth = the
credentials from the call above, endpoint = its ingest URL, JSON) and a
**trigger**. Trigger conditions: comment is public AND current tags include
`tsanet_inbound` or `tsanet_outbound`. Trigger action: notify that webhook with
the ticket's token, latest public comment, ticket ID, and
`{{current_user.role}}`. Full request-body detail is in
[`zis/README.md`](https://github.com/tsanetgit/Zendesk_App/blob/main/zis/README.md).

{% hint style="info" %}
With this in place, the sidebar app's **Add Note &gt; Public** simply posts a
public reply and lets the trigger deliver it, so the partner receives each note
once rather than twice.
{% endhint %}

## Optional: Native Field Actions and One-Click Macros

Teams that do not install the sidebar app for every agent, or that want a
no-app fallback, can drive the inbound lifecycle from two native Zendesk
fields. An agent sets a dropdown and the integration performs the action
against TSANet.

The flow that does this (`flow_field_action`) is included in the same bundle
you deployed in Step 4, but it is **off by default**, so that a first-time
install is not blocked on fields you have not created yet. Turning it on has
three parts.

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

You do not need to note the Field IDs. Once both fields exist, run **Detect
field IDs** again and click **Apply** — these two are optional entries in the
same detection that filled in the core fields, so it picks them up exactly as it
did those.

### Turn the field actions on

Run **Detect field IDs** and **Apply** to fill in the two field IDs, then add
your **TSANet engineer email** by hand. That one is not a ticket field, so
detection cannot find it. It is the address TSANet records as the engineer on an
Accept, and it must be on your member-registered domain.

Then run **Deploy bundle** again (Step 4c). The app re-uploads with the
field-action resources included and reinstalls every job spec.

Both field IDs must be set together. One without the other is reported as an
error rather than guessed at.

If you are setting this up for the first time together with Step 4, just enter
these two IDs in the app settings before your first deploy and there is nothing
to redo.

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
  -H "Authorization: Bearer $SETUP_TOKEN" -H "Content-Type: application/json" \
  -d '{"macro":{"title":"TSANet: Send partner-only note","actions":[{"field":"custom_fields_FIELD_ID","value":"tsanet_action_add_note"}]}}'
```

{% hint style="info" %}
Macros are per-instance Zendesk configuration and cannot be bundled with the
integration, so each Zendesk account creates its own. The field actions work
without macros (set the dropdown by hand); the macros are purely a one-click
convenience.
{% endhint %}

## Step 5: Test Everything Before Going Live

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
   webhook URL and confirm the correct ticket is created or updated.
5. **Does the INFORMATION alert work?** Send a simulated INFORMATION event and
   confirm the TSANet Status field updates and the agent is emailed.
6. **Does the full agent flow work?** Have a real agent open a ticket, click
   **+ New**, search for a partner, and submit.
7. **Does error handling work?** Submit with an incorrect partner ID and
   confirm the agent sees a clear, helpful error.

</details>

When all seven tests pass, contact TSANet to request Production credentials,
then update the app's environment, username, and password, and run **Deploy
bundle** again so the API host in the flows moves with them.

## Authentication Summary

There are seven authentication contexts in this integration. None of them uses
a Zendesk API token.

| Context | Method | Where it is stored |
| --- | --- | --- |
| Setup commands (Step 1b, and the macro call) | Short-lived OAuth setup token (`$SETUP_TOKEN`, client_credentials) | Minted per use from the integration's OAuth client, nothing stored |
| ZIS management calls (Steps 1c, 2, 4a, 4d) | ZIS OAuth bearer (`$ZIS_TOKEN`) | Used per session, nothing stored |
| Bundle deploy (Step 4c) | Your own signed-in admin session, via the TSANet Connect app | Nothing stored, the app inherits the session. This endpoint refuses OAuth, which is why it runs in the app rather than as a setup command |
| ZIS to TSANet API (runtime) | OAuth client credentials (Microsoft Entra) | ZIS connection, minted and renewed automatically |
| ZIS to Zendesk API (runtime) | OAuth client credentials against your own instance (`zendesk` connection, scope `read tickets:write`) | ZIS connection, created in Step 4a, short-lived tokens minted and renewed automatically |
| ZAF app to Zendesk (runtime) | Inherited agent session via the ZAF SDK | Automatic, no credential needed |
| ZAF app to TSANet API (runtime) | Login as the dedicated TSANet API user | The password is a Zendesk secure setting: encrypted, never reaches the front end, and requests using it are proxied server-side |

## Need Help

For credentials, environment access, or installation questions, contact
membership@tsanet.org.
