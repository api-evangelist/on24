---
name: on24-honor-erasure-request
description: Honour a GDPR/CCPA erasure ("forget me") request in ON24 at either event or account scope, and verify
  the outcome.
api: ON24 Platform API (WCC API v2)
generated: '2026-08-13'
method: generated
source: Grounded in openapi/_original/on24-openapi.json (harvested from https://api.on24.com/v2/openapi.json) plus
  https://support.on24.com/hc/en-us/articles/21420786301083-REST-API-Integrations-Overview. Every operationId listed
  is verified present in that spec.
operations:
- AnalyticsClientLevel.RegistrantByEmail
- AnalyticsClientLevel.RegistrantsByEmailMultipleEvents
- Registration.ForgetRegistrantClientlevel
- Registration.ForgetAllRegistrantClientlevel
- Registration.ForgetAllRegistrantEventlevel
---

# Honour a GDPR/CCPA erasure ("forget me") request in ON24 at either event or account scope, and verify the outcome.

## When to use this

A data subject has asked to be erased and their engagement data sits in ON24. ON24 is a first-party engagement
data store, so it is squarely in scope for a DSAR.

## Understand the three scopes before you call anything

ON24 exposes three different forget operations and they are **not** interchangeable:

| Operation | Path | Scope |
|---|---|---|
| `Registration.ForgetRegistrantClientlevel` | `POST /client/{clientId}/forget` | One person, whole account |
| `Registration.ForgetAllRegistrantClientlevel` | `POST /client/{clientId}/forgetall` | Bulk, whole account |
| `Registration.ForgetAllRegistrantEventlevel` | `POST /client/{clientId}/event/{eventId}/forgetall` | Bulk, one event only |

Choose the narrowest scope that satisfies the request. An account-level forget cannot be undone by an event-level call.

## Steps

1. **Find everything about the subject first, while you still can.**
   - `GET /client/{clientId}/registrant/{email}` — `AnalyticsClientLevel.RegistrantByEmail`
   - `GET /client/{clientId}/registrant/{email}/allevents` — `AnalyticsClientLevel.RegistrantsByEmailMultipleEvents`
   This is also what you use to answer the *access* half of a DSAR. Capture it before erasing.

2. **Erase at the chosen scope** using the table above. The subject is identified by **email** — there is no
   opaque registrant id in this API.

3. **Respect the bulk ceiling.** The forget-all operations reject more than **100,000** users per request with
   `400 "You have exceeded the maximum number of user to be deleted (100,000)"`. Chunk larger populations.

4. **Verify.** Re-read with `userstatus=forgotten` on the registrant/attendee analytics operations — forgotten
   users are returned under that filter. Note that `filterforgotten` is **deprecated**; use `userstatus` instead.

5. **Record it.** ON24 returns no receipt object and no erasure audit endpoint. Log the request, the scope, the
   operationId, the timestamp and the HTTP response yourself — that log is your only evidence of compliance.

## Caution

Forget is destructive and there is no idempotency key. It is, however, naturally idempotent in effect: forgetting
an already-forgotten subject is not harmful. Erasing the wrong scope is.


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
