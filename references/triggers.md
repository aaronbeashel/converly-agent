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

- `form_submission` - about 80 providers: Gravity Forms, Contact Form 7, WPForms, Elementor, Typeform, Jotform, Webflow, Wix, Squarespace, Framer, Unbounce, HubSpot, Mailchimp, Marketo, Tally, Fillout, ClickFunnels, and more. Requires at least one filter (see below); "all forms on all pages" plus one filter type is the norm.
- `chat_started` - LiveChat, Intercom, Drift, Tawk.to, HubSpot Chat and similar. Fires when a visitor starts a conversation.
- `meeting_booked` - Calendly, Acuity, Cal.com, OnceHub, SavvyCal style booking tools. Some of these can additionally be connected platform side in the Converly dashboard for richer data (attendee details); the browser detection works without it.
- `custom_event` - no provider. The site's own JavaScript calls Converly's API to report a conversion. See the install page in the dashboard for the call signature.

## Filters

The simple CLI form defaults to all pages (`pageFilter: "all"`). Narrow with:

- `--pages /contact,/demo` for specific pages.
- A specific form id or CSS selector goes in the trigger config via `--json`, for example:

```json
{
  "trigger_config": {
    "integrationId": "gravity-forms",
    "pageFilter": "all",
    "formId": "5"
  }
}
```

Run `converly triggers` and read `filters[]` for the trigger type to see what each type supports (`form_id`, `form_selector`, `page_path_filter`, `event_name`).

## When nothing fires

Work through, in order:

1. `converly install status <site>` shows the loader was seen? If not, the snippet is not on the page (or is blocked by a consent manager until the visitor accepts).
2. Site domain set and matching the real origin? `converly sites list` → `domain`. A staging URL or a different subdomain will be rejected.
3. Flow published? `converly flows get <id>` → `status` must be `published`, `publication_state` should be `in_sync`.
4. Right provider slug for what actually renders the form? (See "Picking the right slug".)
5. Internal traffic rule swallowing your tests? `converly rules list`.
6. Still nothing: `converly events list --status failed` and `converly events get` on anything that appears; the event detail carries per destination errors and user facing notices.
