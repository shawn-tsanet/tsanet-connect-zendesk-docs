---
description: >-
  Security posture of the TSANet Connect integration for Zendesk: trust
  boundaries, data flows, authentication and secret storage for the ZAF sidebar
  app and ZIS, transport security, least privilege, and the hardening roadmap.
---

# Security Considerations

This document describes how the TSANet Connect integration for Zendesk is secured.
It is written in two layers: a posture summary for security and IT reviewers, and
an internal hardening section that is candid about known tradeoffs and the
roadmap to close them.

The integration has two moving parts:

* **The ZAF sidebar app** runs inside Zendesk, in the support agent's browser.
* **ZIS (Zendesk Integration Services)** runs server-side inside Zendesk's own
  infrastructure and handles inbound delivery from TSANet.

Nothing is hosted outside Zendesk and TSANet — there is no external server,
scheduler, or third-party runner in the architecture.

{% hint style="info" %}
**Bottom line.** No data leaves Zendesk except what is required to collaborate
with a partner over TSANet, and every connection is TLS-encrypted. The TSANet API
password is stored as a Zendesk **secure setting** (kept out of the app's
front-end code and injected server-side at login), and it belongs to a dedicated
service account scoped to your TSANet membership. ZIS connection secrets live in
Zendesk's server-side secret store. See
[Credential Handling](#authentication-and-credential-storage) and
[Known Considerations and Hardening](#known-considerations-and-hardening).
{% endhint %}

## Architecture and Trust Boundaries

There are three trust zones. Data crosses a boundary only at the arrows below.

| Zone | What runs there | Trust boundary |
| --- | --- | --- |
| **Agent browser** | The ZAF sidebar app (UI + background poller) | Cross-origin sandboxed iframe served by Zendesk |
| **Zendesk infrastructure** | ZIS flows, connections, ticket data | Zendesk's hosted platform and secret store |
| **TSANet Connect** | The TSANet API and partner routing | External party, reached over TLS |

The agent browser never talks to the partner directly. Outbound requests go to
the TSANet API; inbound events arrive from TSANet into ZIS. The partner's systems
and your systems are never directly connected. TSANet sits in the middle and
routes the collaboration.

## Data Flows and What Leaves Zendesk

| Direction | Data that crosses the boundary | Goes to |
| --- | --- | --- |
| Outbound request | The collaboration detail your agent enters (problem summary, partner, responding engineer email) | TSANet, then the partner |
| Inbound request | The partner's case detail | Into a Zendesk ticket |
| Notes and replies | Only **public** notes and public replies | TSANet, then the partner |
| Internal comments | Never leave Zendesk | Stay on the ticket |

Two controls limit what can leave:

* **Public-only propagation.** Only public content reaches the partner. Internal
  Zendesk comments are never forwarded. A fail-closed guard on the comment
  forwarding flow only forwards replies authored by an **agent or admin**; an end
  customer's public reply is never forwarded.
* **Registered-domain enforcement.** TSANet rejects any submitter or responding
  engineer email that is not on your company's TSANet-registered domain. This is
  enforced server-side by TSANet, independent of the app, and prevents a case
  from being attributed to a customer address or a personal account.

## Authentication and Credential Storage

Each connection authenticates independently. No single credential unlocks
everything.

| Connection | Method | Where the secret lives |
| --- | --- | --- |
| ZAF app to Zendesk | The agent's own Zendesk session, inherited via the ZAF SDK (`client.request()`) | Nothing stored. The app never holds a Zendesk credential. |
| ZAF app to TSANet API | JWT Bearer token, obtained at login. The password is a ZAF **secure setting**: the login request goes through the Zendesk proxy, which substitutes the password server-side, so it never appears in front-end code | Username is a normal app setting; the password is a **secure** setting, not exposed to the browser. The resulting JWT is held in browser memory only (about 50 minutes) and is never written to local or session storage. |
| ZIS to TSANet API | OAuth 2.0 client credentials (Microsoft Entra). ZIS mints and renews its own short-lived tokens | The long-lived Entra client credential is stored **server-side** in the Zendesk ZIS connection store. There is no static token at rest. |
| TSANet to ZIS (inbound) | HTTP Basic auth on every delivery, enforced by ZIS. TSANet additionally signs each delivery with an HMAC-SHA256 signature (`X-Hub-Signature-256`) | Per-subscription ingest credentials, held by TSANet for delivery and by ZIS for verification. |
| ZIS to Zendesk | OAuth 2.0 client credentials against your **own** instance, scope-limited to `read tickets:write`. ZIS mints and renews its own short-lived tokens (about 30 minutes) | The OAuth client credential is stored **server-side** in the Zendesk ZIS connection store. There is no static token and no API token at rest. |

{% hint style="info" %}
**Credential handling, stated plainly.** The TSANet API password is stored as a
Zendesk **secure setting** (`secure: true`). Secure settings are not exposed to
the app's front-end code; when the app logs in to TSANet, the request is made
through the Zendesk proxy, which substitutes the password server-side, so the
cleartext password is never present in the agent's browser. Additional factors:
the credential belongs to a **dedicated TSANet service account** (not a person),
is scoped to your TSANet membership, travels only over TLS, and the resulting
token is held in memory only. This was shipped in ZAF app **v1.0.42** and
validated end to end on Beta.
{% endhint %}

## Transport and Network Security

* **TLS everywhere.** Every connection (browser to TSANet, ZIS to TSANet, TSANet
  to ZIS, and all Zendesk API traffic) is HTTPS.
* **Egress allow-list.** The ZAF app declares a `domainWhitelist` of exactly two
  hosts (`connect2.tsanet.net` and `connect2.tsanet.org`). Zendesk blocks the app
  from making requests to any other external domain, so the app cannot exfiltrate
  data to an arbitrary endpoint.
* **Cross-origin sandbox.** The app runs in a sandboxed, cross-origin iframe.
  Browser primitives such as `prompt()` and `confirm()` are blocked, and the app
  cannot reach the parent Zendesk page outside the ZAF SDK's defined channel.

## Data at Rest

* **No external database.** The integration is stateless. All state lives in
  Zendesk ticket fields and the TSANet API. The app holds no datastore of its own.
* **Tokens in memory only.** The ZAF app's TSANet JWT exists only in browser
  memory for the life of the session and is re-minted on expiry. Nothing is
  written to local or session storage.
* **Secrets held server-side.** ZIS connection secrets (the Entra client
  credential and the Zendesk OAuth client credential) are stored inside Zendesk's
  hosted secret store and are not retrievable in plaintext through the API.
* **No API token, anywhere.** The integration neither stores nor uses a Zendesk
  API token — setup commands authenticate with short-lived OAuth setup tokens
  minted from the integration's confidential OAuth client, and the runtime
  connection mints its own. (Zendesk is retiring API tokens for the Ticketing
  API — all tokens stop working on April 30, 2027 — so this is a compliance
  point as well as a hardening one.) The one-time flow-bundle upload is the only
  call that rejects OAuth on Zendesk's side; it runs inside the TSANet Connect
  app on the installing admin's own session, so it needs no API token, no
  password access, and no long-lived credential of any kind.

## Data Retention and the PII Lifecycle

Cross-org case content (partner company names, submitter and engineer contact
details, problem narratives) mirrors into ticket subjects, descriptions,
comments, and custom fields, and persists until the member removes it. Three
facts govern the lifecycle:

* **Three-copy model.** Every collaboration exists as your ticket, the TSANet
  platform case, and the partner's CRM case, each under a different
  controller. Anything you delete or scrub applies to your copy only; the
  Connect API's notes are append-only, so no erasure propagates cross-org.
  Cross-org erasure is controller-to-controller coordination through TSANet.
* **Selectors are built in.** Every ticket the integration touches is tagged
  `tsanet_inbound` or `tsanet_outbound`; retention tooling keys on those tags.
* **Two supported removal modes.** Whole-ticket deletion (native deletion
  schedules, which act on tickets closed 120+ days; tag-scoped schedules
  require the ADPP add-on, otherwise use a scheduled API sweep) or selective
  scrubbing that preserves the case record (the redaction API permanently
  removes comment and attachment content; note that cleared field values
  remain in ticket audit history, which only deletion removes).

The retention window is the member's policy decision as data controller. The
full data map and both recipes: [PII Retention and Data
Handling](https://github.com/tsanetgit/Zendesk_App/blob/main/PII_Retention_and_Data_Handling.md).

## How the App Package Reaches You

The connector ships as a ZIP that you upload into your own Zendesk instance, so
the integrity of that file is a shared concern: TSANet controls how it is
built and published, and you control what you install.

On the publishing side, the release pipeline builds only from a tagged commit
with no separate build step, so the package cannot diverge from the source it
claims to come from. It runs with a least-privilege token, pins its build
actions to exact commit revisions, and publishes only after an explicit human
approval. Each release carries a **build-provenance attestation** and a
**SHA-256 checksum**.

On your side, those two artifacts let you confirm that the ZIP you downloaded
is the one TSANet published, rather than a file altered in transit or
substituted somewhere along the way:

{% code overflow="wrap" %}
```bash
# Provenance (requires the GitHub CLI)
gh attestation verify tsanet-connect-v<version>.zip --repo tsanetgit/Zendesk_App

# Or checksum only — run where the ZIP and checksums.txt both sit
shasum -a 256 -c checksums.txt
```
{% endcode %}

{% hint style="danger" %}
If verification fails, do not install the package, and contact
membership@tsanet.org. A failure means the file does not match what TSANet
published. This is worth reporting even if a later re-download verifies
cleanly.
{% endhint %}

Step-by-step instructions sit in the [Installation
Guide](installation-guide.md); the commands are repeated here because
verification is a security control rather than an installation detail.

## Access Control and Least Privilege

* **Dedicated service account.** The TSANet API credential is a service account
  belonging to the integration, not an individual, on your registered domain.
* **Scoped ZIS management.** ZIS connections and flows can only be managed with a
  ZIS OAuth token of the correct scope. A standard Zendesk API token cannot read
  or alter them.
* **Scope-limited Zendesk access.** The `zendesk` connection's OAuth tokens carry
  only `read tickets:write` — the integration can read data and write tickets,
  but cannot modify users, settings, or anything else in the account.
* **Fail-closed forwarding.** Comment forwarding to the partner only fires for
  agent or admin public replies. End-user replies and internal notes are never
  forwarded.
* **Asymmetric close.** Only the submitting company can close a collaboration, so
  a receiving member cannot unilaterally end a case it did not open.

## Attack Surface Summary

| Surface | Control |
| --- | --- |
| App distribution and tampering | Private app, ZIP install only, no public Marketplace listing. The release pipeline runs least-privilege, pins its actions to commit SHAs, and publishes only after a required human approval; every package ships with a build-provenance attestation and a SHA-256 checksum that members verify before installing. |
| Outbound exfiltration | `domainWhitelist` limits egress to the two TSANet hosts. |
| Inbound spoofing | Basic auth (`callbackAuth`) on every delivery, enforced by ZIS; two independent rotatable secrets (capability-token ingest URL + Basic credential); event content is pulled from the TSANet API by token, so forged deliveries cannot inject case content. |
| Unauthorized forwarding | Fail-closed agent/admin author guard. |
| Credential theft in transit | TLS on every hop; token held in memory only. |
| Stale or leaked tokens | Short-lived tokens; ZIS re-mints its own; the ZAF JWT expires in about 50 minutes. |

## Operational Security Guidance for Members

* **Verify each package before installing it.** Check the provenance
  attestation or the checksum on every install and upgrade, not only the first
  one. See *How the App Package Reaches You* above.
* Use a **dedicated TSANet API service account** on your registered domain, never
  a personal login or a customer address.
* **Rotate the TSANet API password** on a schedule. When you do, update the ZAF
  app settings.
* **Restrict Zendesk administrator access.** Only administrators can view or
  change the app settings, so administrator accounts are the sensitive surface.
* Keep the app on `BETA` only during setup and testing, and switch to
  `PRODUCTION` when you go live.
* **Leave password access disabled.** Nothing in this integration needs it,
  including the flow-bundle upload, which runs on the installing admin's session
  inside the app. If a procedure ever asks you to enable password access for
  TSANet Connect, it is out of date.

## Known Considerations and Hardening

This section is candid about the current tradeoffs and the work planned to close
them.

### TSANet credential storage (resolved in v1.0.42)

**Resolved.** The TSANet API password is stored as a Zendesk **secure setting**
(`secure: true`), so it is not exposed to the app's front-end code. At login the
app calls TSANet through the Zendesk proxy (`client.request()` with
`secure: true`), and the proxy substitutes the password server-side, so the
cleartext never reaches the agent's browser. The token-based calls that follow
carry only the short-lived JWT, not a secret.

**History.** Earlier versions stored the password as a non-secure setting and
called the TSANet API directly from the browser via `fetch()`, which made the
password readable in the app's browser context. This was hardened in ZAF app
**v1.0.42** (tracked as issue #49 in `tsanetgit/Zendesk_App`) and validated end to
end on Beta.

### Inbound signature verification (platform-gated; compensating controls in place)

TSANet signs each delivery with an HMAC-SHA256 signature, but the Zendesk
Integration Services platform cannot verify it: ZIS inbound webhooks support
Basic auth only, and flows receive parsed events with no access to the raw
body or headers. Authenticity therefore rests on two independent rotatable
secrets (the capability-token ingest URL and the Basic `callbackAuth`
credential), with blast radius bounded because the flow pulls case content
from the TSANet API by token rather than trusting the delivery body. Members
whose posture requires signature verification can front the ingest URL with a
small verification proxy; the pattern is documented in
[https://github.com/tsanetgit/Zendesk_App/issues/90](https://github.com/tsanetgit/Zendesk_App/issues/90), which records the full
disposition and evidence.

### Secret rotation (webhook credential rotation shipped)

The ingest (webhook Basic) credential now has a scripted make-before-break
rotation with no delivery downtime and true revocation of the old credential:
see the [rotation runbook](https://github.com/tsanetgit/Zendesk_App/blob/main/ZIS_Rotation_Runbook.md)
and `scripts/rotate-inbound-webhook.py`. Rotating it on a schedule (quarterly
is a reasonable default) is the primary control on the inbound pipe. The Entra
client credential and the Zendesk OAuth client secret should also be rotated
on a defined schedule; scripted procedures for both are tracked in
[https://github.com/tsanetgit/Zendesk_App/issues/91](https://github.com/tsanetgit/Zendesk_App/issues/91). Note that regenerating a
Zendesk OAuth client secret invalidates the ZIS connection built on it —
re-register the ZIS OAuth client and recreate the connection as part of the
rotation.

### Zendesk API token retirement (migration complete)

Zendesk is removing API tokens as an authentication method for the Ticketing
API: creation is blocked for new accounts on July 28, 2026 and for all accounts
on October 27, 2026, and all existing tokens stop working on April 30, 2027.
The integration is ahead of this: no API token is used anywhere. The `zendesk`
ZIS connection uses OAuth client credentials (no stored token), and setup
commands authenticate with short-lived OAuth setup tokens minted from the
integration's confidential OAuth client. The flow-bundle upload is the one call
Zendesk refuses OAuth on, and it is handled without a token at all: the TSANet
Connect app performs it on the installing administrator's own session. Because
the session is the credential, it lasts exactly as long as the admin is signed
in, there is nothing to store, and nothing to rotate or revoke afterwards.

This also means the upload inherits the administrator's own privileges rather
than holding standing ones. Zendesk documents the ZIS registry endpoints as
admin-only, and the app additionally checks the signed-in user's role before
enabling the deploy button — but note that the in-app check is a usability
guard, not the security boundary. The boundary is Zendesk's own server-side
authorization on the endpoint.

## Questions

For a deeper security review, environment access, or to request the current
hardening status, contact membership@tsanet.org.
