# TSANet Connect — Zendesk Integration Guide

TSANet Connect does not yet ship a managed Zendesk package. This guide provides everything you need to build a production-ready integration yourself using Zendesk's native tooling — no custom server required.

The integration uses **two required components**, plus one optional add-on:

| Component | What It Does | Technology |
|---|---|---|
| **ZAF Sidebar App** | Agents view and act on TSANet collaboration cases directly inside Zendesk tickets. Handles Accept, Reject, Request Info, Add Note (with Internal / Partner-only / Public visibility), and Close. Runs a background poller as an inbound-delivery fallback. | Zendesk Apps Framework (private app, ZIP upload) |
| **ZIS OAuth/Entra Connection** | Authenticates Zendesk to the TSANet API using Microsoft Entra client-credentials — ZIS holds the long-lived client credential and mints/renews its own short-lived tokens, so there's no refresh job to run. The same ZIS integration also receives inbound case/note events via an authenticated webhook (`callbackAuth`) and forwards agents' public replies back to the partner. | Zendesk Integration Services (ZIS) |
| **GitHub Actions SLA Monitor** *(optional)* | Scheduled job that checks TSANet OPEN cases for SLA breaches and tags Zendesk tickets to fire an email alert. External add-on — needs its own GitHub repo, Actions, and secrets. Skip it and TSANet still enforces the SLA server-side. | GitHub Actions (free tier sufficient) |

{% hint style="info" %}
**Estimated setup time:** 1–1.5 hours for the two required components following the Quick Start guides end-to-end. No development environment or custom server required.
{% endhint %}

---

## Architecture Overview

```
AGENT ACTS ON A COLLABORATION (ZAF Sidebar)
──────────────────────────────────────────────────────────────────

  Zendesk Ticket Sidebar          ZAF App (iframe)           TSANet Connect API
  ┌──────────────────┐            ┌────────────────────┐     ┌──────────────────┐
  │ TSANet Connect   │            │ Fallback poll every │     │                  │
  │ panel shows:     │            │ 1 min (only backs  │────▶│ /v1/collab-reqs  │
  │ • Case status    │            │ up ZIS push)        │     │ /v1/notes        │
  │ • SLA countdown  │◀── proxy ──│ Agent clicks       │────▶│ /v1/approval     │
  │ • Notes feed     │            │ Accept / Reject /  │     │ /v1/rejection    │
  │ • Partner info   │            │ Add Note / Close   │     │ /v1/notes (POST) │
  └──────────────────┘            └────────────────────┘     └──────────────────┘


INBOUND CASE + NOTE DELIVERY (ZIS — primary path)
──────────────────────────────────────────────────────────────────

  TSANet Connect API ── authenticated webhook (callbackAuth) ──▶ ZIS ──▶ Zendesk
  (new case / note)                                                     ticket + comment

  The ZAF background poller only creates a ticket itself if one still
  doesn't exist 3 minutes after a case opens — an automatic fallback,
  not the primary path.


OUTBOUND COMMENT FORWARDING (ZIS)
──────────────────────────────────────────────────────────────────

  Agent posts a PUBLIC reply on a TSANet ticket
    → Zendesk trigger → webhook → ZIS flow_forward_comment
    → guards (token / comment / author) → POST /notes to the partner

  Internal comments are never forwarded — see Known Limitations.


AUTH (Microsoft Entra client-credentials — replaces the retired bearer-token job)
──────────────────────────────────────────────────────────────────

  ZIS holds a long-lived Entra client credential and mints/renews its
  own short-lived tokens. No scheduled refresh job, no static bearer
  connection to maintain.


SLA MONITORING (GitHub Actions — optional, not part of the core integration)
──────────────────────────────────────────────────────────────────

  GitHub Actions (:00 and :50)
  └── sla-monitor job
        TSANet: GET OPEN cases
        For each past-deadline case:
          Zendesk: find ticket by TSANet token field
          Zendesk: POST tag tsanet_sla_breached
          → Zendesk trigger fires → emails ticket assignee
```

---

## How the Token Links Everything

Every TSANet collaboration case has a unique `token` — a string that is the primary key for all API operations. The ZAF app reads this token from a custom field on the Zendesk ticket, then uses it for every TSANet API call on that ticket.

When a new case arrives, the ZIS inbound webhook is the primary path: it creates the ticket and writes the token to the field. The ZAF background poller only steps in as a fallback if a ticket still doesn't exist a few minutes later. When the ZAF app creates a new *outbound* collaboration, it writes the returned token back to the ticket itself.

**Nothing works without the token being stored in the Zendesk ticket field.** This is the most important implementation detail.

---

## What the ZAF App Does

**On tickets with no TSANet case (compact mode):**
- Collapses to a slim 44px bar — unobtrusive on regular support tickets
- Shows a **+ New** button to start a new outbound collaboration

