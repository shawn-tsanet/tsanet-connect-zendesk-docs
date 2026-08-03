---
description: >-
  How support engineers use the TSANet Connect sidebar in Zendesk to
  collaborate with partner vendors: creating requests, handling inbound cases,
  exchanging notes, and tracking status.
---

# User Guide

This guide is for the people who use the TSANet Connect integration day to day,
once an administrator has installed it. For setup, see the
[Installation Guide](installation-guide).

## User Types

**Technical Support Engineer.** Staff responsible for resolving customer cases.
They initiate outbound collaboration requests with partner vendors and
collaborate on inbound requests assigned to them.

**Customer Service / Management Team.** Handles the intake of inbound requests
and monitors the progress of all collaborations.

{% hint style="info" %}
The screens below describe the standard TSANet Connect sidebar. If your
organization customizes the Zendesk workspace, this guide is the baseline for
your own internal documentation.
{% endhint %}

## Collaboration Process Overview

TSANet Connect sits between two vendors' support systems. Neither vendor
connects directly to the other. Both connect to TSANet, which routes the
collaboration.

There are two directions of work:

* **Outbound.** Your engineer needs help from a partner vendor and sends them a
  collaboration request.
* **Inbound.** A partner vendor needs help from your team and sends you a
  request. It arrives as a Zendesk ticket.

## The TSANet Sidebar Panel

The TSANet panel appears on the right side of every Zendesk ticket.

* On tickets with no TSANet collaboration, the panel collapses to a compact bar
  with a single **New Collaboration** button.
* On tickets with an active collaboration, the panel expands to show the full
  case detail.

When expanded, the panel shows:

* Every active TSANet collaboration on the ticket
* The status of each one: OPEN, ACCEPTED, INFORMATION, REJECTED, or CLOSED
* The partner engineer's name and contact details
* A live countdown to the SLA response deadline
* Action buttons: New Collaboration, Accept, Reject, Request More Info,
  Respond Now, and Add Note (with an Internal / Partner only / Public choice)

The panel refreshes automatically about once a minute while any agent has Zendesk
open, so the case detail stays current without a manual refresh.

{% hint style="info" %}
**The panel sizes itself to what is on screen.** Requires app **v1.0.63 or
later**. It re-measures when the New Collaboration dialog opens and closes,
when a partner's request form renders, when an active case's notes finish
loading, and when a case closes.

Before v1.0.63 the panel asked Zendesk for a fixed height no matter what was
in it, so a partner form with six or more fields was cut off at the bottom,
partner search results were clipped, and note threads were truncated mid-note.
If you are seeing any of that, your instance is on an older version and an
administrator needs to update the app.

A very busy ticket, carrying several collaboration cases each with a long note
thread, can still scroll. That is intended: the panel is bounded so it cannot
push your other sidebar apps out of reach.
{% endhint %}

> 📸 **Screenshot placeholder:** The expanded TSANet sidebar panel on a Zendesk
> ticket, showing case detail, the SLA countdown, and the action buttons.

## Creating a New Outbound Request

When your engineer needs help from a partner vendor:

1. Open the Zendesk ticket for the customer case.
2. In the TSANet panel, click **New Collaboration**.
3. Search for the partner by name and select them.
4. Complete the request form with the problem detail the partner needs.
5. Submit. The collaboration token is saved to the ticket automatically.

> 📸 **Screenshot placeholder:** The New Collaboration request form with the
> partner search and request detail fields.

The case now appears in the panel as OPEN. The SLA countdown begins, and you
will be notified in the ticket when the partner responds.

{% hint style="danger" %}
**If you see "Submit failed", nothing was sent, and submitting again is the
right thing to do.** Requires app **v1.0.63 or later**, where that message
means what it says.

On older versions it did not. Creating the case on TSANet and recording it on
your own Zendesk ticket shared one error handler, so a failure in the second
step showed **Submit failed** beside a dialog still filled in with the same
request. The case existed and the partner already had it. Pressing Submit
again opened a **second** collaboration request to that partner.

From v1.0.63 the dialog closes the moment the case is created, so re-sending
is unavailable rather than merely discouraged. If the follow-up bookkeeping
then fails, the message tells you the request **was** submitted, names the
case token so it can be picked up by hand, and says not to submit again. That
second step can fail in ordinary use: it writes to the ticket under Zendesk's
safe-update guard, which is designed to refuse when a trigger, an automation,
or another agent touches the same ticket at the same moment. Opening a
collaboration case from a live support ticket is exactly when that happens.
{% endhint %}

## Working an Inbound Request

When a partner vendor needs help from your team, the request arrives as a new
Zendesk ticket with the TSANet case detail already attached.

Open the ticket and review the request in the TSANet panel, then respond using
one of the action buttons.

> 📸 **Screenshot placeholder:** The action buttons on an inbound TSANet case
> (Accept, Reject, Request More Info).

<details>

<summary>Accept the request</summary>

Click **Accept** to take on the collaboration. You will be asked to provide
your engineer contact detail and next steps for the partner. Once accepted, the
case status changes to ACCEPTED and the initial SLA clock stops.

</details>

<details>

<summary>Request more information</summary>

