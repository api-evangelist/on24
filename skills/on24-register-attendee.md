---
name: on24-register-attendee
description: Register a person for an ON24 webinar, honouring the event's configured registration fields, and update
  or soft-delete that registration afterwards.
api: ON24 Platform API (WCC API v2)
generated: '2026-08-13'
method: generated
source: Grounded in openapi/_original/on24-openapi.json (harvested from https://api.on24.com/v2/openapi.json) plus
  https://support.on24.com/hc/en-us/articles/21420786301083-REST-API-Integrations-Overview. Every operationId listed
  is verified present in that spec.
operations:
- Helpers.RetrieveRegistrationFields
- Registration.RESTRegistrationAPI
- Registration.UpdateRegistrantEventlevel
- Registration.UpdateRegistrantClientlevel
- AnalyticsClientLevel.RegistrantByEmail
---

# Register a person for an ON24 webinar, honouring the event's configured registration fields, and update or soft-delete that registration afterwards.

## When to use this

A person needs to be registered for a specific ON24 event from an external system (a form, a CRM, a marketing
automation platform), or an existing registration needs correcting.

## Steps

1. **Read the event's registration fields first.**
   `GET /client/{clientId}/event/{eventId}/regfield` — operation `Helpers.RetrieveRegistrationFields`.
   This returns the field configuration for THAT event. Fields marked required must be supplied or the
   registration is rejected. Do not assume a standard field set; it is configured per event.

2. **Create the registrant.**
   `POST /client/{clientId}/event/{eventId}/registrant` — operation `Registration.RESTRegistrationAPI`.
   - `content-type` **must** be `application/x-www-form-urlencoded`. This is a form post, not JSON.
   - Body is form parameters: `email`, `firstname`, `lastname`, `company`, plus any configured custom fields.
   - `honorrequired=N` makes required fields optional (including email). `honorvalidation=N` allows values
     outside a dropdown's configured options. Both default to `Y` — leave them alone unless you know why.
   - If passing a `country` parameter, the value must match one of the values on the registration page.

3. **Update an existing registration.**
   - Within one event: `PATCH /client/{clientId}/event/{eventId}/registrant` — `Registration.UpdateRegistrantEventlevel`.
   - Across the account: `PATCH /client/{clientId}/registrant/{email}` — `Registration.UpdateRegistrantClientlevel`.
   Both are keyed by **email**, which is the natural key for a person in ON24. There is no registrant id.

4. **Soft delete.** Send `isDeleted=Y` (or `N` to restore) as a form parameter on the update call.
   Deleting or undeleting is **not allowed after the event has started**.
   Do not attempt to create and soft-delete in the same request — that returns `412`.

5. **Verify.** `GET /client/{clientId}/registrant/{email}` — `AnalyticsClientLevel.RegistrantByEmail`.
   Note the lag: registrant data becomes available in roughly 15 minutes. An immediate read-back may miss.


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
