# REST API fallback (no CLI)

For environments where npm is unavailable. Everything the CLI does is a `/v1` REST call. Full docs: https://developers.converly.io

Base URL: `https://app.converly.io/api/v1`

Auth: `Authorization: Bearer $CONVERLY_API_KEY` on every request. Without the CLI there is no browser login, so this path requires the key to already exist in the environment. Never echo it; use the env var in the command.

The rules from SKILL.md still apply, especially: a human completes the destination connect URL, and success is only reported after a test event succeeds.

```bash
BASE="https://app.converly.io/api/v1"
AUTH="Authorization: Bearer $CONVERLY_API_KEY"

# 1. Sites
curl -sS "$BASE/sites" -H "$AUTH"
curl -sS -X PATCH "$BASE/sites/{site_id}" -H "$AUTH" \
  -H "Content-Type: application/json" -d '{"domain":"example.com"}'

# 2. Install snippet + status
curl -sS "$BASE/sites/{site_id}/install-snippet" -H "$AUTH"
curl -sS "$BASE/sites/{site_id}/install-status" -H "$AUTH"

# 3. Connect an ad platform (human opens `url`, then poll)
curl -sS -X POST "$BASE/handoffs" -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"purpose":"connect_destination","destination_type":"google-ads","site_id":"{site_id}"}'
curl -sS "$BASE/handoffs/{handoff_id}" -H "$AUTH"   # until status == "completed"

# 4. Catalogues (public, no auth needed)
curl -sS "$BASE/trigger-types"
curl -sS "$BASE/destination-types"
curl -sS "$BASE/action-types?destination_type=google-ads"
curl -sS "$BASE/destinations/dest_google-ads/conversions" -H "$AUTH"

# 5. Create, validate, publish a flow
curl -sS -X POST "$BASE/flows" -H "$AUTH" -H "Content-Type: application/json" -d '{
  "site_id": "{site_id}",
  "name": "Demo requests",
  "trigger_config": { "integrationId": "generic-form", "pageFilter": "all" },
  "actions_config": [{
    "id": "act-1",
    "integrationId": "google-ads",
    "config": { "conversion": { "id": "123456789" }, "enhancedConversions": true }
  }]
}'
curl -sS -X POST "$BASE/flows/{flow_id}/validate" -H "$AUTH"
curl -sS -X POST "$BASE/flows/{flow_id}/publish" -H "$AUTH"

# 6. Verify end to end
curl -sS -X POST "$BASE/test-event" -H "$AUTH" -H "Content-Type: application/json" -d '{
  "flow_id": "{flow_id}",
  "action": { "action_id": "act-1", "integration_id": "google-ads",
              "config": { "conversion": { "id": "123456789" } } }
}'

# 7. Read conversions
curl -sS "$BASE/events?limit=20" -H "$AUTH"
curl -sS "$BASE/events/{event_id}" -H "$AUTH"
```

Notes that differ from what you might assume:

- IDs are prefixed (`site_`, `flow_`, `hdf_`, `evt_`). Use them verbatim in paths.
- Every POST accepts an `Idempotency-Key` header; use one UUID per logical operation so retries are safe.
- `meta-ads` is rejected as a slug; the Meta slug is `meta`.
- `value`/`currency` nest INSIDE the `conversion` object; the enhanced conversions toggle is top level `enhancedConversions`.
- Events list is a bounded snapshot (limit max 100, no cursors). Narrow with `occurred_after`, `flow_id`, `email`, `status`.