Click **Request More Info** if you cannot act on the request as written, for
example if entitlement detail or a clearer problem description is needed. The
case moves to INFORMATION status, and you can exchange notes with the submitter
until you have what you need.

</details>

<details>

<summary>Reject the request</summary>

Click **Reject** if your team cannot take on the collaboration. You will be
asked for a reason, which is sent back to the submitting partner.

</details>

{% hint style="info" %}
Because inbound requests are not high volume, set up a notification for the
team that manages inbound intake so requests are not missed.
{% endhint %}

## Working a Case Without the Sidebar App (Native Fields)

Some teams run the integration without the sidebar app. The inbound actions
above (Accept, Request More Info, Reject, and Add Note) are all available
directly from two native Zendesk fields, so an agent never has to leave the
ticket form:

* **TSANet Action** is a dropdown that triggers an action: Accept, Reject,
  Request Info, or Add Note.
* **TSANet Action Text** is a text field that carries the supporting text, such
  as a reject reason, an information question, or a note body.

To act on a case, type any required text into **TSANet Action Text**, set
**TSANet Action** to the action you want, and submit the ticket. The integration
runs the action against TSANet, records a confirmation as an internal comment,
and clears the Action field so it is ready for the next action.

| You want to | Set TSANet Action to | Put in TSANet Action Text |
| --- | --- | --- |
| Accept an inbound request | Accept | nothing required |
| Ask the partner for more detail | Request Info | your question |
| Decline the request | Reject | the reason |
| Send a note to the partner | Add Note | the note body |

{% hint style="info" %}
**One-click macros.** Your administrator can create a Zendesk macro for each
action (for example **TSANet: Accept** or **TSANet: Send partner-only note**) so
an agent applies the macro instead of opening the dropdown by hand. The agent
still types any needed text into TSANet Action Text first. See the
[Installation Guide](installation-guide) for how to create them.
{% endhint %}

> 📸 **Screenshot placeholder:** The TSANet Action dropdown and TSANet Action
> Text field on a Zendesk ticket.

Creating a brand-new outbound request still needs the sidebar app (or the TSANet
web app), because each partner's request form is different. The native fields
cover responding to inbound cases and exchanging notes.

## Responding and Adding Notes

Once a collaboration is active, both sides keep working it through notes.

* **Respond Now** posts a response to the partner on an open request.
* **Add Note** posts a note to the collaboration. A note has a **Subject** (a
  short summary) and optional **Details** (the longer body); fill in only the
  Subject if a short note is enough. When you add a note you also choose who can
  see it:
  * **Internal** (the default) keeps the note in Zendesk only. It is **not**
    sent to the partner. Use it for private notes to your own team.
  * **Partner only** sends the note to the partner but **hides it from the end
    customer**. Use it for partner-directed troubleshooting the customer should
    not see. It also appears as an internal note on the ticket so your team has
    a record.
  * **Public** sends the note to the partner **and** shows it to the end
    customer on the ticket.

{% hint style="info" %}
**Partner-only lives only in the app dialog or the TSANet Action field.**
Zendesk's own reply box offers just *Public reply* and *Internal note* and
cannot be changed, so it cannot send a partner-only note. A normal **public
reply** typed in the Zendesk composer reaches the partner *and* the end customer
(it is forwarded to the partner automatically); an **internal note** stays in
Zendesk only. To reach the partner while hiding from the customer, use **Add
Note, Partner only** in the sidebar, set **TSANet Action to Add Note** on the
ticket, or apply the **TSANet: Send partner-only note** macro.
{% endhint %}

> 📸 **Screenshot placeholder:** The Add Note dialog showing the Subject and
> Details fields and the Internal / Partner only / Public choice.

Notes posted by the partner are mirrored into the Zendesk ticket as internal
comments, so the whole team can follow the conversation from the ticket without
opening the panel. Each note is labeled by direction, **You** for notes your
team sent and the partner company for notes received, so it is clear who said
what. Each mirrored note appears only once, and your own public replies are not
echoed back into the ticket as duplicates.

## Ongoing Collaboration

While a collaboration is active:

* New notes from the partner appear in the ticket thread automatically.
* The case status and SLA countdown in the panel stay current.
* The management team can monitor progress across all collaborations and align
  the work with the organization's own case-handling process.

{% hint style="info" %}
Every TSANet ticket is tagged `tsanet_outbound` or `tsanet_inbound`. Your
administrator can build a Zendesk View on these tags (or on the TSANet Status
field) so the team can watch all active collaborations from one list without
opening each ticket.
{% endhint %}

The submitting company closes the collaboration when the work is complete.

## TSANet Case Status Definitions

| Status | Meaning |
| --- | --- |
| OPEN | The request has been sent and is awaiting a first response. The SLA clock is running. |
| ACCEPTED | The receiving partner has taken on the collaboration. The initial SLA clock has stopped. |
| INFORMATION | The receiving partner needs more information before they can accept. |
| REJECTED | The receiving partner has declined the collaboration. |
| CLOSED | The collaboration is complete. Only the submitting company can close a case. |

{% hint style="info" %}
The SLA deadline tracks only the first response. Once a case has been accepted,
rejected, or moved to INFORMATION, the SLA countdown stops.
{% endhint %}

## Need Help

For questions about using the integration, contact membership@tsanet.org.
