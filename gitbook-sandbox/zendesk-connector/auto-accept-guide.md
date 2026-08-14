---
description: >-
  Configure the TSANet Connect Zendesk integration to accept inbound
  collaboration requests automatically the moment they arrive, using one
  Zendesk trigger and the integration's native TSANet Action field.
---

# Auto-Accepting Inbound Cases

Some teams want every inbound TSANet collaboration request accepted the moment
it lands in Zendesk, with no agent action required. This page shows an
administrator how to set that up with one Zendesk trigger. It works with the
[current app release](https://github.com/tsanetgit/Zendesk_App/releases/latest)
and requires no app changes, no custom code, and nothing from TSANet's side.

{% hint style="info" %}
**Roadmap note.** A future release will add a built-in auto-accept option: a
checkbox in the app's settings that makes the trigger below unnecessary and the
acceptance text customizable
([tracked here](https://github.com/tsanetgit/Zendesk_App/issues/219)). It ships
off by default. It is an enhancement, not a prerequisite; there is no reason to
wait for it.
{% endhint %}

## How It Works

The complete accept pipeline already ships with the integration: it listens for
changes to the **TSANet Action** ticket field and calls the TSANet Connect
approval API, using the TSANet credentials it already holds, whenever the field
is set to **Accept** (see
[Working a Case Without the Sidebar App](user-guide#working-a-case-without-the-sidebar-app-native-fields)).
The only missing piece is *who sets that field when an inbound ticket is
created*. Normally that is an agent. The trigger below makes it automatic.

Everything runs server-side. No agent needs to be logged in and no browser
session is involved. This configuration is verified end to end in TSANet's test
environment: the acceptance fires about 2 seconds after the ticket is created,
and the partner sees the case as Accepted roughly 15 seconds after submitting
it.

## Prerequisites

You likely have all of this already if agents can use the TSANet Action field
today. Confirm before creating the trigger:

1. The TSANet Connect app is installed and the ZIS bundle is deployed (the
   app's deploy screen in the left navigation shows the current state).
2. The **TSANet Action** dropdown field exists with its standard option tags
   (`tsanet_action_accept`, `tsanet_action_reject`, `tsanet_action_request_info`,
   `tsanet_action_add_note`), along with the **TSANet Action Text** field. See
   the [Installation Guide](installation-guide).
3. Field IDs have been detected and applied from the app's **Detect field IDs**
   screen.
4. The bundle was redeployed **after** those fields existed. This matters: a
   bundle deployed before the fields were created skips the field-action
   machinery entirely. If in doubt, redeploy from the app's deploy screen; it
   is quick and settings are preserved.

## Setup: One Trigger

In Admin Center go to **Objects and rules > Business rules > Triggers >
Create trigger**:

* **Name:** `TSANet: Auto-accept inbound collaboration requests`
* **Conditions (meet ALL):**
  * `Ticket` | `Is` | `Created`
  * `Tags` | `Contains at least one of the following` | `tsanet_inbound`
* **Actions:**
  * `TSANet Action` | `Accept`

Save. That is the entire setup.

## Accept Only Some Cases

Trigger conditions give you scoping for free. For example, to auto-accept only
cases from a specific partner, add a condition on the **TSANet Partner** field:

* `TSANet Partner` | `Contains the following string` | `<partner company name>`

Cases that do not match simply stay open for normal agent triage, exactly as
before.

## Verify It

Ask TSANet (or a partner contact) to send a test submission, or coordinate a
live test case. Within about 15 seconds of submission you should see:

1. A new ticket tagged `tsanet_inbound` for the case.
2. An internal comment on it: "TSANet: case accepted (via Action field)."
3. **TSANet Status** set to **Accepted**.
4. On the partner's side: the case shows Accepted, with your Zendesk ticket ID
   as your case number.

If nothing happens, re-check prerequisite 4 above (bundle redeployed after the
fields existed), then confirm the trigger fired by opening the ticket's events
(Events view under the ticket conversation). The
[Troubleshooting Guide](troubleshooting-guide) covers the rest of the pipeline.

## What the Partner Sees

* Case status Accepted, with your Zendesk ticket ID as the receiving case
  number.
* Accepting engineer shown as "Zendesk Agent" with your TSANet API user's email
  address.
* Next steps text: "Accepted via Zendesk."

## Things to Be Aware Of

{% hint style="warning" %}
**Auto-accept commits you instantly.** The TSANet SLA clock is satisfied at
acceptance, but acceptance also tells the partner an engineer is engaged. The
triage options (Reject, Request More Info) are skipped for auto-accepted cases,
so use the conditional form above if some cases should still be triaged by a
human first.
{% endhint %}

* **The acceptance text is currently fixed** ("Accepted via Zendesk.").
  Customizable text arrives with the built-in option on the roadmap (see the
  note at the top of this page).
* **Turning it off:** deactivate or delete the trigger in Admin Center. Cases
  received while it was active remain accepted.
