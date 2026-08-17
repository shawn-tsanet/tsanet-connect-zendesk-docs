---
description: >-
  Configure the TSANet Connect Zendesk integration to accept inbound
  collaboration requests automatically the moment they arrive, using either the
  built-in app setting or a Zendesk trigger for conditional acceptance.
---

# Auto-Accepting Inbound Cases

Some teams want every inbound TSANet collaboration request accepted the moment
it lands in Zendesk, with no agent action required. There are two ways to get
there, suited to different policies:

| Path | Suits | Requires |
| --- | --- | --- |
| **The built-in setting** (v1.0.69+) | Accepting **everything, unconditionally** | The [current app release](https://github.com/tsanetgit/Zendesk_App/releases/latest); nothing else |
| **A Zendesk trigger** | Accepting **conditionally** — only certain partners, priorities, or business hours | The optional field-actions feature from the [Installation Guide](installation-guide.md) |

Both run server-side: no agent needs to be logged in and no browser session is
involved. Both are verified end to end in TSANet's test environment, with the
partner seeing the case as Accepted roughly 15 seconds after submitting it.

{% hint style="warning" %}
**Auto-accept commits you instantly.** The TSANet SLA clock is satisfied at
acceptance, but acceptance also tells the partner an engineer is engaged. The
triage options (Reject, Request More Info) are skipped for auto-accepted cases.
If some cases should still be triaged by a human first, use the trigger path
and put those conditions in the trigger.
{% endhint %}

## The Built-In Setting (Unconditional)

In Admin Center open the TSANet Connect app's settings and turn on
**Auto-accept inbound requests**. Set **Auto-accept next steps** to the text the
partner should see on the acceptance — for example "Case accepted
automatically; an engineer will follow up shortly." — then **redeploy the
bundle** from the app's deploy screen. The settings are baked into the bundle
at deploy time, so a toggle without a redeploy changes nothing; the deploy
screen's Current state card flags a toggled-but-not-redeployed instance.

Acceptance then happens inside the inbound flow itself, immediately after the
ticket is created: deterministic, with no trigger ordering involved. Your
Zendesk ticket number is sent as the receiving case number.

**If an auto-accept attempt fails** (TSANet rejects the call, or a transient
error), the ticket receives an internal comment saying the case was **not**
accepted and needs a manual Accept. The case stays OPEN on the TSANet side with
its SLA clock still running, exactly as if auto-accept were off — a failed
auto-accept never shows a false "accepted" status anywhere.

**Turning it off:** untick the setting and redeploy the bundle. Cases accepted
while it was on remain accepted.

## The Trigger (Conditional)

The field-action machinery that ships with the integration listens for changes
to the **TSANet Action** ticket field and calls the TSANet approval API
whenever the field is set to **Accept** (see
[Working a Case Without the Sidebar App](user-guide.md#working-a-case-without-the-sidebar-app-native-fields)).
A trigger that sets the field at ticket creation makes acceptance automatic,
and trigger conditions carry your acceptance policy naturally.

### Prerequisites

You likely have all of this already if agents can use the TSANet Action field
today. Confirm before creating the trigger:

1. The TSANet Connect app is installed and the ZIS bundle is deployed (the
   app's deploy screen in the left navigation shows the current state).
2. The **TSANet Action** dropdown field exists with its standard option tags
   (`tsanet_action_accept`, `tsanet_action_reject`, `tsanet_action_request_info`,
   `tsanet_action_add_note`), along with the **TSANet Action Text** field. See
   the [Installation Guide](installation-guide.md).
3. Field IDs have been detected and applied from the app's **Detect field IDs**
   screen.
4. The bundle was redeployed **after** those fields existed. This matters: a
   bundle deployed before the fields were created skips the field-action
   machinery entirely. If in doubt, redeploy from the app's deploy screen; it
   is quick and settings are preserved.

### Setup: One Trigger

In Admin Center go to **Objects and rules > Business rules > Triggers >
Create trigger**:

* **Name:** `TSANet: Auto-accept inbound collaboration requests`
* **Conditions (meet ALL):**
  * `Ticket` | `Is` | `Created`
  * `Tags` | `Contains at least one of the following` | `tsanet_inbound`
* **Actions:**
  * `TSANet Action` | `Accept`

Save. That is the entire setup.

### Accept Only Some Cases

Trigger conditions give you scoping for free. For example, to auto-accept only
cases from a specific partner, add a condition on the **TSANet Partner** field:

* `TSANet Partner` | `Contains the following string` | `<partner company name>`

Cases that do not match simply stay open for normal agent triage, exactly as
before.

**Turning it off:** deactivate or delete the trigger in Admin Center.

## Verify It

Ask TSANet (or a partner contact) to send a test submission, or coordinate a
live test case. Within about 15 seconds of submission you should see:

1. A new ticket tagged `tsanet_inbound` for the case.
2. An internal confirmation comment on it (the setting path says "case accepted
   automatically"; the trigger path says "via Action field").
3. **TSANet Status** set to **Accepted**.
4. On the partner's side: the case shows Accepted, with your Zendesk ticket ID
   as your case number.

If nothing happens on the trigger path, re-check prerequisite 4 above (bundle
redeployed after the fields existed), then confirm the trigger fired by opening
the ticket's events (Events view under the ticket conversation). On the setting
path, confirm the bundle was redeployed after the setting was turned on. The
[Troubleshooting Guide](troubleshooting-guide.md) covers the rest of the
pipeline.

## What the Partner Sees

* Case status Accepted, with your Zendesk ticket ID as the receiving case
  number.
* Accepting engineer shown as "Zendesk Agent" with the engineer email from your
  app settings (or your TSANet API user's address when that setting is blank).
* Next steps text: whatever **Auto-accept next steps** is set to, "Accepted via
  Zendesk." by default. Both paths use the same text setting (v1.0.69+).