**On tickets linked to a TSANet case (full panel):**
- Displays all active collaborations with status and partner info
- **SLA countdown** — color-coded timer on unacknowledged (OPEN) cases only
- **Action buttons:** Accept, Reject, Request Info, Respond, Add Note, Close
- **Add Note visibility tiers** (v1.0.43) — Internal (default, Zendesk-only, never sent to TSANet), Partner-only (posted to TSANet with no public Zendesk comment), or Public (posts a public reply that a Zendesk trigger forwards to the partner)
- **Notes feed** — all TSANet case notes, direction-labeled (sent vs. received) and rendered as clean plain text
- **Notes mirrored to ticket thread** — partner and partner-only notes are posted as Zendesk internal comments so agents can see partner communication in the ticket timeline without opening the sidebar

**Background behavior (while any Zendesk tab is open):**
- Polls TSANet every 1 minute as a fallback for inbound case delivery (the ZIS webhook is primary — see Architecture Overview)
- Detects SLA breaches and tags tickets to fire email alerts, independent of whether the optional GitHub Actions monitor is deployed

{% hint style="info" %}
**ZIS inbound webhooks are supported** ([issue #2](https://github.com/tsanetgit/Zendesk_App/issues/2), closed). TSANet delivers new cases and notes to ZIS via an authenticated webhook secured by the `callbackAuth` capability, and this is the primary path for inbound delivery — not the ZAF poller. The poller remains registered purely as a fallback: it backfills a ticket only if push hasn't created one after a 3-minute grace window.
{% endhint %}

---

## Known Limitations

| Limitation | Workaround |
|---|---|
| ZIS scheduled polling (`flow_poll_tsanet`) is architecturally broken — ZIS flows cannot call ZIS management endpoints (circular OAuth scope) | Not needed: auth uses the Entra client-credentials connection (self-renewing), and inbound delivery uses the `callbackAuth` webhook push with the ZAF poller as fallback |
| Zendesk Views API silently ignores custom field columns | Add custom field columns to views manually in Admin Center → Workspaces → Views |
| ZIS custom integrations do not appear in Admin Center UI | Verify via `GET /api/services/zis/integrations/tsanet_connect/connections` API call |
| ZAF app binary updates via API are broken | Upload updated ZIP manually via Admin Center → Zendesk Support Apps → Update |

---

## Guides in This Section

* [ZAF App Quick Start](documentation/ZAF_Quick_Start.md) — install and configure the sidebar app (~30 min)
* [ZIS Quick Start](documentation/ZIS_Quick_Start.md) — connect ZIS to TSANet via OAuth client credentials (Entra) and deploy the flow bundle (~30 min)
* [Plain Language Implementation Guide](documentation/Zendesk_PlainLanguage_Implementation_Guide_v2.14.docx) — full narrative walkthrough with context and rationale (v2.14)
* [Claude Code Skill](documentation/SKILL_TSANet_Connect.md) — drop this into `~/.claude/skills/tsanet-connect/SKILL.md` to give Claude Code expert knowledge of this integration
* [GitHub Actions SLA Monitor (Optional)](https://github.com/tsanetgit/Zendesk_App/blob/main/GitHub_Actions_SLA_Monitor.md) — external add-on for an in-Zendesk SLA breach email alert; not required for the core integration

---

## Current App Version

**ZAF App: v1.0.43** (June 2026)

| Version | Key Change |
|---|---|
| v1.0.43 | Note direction labels (sent vs. received) + Partner-only visibility tier added to Add Note |
| v1.0.41 | Public Add Note posts the Zendesk comment only; the forwarding trigger delivers it to the partner exactly once (previously double-sent) |
| v1.0.40 | Internal notes stay in Zendesk only and never propagate to TSANet |
| v1.0.39 | Per-note Public/Internal visibility toggle added to Add Note |
| v1.0.38 | Public Zendesk replies on a TSANet ticket are forwarded to the partner as a TSANet note |
| v1.0.37 | ZIS inbound push (`callbackAuth`) promoted to primary inbound-delivery path; ZAF poller demoted to a 3-minute-grace fallback |
| v1.0.29 | Manifest cleanup before first member distribution: cleared dev-instance Field ID defaults so new installs require members to enter their own field IDs (was previously pre-populated with TSANet's dev instance values, causing silent failures on member installs) |
| v1.0.28 | TSANet notes mirrored to Zendesk ticket thread as internal comments (`syncNotesToZendesk`) |
| v1.0.27 | Add Note modal split into Subject + Details fields; eliminates duplication in TSANet web app |
| v1.0.25 | Adaptive height — compact 44px bar on non-TSANet tickets, full panel on TSANet tickets |
| v1.0.24 | App tray icon (128×128 transparent PNG) |
| v1.0.22 | SLA respondBy synced to Zendesk date field; GitHub Actions sla-monitor + Zendesk trigger |
| v1.0.20 | Close button hidden on inbound cases (submitter-only TSANet API restriction) |
| v1.0.18 | HTML stripped from TSANet note content before display |
| v1.0.16 | Accept fix — `engineerEmail` required field added; uses TSANet API username for domain validation |
| v1.0.15 | All action buttons fixed — replaced `prompt()`/`confirm()` with custom inline modal |
