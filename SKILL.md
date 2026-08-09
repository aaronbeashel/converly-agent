---
name: converly
description: Set up server side conversion tracking for ad platforms (Google Ads, Meta, GA4, LinkedIn, TikTok and more). Use this whenever the user wants to track form submissions, leads, or conversions from their ads, capture GCLID or FBCLID click IDs, fix broken conversion tracking, set up Google Enhanced Conversions or Meta CAPI, check which ads are converting, or connect a form tool like Typeform, Webflow forms, or a custom HTML form to an ad platform, even if they never mention Converly by name.
version: 1.0.0
homepage: https://developers.converly.io
metadata:
  openclaw:
    envVars:
      CONVERLY_API_KEY:
        required: false
        description: Optional. Only needed for headless use. The normal path is `converly login`, which opens a browser and stores credentials locally, no key handling involved.
---

# Converly: conversion tracking for AI agents

Converly tracks form submissions on a website and fires conversion events to ad platforms (Google Ads, Meta, GA4 and more), including server side delivery like Google Enhanced Conversions and Meta CAPI. You do the whole setup from the terminal with the `converly` CLI, verify it works with a real test event, and read conversion results afterwards.

Every CLI command prints one JSON document to stdout. Exit code 0 means success. Errors are JSON on stderr with the API's error code.

## Hard rules (read first)

1. **Never handle credentials in chat.** Do not ask the user to paste an API key, and never print `CONVERLY_API_KEY` or any token. Authentication is `converly login`, which happens in the user's browser.
2. **Connecting an ad platform requires a human.** `converly destinations connect` returns a URL the user must open in their browser. Never claim a destination is connected until `converly handoffs wait <id>` returns `"status": "completed"`. Never skip this step, and never work around it by asking for tokens.
3. **Never report tracking as working until it is proven.** Proof is `converly test-event` returning `"server_status": "success"`, or a real conversion appearing in `converly events list`. Publishing a flow is not proof.
4. **Set the site's domain before publishing.** A site with `"domain": null` rejects every event server side. The flow will publish fine and capture nothing. Check `converly sites list` and fix with `converly sites update <site_id> --domain <their-domain>` before you publish.
5. **Do not delete or unpublish anything you did not create in this session** without the user explicitly confirming first.
6. **If a command fails twice with the same error, stop and show the user the error.** Do not loop retries.

## Setup

Install the CLI if it is missing:

```
npm install -g converly
```

(Or run every command through `npx converly@latest ...` without installing.)

Authenticate. First check state with `converly whoami`:

- If it succeeds, you are logged in. It returns the subscription and the account's sites.
- If it fails with `not_logged_in`, run `converly login` and tell the user a browser window will open for them to log in. If they do not have a Converly account yet, run `converly login --signup` instead. Signing up starts a free trial automatically, no card needed. Wait for the command to finish, then re-run `converly whoami`.
- If `CONVERLY_API_KEY` is set in the environment, the CLI uses it automatically and no login is needed.

## The setup workflow

Work through these steps in order. Steps 3 and 4 can run in parallel.

### 1. Pick the site

```
converly sites list
```

Every account has a default site. Use it unless the user is tracking a second website. If `domain` is null, set it now (hard rule 4):

```
converly sites update site_XXXX --domain example.com
```

Use the site's real public domain. One domain covers both the apex and its www variant.

### 2. Install the tracking snippet

```
converly install snippet site_XXXX
```

Returns the `<script>` tag.

- **If you have access to the website's codebase** (you are running in their repo, or can edit their site), add the snippet to the `<head>` of every page yourself and deploy it. Say what you changed.
- **If you do not**, give the user the snippet with one clear instruction: paste it into the `<head>` of every page, or into their site builder's custom code slot (Settings → Custom Code in Webflow / Wix / Squarespace, a header scripts plugin in WordPress).

Check with `converly install status site_XXXX`. Read `detection` carefully:

- `"confirmed"` means tracking is proven live.
- `"never_seen"` does NOT mean the snippet is missing. A correctly installed site that has not captured a conversion yet looks exactly like this. Do not tell the user the install failed based on this value. The test event in step 6 is the real check.

### 3. Connect the ad platform

See what is available and what is already connected:

```
converly destinations list
```

If the target platform shows `"connected": false`:

```
converly destinations connect google-ads --site site_XXXX
```

Give the user the returned `url` and say: "Open this link in your browser to connect your Google Ads account." Then poll until they finish:

```
converly handoffs wait hdf_XXXX
```

This blocks up to 10 minutes. Links expire after 30 minutes; if the handoff expires, create a fresh one. Destination types: `google-ads`, `meta`, `google-analytics`, `linkedin-ads`, `tiktok-ads`, `reddit-ads`. Destinations are account wide, so one connection serves every flow.

### 4. Find the trigger

The trigger is the form tool on the user's website. Get the catalogue:

```
converly triggers
```

Match the user's form tool to a slug in `providers[]` (for example `typeform`, `webflow-forms`, `gravity-forms`, `jotform`, `hubspot-forms`). For a hand coded HTML form use `generic-form`. If you have repo access, identify the form tool from the code instead of asking.

