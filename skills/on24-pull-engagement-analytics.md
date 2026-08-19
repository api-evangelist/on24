---
name: on24-pull-engagement-analytics
description: Pull ON24 event, registrant, attendee and in-event engagement analytics into a CRM, marketing automation
  platform or data warehouse.
api: ON24 Platform API (WCC API v2)
generated: '2026-08-13'
method: generated
source: Grounded in openapi/_original/on24-openapi.json (harvested from https://api.on24.com/v2/openapi.json) plus
  https://support.on24.com/hc/en-us/articles/21420786301083-REST-API-Integrations-Overview. Every operationId listed
  is verified present in that spec.
operations:
- AnalyticsClientLevel.Events
- AnalyticsEventLevel.MetadataAndUsage
- AnalyticsEventLevel.Registrants
- AnalyticsEventLevel.Attendees
- AnalyticsEventLevel.AttendeeViewingSessions
- AnalyticsEventLevel.Polls
- AnalyticsEventLevel.Surveys
- AnalyticsEventLevel.CallToAction
- AnalyticsEventLevel.ResourcesViewed
- AnalyticsClientLevel.PEP
- AnalyticsClientLevel.EngagedAccounts
---

# Pull ON24 event, registrant, attendee and in-event engagement analytics into a CRM, marketing automation platform or data warehouse.

## When to use this

You are syncing ON24 webinar engagement into a downstream system on a schedule.

## Respect the data availability lag before you schedule anything

ON24 publishes how long data takes to land in the warehouse the API reads from:

| Data | Available after |
|---|---|
| Registrant | ~15 minutes |
| Attended — live | 30 minutes to 2 hours after the event ends |
| Attended — on demand | 4 to 12 hours |

Polling an event the minute it ends returns incomplete attendee data. Schedule the attendee pull for at least
2 hours after the end time, and re-pull on-demand activity daily.

## Steps

1. **List events in the window.**
   `GET /client/{clientId}/event` — operation `AnalyticsClientLevel.Events`.
   Filter with `startdate` / `enddate` and `datefiltermode`. Use `partnerref` to filter to your own integration's events.
   - **Pagination is conditional here.** With `includesubaccounts=Y`, pagination becomes REQUIRED
     (`pageoffset` + `itemsperpage`) **and the response schema changes**. Branch your parser on that flag.
   - The response carries `totalevents`; compute page count yourself. There is no cursor and no `Link` header.

2. **Per event, get metadata and usage.**
   `GET /client/{clientId}/event/{eventId}` — `AnalyticsEventLevel.MetadataAndUsage`.

3. **Pull the people.**
   - `GET /client/{clientId}/event/{eventId}/registrant` — `AnalyticsEventLevel.Registrants`
   - `GET /client/{clientId}/event/{eventId}/attendee` — `AnalyticsEventLevel.Attendees`
   - `GET /client/{clientId}/event/{eventId}/attendeesession` — `AnalyticsEventLevel.AttendeeViewingSessions`
     (per-session live/archive minutes — the granular view behind the averages)
   Useful filters: `userstatus` (`any` / `deleted` / `forgotten`; `all` is **deprecated**), `excludeanonymous`,
   `excludelive`, `noshow`. Note ON24 states there is **no** option to exclude soft-deleted users from the response.

4. **Pull in-event engagement.**
   `AnalyticsEventLevel.Polls`, `AnalyticsEventLevel.Surveys`, `AnalyticsEventLevel.CallToAction`,
   `AnalyticsEventLevel.ResourcesViewed` — each `GET /client/{clientId}/event/{eventId}/{poll|survey|cta|resource}`.

5. **Roll up to the person and the account.**
   - `GET /client/{clientId}/lead/{email}` — `AnalyticsClientLevel.PEP` (Prospect Engagement Profile)
   - `GET /client/{clientId}/engagedaccount` — `AnalyticsClientLevel.EngagedAccounts` (ABM view)

## Shape warning

**No 200 response in this API declares a schema.** All 67 responses carry examples only. Write your parser
defensively against `examples/` in this repo and the examples in `openapi/_original/on24-openapi.json` — a
generated typed client will be guessing.


## Preconditions (identical for every ON24 skill)

- **Base URL** — `https://api.on24.com/v2` (North America) or `https://api.eu.on24.com/v2` (European Union).
  Pick the host matching the customer's data centre; they are separate deployments serving an identical contract.
  The published OpenAPI declares only a relative server (`/wcc/api/v2`) and names no host — do not use it as written.
- **Auth** — send BOTH headers on every call: `accesstokenkey` and `accesstokensecret`.
  Sending only one returns `401 {"message":"Header must contain both accessTokenKey and accessTokenSecret."}`.
- **Scope** — every path begins `/client/{clientId}`, a numeric account id. You must know it before you start.
- **Server-side only** — ON24 requires these calls be made server to server, never from browser JavaScript.
- **Entitlement** — the account contract must include *Connect* for a token to exist at all, and several
  operations additionally require the *Elite* tier (`403 "The client id is not an Elite user"`).

## Error handling (identical for every ON24 skill)

Errors are a plain JSON object with one field: `{"message": "..."}`. There is no RFC 9457 problem type and no error code.

| Status | Meaning | What to do |
|---|---|---|
| 401 | Headers missing, or token wrong/invalid/deactivated | Stop. Do not retry. Re-provision the token. |
| 403 | **Ambiguous** — permission denied, client inactive, non-Elite, **or quota exhausted** | Read `message`. Only back off if it contains "maximum number of calls". |
| 404 | Client, event or registrant does not exist | Stop; verify the ids. |
| 410 | Event already deleted | Terminal success for a delete flow. |
| 412 | Precondition failed | Do not retry as-is; the request violates a state rule. |

**The 403 trap.** ON24 signals rate-limit exhaustion as **403, not 429**, and returns no `Retry-After` or
`RateLimit-*` header. You cannot distinguish quota from permission by status code — you must string-match
`message` for "You have reached the maximum number of calls" / "You reached the maximum number of calls".
Back-off must be blind (exponential, jittered); there is no published limit and no runtime signal.

**No idempotency keys.** ON24 publishes no `Idempotency-Key` header. Registration is upsert-like by email, so
retrying a registration is comparatively safe; retrying `EventManagement.CreateAWebinar`, `UploadSlides` or a
Media Manager upload is **not** — it will create duplicates. Guard those with your own dedupe before retrying.
