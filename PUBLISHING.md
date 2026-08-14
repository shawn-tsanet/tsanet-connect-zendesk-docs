# Publishing map — Zendesk connector docs → GitBook

How the markdown in `gitbook-sandbox/zendesk-connector/` reaches GitBook.
The agent (tsanet-ops) owns publishing; content approval in chat is the
publish approval. Humans do not run these commands.

## The only sanctioned path: cr-publish (per page)

```bash
# from the tsanet-ops main checkout, .env loaded
.venv/bin/python scripts/gitbook-page-update.py cr-publish \
  --space-id <SPACE_ID> \
  --update "<PAGE_ID>=gitbook-sandbox/zendesk-connector/<file>.md" \
  --subject "<what changed>"
```

Do **NOT** use:
- `git/import` — silently no-ops on spaces without a Git provider (GitBook
  contract change, ~2026-07; tracked in tsanet-ops#124)
- **Git Sync** — whole-space replace. Installing it on the shared live
  Documentation space wiped all products' docs on 2026-07-24 (recovered via
  UI Version history rollback). Never again.

## Spaces

| Space | ID | Notes |
|---|---|---|
| Documentation (LIVE, public) | `FnKqXLEByXxXehD4FNIm` | Shared with all products — only ever touch the Zendesk App pages below |
| tsanet-ops sandbox | `kaogBxzUeP7OMwN0iEqz` | Staging copy, Zendesk-only |

## File → page IDs

| File | LIVE page | Sandbox page |
|---|---|---|
| `installation-guide.md` | `5FlDPw223LZh93g58WXx` | `tKATo5xYu77UwOmACROK` |
| `user-guide.md` | `XBhXwuCcbZ8wtvN4qXvJ` | `2oT3ZGpcU7t8O4h15KZv` |
| `auto-accept-guide.md` | `TS6KK9Ha4RcoCKrIxlLA` | `LbfXlwayefCWS8cJz3ER` |
| `troubleshooting-guide.md` | `vedHvvcF9xsUCsYZ8W7d` (live title: "Debug & Report Issues") | `qXsNk3qOOJBLlU6LjGPL` |
| `security-considerations.md` | `dIHGbD3q7JUk0ROOVwil` | `NHdvTxvNYbpMWadpSvAo` |

Publish every changed file to **both** spaces. `SUMMARY.md` is not published
(page create/delete/reorder has no scripted path yet — tsanet-ops#124). Page
**creation** is done with a raw change-request `insert_page` op + verify +
merge (first done for `auto-accept-guide.md`, 2026-08-14: live CR 441, sandbox
CR 24; live parent group `EjOHvkflli6hLJPRmrS9` "Zendesk App"); record the new
page ids in the table above afterward.

## Gotchas

- The script's size-delta verify guard blocks merges when a page grows/shrinks
  beyond ~25%. For a legitimate large edit it leaves the CR open: verify the CR
  page content by probes
  (`GET /spaces/{space}/change-requests/{cr}/content/page/{page}?format=markdown`),
  then merge with `POST /spaces/{space}/change-requests/{cr}/merge`.
- Page IDs are stable across cr-publish updates, but re-verify with
  `list-pages` if a page was ever recreated.
