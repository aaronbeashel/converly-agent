# Trigger specifics

The trigger is what Converly's loader watches for on the user's website. Get the live catalogue with `converly triggers`; use a slug from `providers[]` as the flow's trigger.

## How detection works

The Converly snippet loads a per site config and watches the page for the configured form tool's submission signals (native events, embedded iframe messages, DOM signals, depending on the tool). No code changes to the form itself, no webhooks to set up for the browser detected tools. That means:

- The snippet MUST be on the page where the form lives.
- The flow must be PUBLISHED before anything is captured.
- Detection is per tool. Picking the right provider slug matters. `generic-form` covers hand coded `<form>` elements.

## Picking the right slug

Match what actually renders the form, not the brand of the website builder:

- A Typeform embedded on a Webflow site is `typeform`, not `webflow-forms`.
- A HubSpot form embedded anywhere is `hubspot-forms`.
- Webflow's own native form blocks are `webflow-forms`.
- A React app's hand rolled form is `generic-form`.

If you have repo or page access, inspect the form markup and decide yourself. Embed scripts (`js.hsforms.net`, `embed.typeform.com`) are the giveaway.

## Trigger types

- `form_submission` - about 80 providers: Gravity Forms, Contact Form 7, WPForms, Elementor, Typeform, Jotform, Webflow, Wix, Squarespace, Framer, Unbounce, HubSpot, Mailchimp, Marketo, Tally, Fillout, ClickFunnels, and more. No filter is required. A flow with no `--pages` fires for every form of that type anywhere on the site, so offer page narrowing when the site has more than one form.
- `chat_started` - LiveChat, Intercom, Drift, Tawk.to, HubSpot Chat and similar. Fires when a visitor starts a conversation.
- `meeting_booked` - Calendly, Acuity, Cal.com, OnceHub, SavvyCal style booking tools.
- Connection-required providers. Read each provider's `setup` block in the catalogue. `requires_connection: true` (Typeform, Jotform, Calendly, Acuity today) means the tool must be connected with `converly triggers connect <slug> --site <id>` BEFORE its flow properly works. `capture_without_connection` tells you honestly what happens if the user declines (for example count-only capture with no visitor details). `post_connect_user_steps` lists steps the USER must still do outside Converly (Acuity's pasted snippet); Converly cannot verify those, so setup is only proven when the first event arrives.
- `custom_event` - cannot be set up through this surface. Never offer it, and never improvise a custom-event workaround for an unsupported tool. If no listed provider fits, say so plainly.

## Filters

The simple CLI form defaults to all pages (`pageFilter: "all"`), and that is a valid, publishable flow. Narrowing options, honestly stated:

- **By page** (works for every browser-detected tool): `--pages /contact,/demo`. This is the supported narrowing on this surface. When the site has more than one form, ask which page the form the user cares about lives on and narrow to it, otherwise a contact form and a newsletter box report as the same conversion.
- **By specific form or event type** (connection-required providers only): the connected platforms support narrowing to a real form or event type, but listing the account's actual forms is not available through the CLI yet. Create the flow unfiltered and tell the user they can narrow it to a specific form in the Converly dashboard's flow builder. Do not invent condition values.

To adjust an existing flow, pass just the changed field to `converly flows update <flow_id> --json '{"trigger_config": {...}}'`.

## When nothing fires

Work through, in order:

1. `converly install status <site>`. Read `detection` precisely: `"confirmed"` means the site half works, look further down this list. `"never_seen"` is NOT proof the snippet is missing (a correct install with no conversions yet looks identical), it just means nothing is proven yet. `"pending"` means the check itself couldn't run right now, try again shortly. Only conclude "snippet missing" from actually inspecting the page's HTML, never from this endpoint.
2. Site domain set and matching the real origin? `converly sites list` → `domain`. A staging URL or a different subdomain will be rejected.
3. Flow published? `converly flows get <id>` → `status` must be `published`, `publication_state` should be `in_sync`.
4. Right provider slug for what actually renders the form? (See "Picking the right slug".)
5. Internal traffic rule swallowing your tests? `converly rules list`.
6. Still nothing: `converly events list --status failed` and `converly events get` on anything that appears; the event detail carries per destination errors and user facing notices.
