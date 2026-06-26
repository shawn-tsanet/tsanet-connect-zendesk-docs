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

The integration has two moving parts, plus one optional helper:

* **The ZAF sidebar app** runs inside Zendesk, in the support agent's browser.
* **ZIS (Zendesk Integration Services)** runs server-side inside Zendesk's own
  infrastructure and handles inbound delivery from TSANet.
* **An optional GitHub Actions job** (`sla-monitor`) runs on a schedule to flag
  SLA breaches when no agent is online.

{% hint style="info" %}
**Bottom line.** No data leaves Zendesk except what is required to collaborate
with a partner over TSANet, every connection is TLS-encrypted, and the only
credential at meaningful risk (the TSANet API password) belongs to a dedicated
service account scoped to your TSANet membership. The one item a reviewer should
weigh is how that password is stored in the ZAF app today. See
[Credential Handling](#authentication-and-credential-storage) and
[Known Considerations and Hardening](#known-considerations-and-hardening).
{% endhint %}

## Architecture and Trust Boundaries

There are four trust zones. Data crosses a boundary only at the arrows below.

| Zone | What runs there | Trust boundary |
| --- | --- | --- |
| **Agent browser** | The ZAF sidebar app (UI + background poller) | Cross-origin sandboxed iframe served by Zendesk |
| **Zendesk infrastructure** | ZIS flows, connections, ticket data | Zendesk's hosted platform and secret store |
| **TSANet Connect** | The TSANet API and partner routing | External party, reached over TLS |
| **GitHub Actions (optional)** | The `sla-monitor` job | GitHub-hosted runner with encrypted secrets |

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
| ZAF app to TSANet API | JWT Bearer token, obtained by logging in with the TSANet API username and password from the app settings | Username and password in the app's installation settings. The JWT is held in browser memory only (about 50 minutes) and is never written to local or session storage. |
| ZIS to TSANet API | OAuth 2.0 client credentials (Microsoft Entra). ZIS mints and renews its own short-lived tokens | The long-lived Entra client credential is stored **server-side** in the Zendesk ZIS connection store. There is no static token at rest. |
| TSANet to ZIS (inbound) | HTTP Basic auth on every delivery, enforced by ZIS. TSANet additionally signs each delivery with an HMAC-SHA256 signature (`X-Hub-Signature-256`) | Per-subscription ingest credentials, held by TSANet for delivery and by ZIS for verification. |
| ZIS to Zendesk | A basic-auth ZIS connection using a Zendesk API token | Stored **server-side** in the Zendesk ZIS connection store. |
| GitHub Actions (optional) | Zendesk API token and TSANet API credentials | Encrypted GitHub Actions repository secrets. |

{% hint style="warning" %}
**Credential handling, stated plainly.** The TSANet API username and password are
stored as standard Zendesk app settings and are read by the app's front-end code,
because the app calls the TSANet API directly from the browser. They are
therefore readable inside the app's browser context (for example, by an
administrator inspecting the running app). Mitigating factors: the credential
belongs to a **dedicated TSANet service account** (not a person), is scoped to
your TSANet membership, travels only over TLS, and the resulting token is never
persisted in the browser. The roadmap to remove even this exposure is in
[Known Considerations and Hardening](#known-considerations-and-hardening).
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
  credential and the Zendesk API token) are stored inside Zendesk's hosted secret
  store and are not retrievable in plaintext through the API.

## Access Control and Least Privilege

* **Dedicated service account.** The TSANet API credential is a service account
  belonging to the integration, not an individual, on your registered domain.
* **Scoped ZIS management.** ZIS connections and flows can only be managed with a
  ZIS OAuth token of the correct scope. A standard Zendesk API token cannot read
  or alter them.
* **Fail-closed forwarding.** Comment forwarding to the partner only fires for
  agent or admin public replies. End-user replies and internal notes are never
  forwarded.
* **Asymmetric close.** Only the submitting company can close a collaboration, so
  a receiving member cannot unilaterally end a case it did not open.

## Attack Surface Summary

| Surface | Control |
| --- | --- |
| App distribution | Private app, ZIP install only. No public Marketplace listing. |
| Outbound exfiltration | `domainWhitelist` limits egress to the two TSANet hosts. |
| Inbound spoofing | Basic auth on every delivery, plus an HMAC-SHA256 signature available for verification. |
| Unauthorized forwarding | Fail-closed agent/admin author guard. |
| Credential theft in transit | TLS on every hop; token held in memory only. |
| Stale or leaked tokens | Short-lived tokens; ZIS re-mints its own; the ZAF JWT expires in about 50 minutes. |

## Operational Security Guidance for Members

* Use a **dedicated TSANet API service account** on your registered domain, never
  a personal login or a customer address.
* **Rotate the TSANet API password** on a schedule. When you do, update the ZAF
  app settings and (if you run the optional SLA monitor) the matching GitHub
  Actions secret.
* **Restrict Zendesk administrator access.** Only administrators can view or
  change the app settings, so administrator accounts are the sensitive surface.
* Keep the app on `BETA` only during setup and testing, and switch to
  `PRODUCTION` when you go live.
* If you run the optional GitHub Actions job, treat its repository secrets like
  any production credential and watch for failed runs, which can indicate an
  expired secret.

## Known Considerations and Hardening

This section is candid about the current tradeoffs and the work planned to close
them.

### TSANet credential is readable in the app's browser context

**Current state.** The ZAF app calls the TSANet API directly from the browser, so
the API password is a non-secure app setting (`secure: false`) and is read by
front-end code. Anyone who can inspect the running app in the agent's browser can
read it.

**Why it is acceptable today.** It is a dedicated service account scoped to your
TSANet membership, used over TLS, with the resulting token kept in memory only.

**Hardening path.** Zendesk's ZAF framework supports **secure settings**
(`secure: true`), which keep a value out of the front-end entirely and inject it
server-side into outbound requests. Adopting this for the TSANet password
requires routing the app's TSANet calls through `client.request()` with
server-side templating instead of a direct browser `fetch()`. That is an
architectural change to the app's request path and is tracked as a hardening item
in the `tsanetgit/Zendesk_App` repository.

### Inbound signature verification

Inbound deliveries are authenticated by HTTP Basic auth, which ZIS enforces.
TSANet additionally signs each delivery with an HMAC-SHA256 signature. Confirm
that the receiving flow verifies that signature against the per-subscription
secret, so authenticity does not rest on Basic auth alone.

### Secret rotation

The Entra client credential, the ingest credentials, and the Zendesk API token
should each be rotated on a defined schedule. Rotating the Entra client
credential is the highest-value rotation because it is the longest-lived secret
in the system.

## Questions

For a deeper security review, environment access, or to request the current
hardening status, contact membership@tsanet.org.
