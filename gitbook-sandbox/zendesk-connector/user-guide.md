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
  Respond Now, and Add Note (with a Public / Internal choice)

The panel refreshes automatically about once a minute while any agent has Zendesk
open, so the case detail stays current without a manual refresh.

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

## Responding and Adding Notes

Once a collaboration is active, both sides keep working it through notes.

* **Respond Now** posts a response to the partner on an open request.
* **Add Note** posts a note to the collaboration. A note has two parts: a
  **Subject** (a short summary) and optional **Details** (the longer body).
  Fill in only the Subject if a short note is enough. When you add a note you
  also choose who can see it:
  * **Internal** (the default) keeps the note in Zendesk only. It is **not**
    sent to the partner — use it for private notes to your own team.
  * **Public** sends the note to the partner **and** shows it to the end
    customer on the ticket.

{% hint style="info" %}
The rule is simple: **public content reaches the partner; internal notes stay
in Zendesk.** A normal **public reply** typed in the Zendesk composer is also
forwarded to the partner automatically — so you can answer a partner from the
ticket itself, not only from the panel. Internal comments are never forwarded.
{% endhint %}

> 📸 **Screenshot placeholder:** The Add Note dialog showing the Subject and
> Details fields and the Public / Internal choice.

Notes posted by the partner are mirrored into the Zendesk ticket as internal
comments, so the whole team can follow the conversation from the ticket without
opening the panel. Each mirrored note appears only once, and your own public
replies are not echoed back into the ticket as duplicates.

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