### 5. Create and publish the flow

For Google Ads, first pick which conversion action to fire:

```
converly destinations conversions google-ads
```

Use the `id` of the conversion the user wants (ask if several look plausible). If the list is empty, the user must create a conversion action in Google Ads first (Goals → Conversions → New conversion action), then re-run with `--refresh`.

```
converly flows create --site site_XXXX --name "Demo requests" \
  --trigger generic-form --destination google-ads --conversion-id 123456789
```

For Meta or GA4, use an event name instead of a conversion id:

```
converly flows create --site site_XXXX --name "Leads to Meta" \
  --trigger webflow-forms --destination meta --event-name Lead
```

Add `--value 50 --currency USD` if the user wants a conversion value. Restrict to specific pages with `--pages /contact,/demo`. For anything richer (multiple actions, filters on a specific form), pass the whole flow body with `--json`; run `converly actions <destination>` to see each destination's config schema.

Then validate and publish:

```
converly flows validate flow_XXXX
converly flows publish flow_XXXX
```

`validate` returns `problems[]` (blockers, fix before publishing) and `warnings[]` (site readiness, for example `site_missing_domain`). Take warnings seriously, they are the "publishes fine, captures nothing" cases.

### 6. Verify end to end

```
converly test-event --flow flow_XXXX
```

This fires a real test conversion through to the ad platform and returns the platform's response. `"server_status": "success"` is your proof. For Meta, pass `--meta-code TEST12345` (from Meta Events Manager → Test events) so the test shows up there without polluting real data.

Only now report success to the user. Tell them: tracking is live, verified with a test conversion delivered to [platform], and real conversions will appear when someone submits the form after clicking an ad.

### 7. Keep their own traffic out

Offer to exclude the team's own form testing:

```
converly rules create --email-pattern '*@theircompany.com' --description "Internal team"
```

## After setup: reading results

This is the ongoing value. When the user asks "did my ads convert", "is tracking still working", or "who converted this week":

```
converly events list --limit 20
converly events list --since 2026-08-01T00:00:00Z --status failed
converly events get evt_XXXX
```

Each event carries per destination delivery status. `converly events get` shows exactly what the ad platform returned, plus any pipeline notices with user facing explanations. If a user reports a specific lead, find it with `--email person@example.com`.

Health checks: `converly install status <site>` (is the loader still live), `converly flows list` (is the flow still published), `converly subscription` (is the account in good standing).

## Common gotchas

- **`site_key` vs site id.** `site_1VQH84sr` (8 chars, in the snippet URL) is not the same as `site_29EtXn...` (22 chars, the API id). Commands take the API id.
- **The Meta slug is `meta`, not `meta-ads`.** The API rejects `meta-ads`.
- **Conversion value and currency nest inside the conversion.** The simple `flows create` flags handle this. If you build `--json` yourself, `value` and `currency` go inside the `conversion` object, and the enhanced conversions toggle is top level `enhancedConversions`. Copy the `config_example` from `converly actions <destination>`.
- **Editing a published flow does not change what runs.** After `flows update`, the old version keeps running until you `flows publish` again.
- **`events list` is a bounded snapshot of recent events, max 100, no paging.** Narrow with `--since`, `--flow`, `--email` instead of trying to page. An empty filtered result with `"complete": false` does not prove the conversion never happened.
- **Click IDs only exist on ad clicks.** GCLID/FBCLID are captured when a visitor arrives from an ad. Organic test submissions will not carry them; that is normal, not a bug.
- **402 `entitlement_required`** means the trial expired or billing lapsed. The user fixes this at app.converly.io → Settings → Billing. Do not retry around it.
- **`publication_in_progress` (409)** means another publish is mid flight. Wait a few seconds and retry once.
- **Anything the named commands do not cover**: `converly api GET /flows?limit=5` sends raw requests to the REST API.

## Quick reference

```
converly login [--signup]                # browser login, trial auto starts on signup
converly whoami                          # account, subscription, sites
converly sites list / update <id> --domain X
converly install snippet <site> / install status <site>
converly destinations list / connect <type> --site <id>
converly handoffs wait <hdf_id>          # block until human finishes OAuth
converly triggers                        # form tool slugs
converly destinations conversions <type> # pickable conversion actions
converly actions <type>                  # action config schema
converly flows create/validate/publish/unpublish/list/get/delete
converly test-event --flow <id>          # end to end proof
converly events list/get                 # the conversion log
converly rules create --email-pattern X  # exclude internal traffic
```

## Deeper docs

- [references/destinations.md](references/destinations.md) - per platform specifics: Google Ads conversion actions and Enhanced Conversions, Meta event names and CAPI, GA4, the browser side pixel destinations.
- [references/triggers.md](references/triggers.md) - the full form tool catalogue, filter options (specific form, specific pages), and how detection works.
- [references/rest-api.md](references/rest-api.md) - curl equivalents for every step, for environments where the CLI cannot be installed. Full API docs: https://developers.converly.io
