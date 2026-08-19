---
name: on24-create-and-configure-webinar
description: Create or copy an ON24 webinar and configure its content, speakers, email notifications and calendar
  reminder.
api: ON24 Platform API (WCC API v2)
generated: '2026-08-13'
method: generated
source: Grounded in openapi/_original/on24-openapi.json (harvested from https://api.on24.com/v2/openapi.json) plus
  https://support.on24.com/hc/en-us/articles/21420786301083-REST-API-Integrations-Overview. Every operationId listed
  is verified present in that spec.
operations:
- Helpers.RetrieveEventTypes
- Helpers.RetrieveEventProfiles
- Helpers.RetrieveTimezonesCodes
- Helpers.RetrieveLanguageCodes
- Helpers.RetrieveEventManagers
- EventManagement.CreateAWebinar
- EventManagement.UpdateAWebinar
- EventManagement.UploadSlides
- EventManagement.SlideListing
- EventManagement.CreateSpeakerBio
- EventManagement.UpdateEmailNotification
- EventManagement.UpdateEmailCalendarReminder
- EventManagement.DeleteWebinar
---

# Create or copy an ON24 webinar and configure its content, speakers, email notifications and calendar reminder.

## When to use this

You are provisioning webinars programmatically instead of through the ON24 platform UI.

## Steps

1. **Resolve the reference data first.** These are cheap `Helpers` lookups and every one of them feeds a required
   field on create:
   - `GET /client/{clientId}/eventtypes` — `Helpers.RetrieveEventTypes`
   - `GET /client/{clientId}/eventprofile` — `Helpers.RetrieveEventProfiles`
   - `GET /client/{clientId}/timezones` — `Helpers.RetrieveTimezonesCodes`
   - `GET /client/{clientId}/languages` — `Helpers.RetrieveLanguageCodes`
   - `GET /client/{clientId}/eventmanager` — `Helpers.RetrieveEventManagers`

2. **Create or copy the webinar.**
   `POST /client/{clientId}/event` — operation `EventManagement.CreateAWebinar`.
   The request body is the `AddEventForm` schema. The same operation copies an existing event.
   **This operation is NOT idempotent and ON24 publishes no idempotency key.** A retried create makes a second
   webinar. Record the returned event id before you retry anything.

3. **Update it.** `PUT /client/{clientId}/event/{eventId}` — `EventManagement.UpdateAWebinar` (`UpdateEventForm`).

4. **Add content and people.**
   - `POST /client/{clientId}/event/{eventId}/slides` — `EventManagement.UploadSlides` (multipart)
   - `GET  /client/{clientId}/event/{eventId}/slides` — `EventManagement.SlideListing` (verify the upload landed)
   - `POST /client/{clientId}/event/{eventId}/speakerbio` — `EventManagement.CreateSpeakerBio`

5. **Configure communications.**
   - `PUT /client/{clientId}/event/{eventId}/email/{emailId}` — `EventManagement.UpdateEmailNotification`
   - `PUT /client/{clientId}/event/{eventId}/calendarreminder` — `EventManagement.UpdateEmailCalendarReminder`

6. **Tear down.** `DELETE /client/{clientId}/event/{eventId}` — `EventManagement.DeleteWebinar`.
   `410` means it was already deleted (treat as success). `412` means the event does not meet the deletion criteria.

## Media Manager

Assets that live at the account level rather than on one event go through
`EventManagement.MediaManagerUploadVideo` and `EventManagement.MediaManagerUploadDocument`
(`POST /client/{clientId}/mediamanager/uploadvideo` and `/mediamanager/document`). A `500` here means
"response code from uploading service is not success" — retry the upload, not the whole flow.


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
