---
name: bigtincan-publish-story
description: Publish a new sales enablement story into a Bigtincan Hub channel, attach files and tags, and verify it landed.
api: Bigtincan Hub API
base_url: https://pubapi.bigtincan.com
spec: openapi/bigtincan-hub-api-openapi.json
generated: '2026-08-14'
method: generated
source: openapi/bigtincan-hub-api-openapi.json + https://pubapi.bigtincan.com/doc/interactive/
operations:
  - get-v1-user-me
  - get-v1-channel-all
  - post-v1-story-upload-file
  - post-v1-2-story-add
  - post-v1-story-add-tag-by-story-perm-id
  - get-v1-2-story-all
---

# Publish a story to Bigtincan Hub

A story is Bigtincan's unit of publishable sales content. It must belong to at
least one channel, and it may carry files, events and tags.

## Before you start

Get a token. The API is OAuth 2.0 only.

```
POST https://pubapi.bigtincan.com/services/oauth2/token
grant_type=password&client_id=...&client_secret=...&api_key=...
```

Send `Authorization: Bearer <access_token>` on every call below. If you are
acting for a specific tenant user, add `As-User: <USER_ID>` — this works only
with the password grant, never with the authorization code grant.

## Steps

1. **Confirm who you are acting as** — `get-v1-user-me`
   `GET /v1/user/me`. If you set `As-User`, confirm the response is the intended
   user before writing anything.

2. **Find the target channel** — `get-v1-channel-all`
   `GET /v1/channel/all?page=1&limit=100`. `limit` caps at 100, so page through
   for large tenants. Capture the channel `id`. A story with no channel cannot be
   created — `channels` is a required field.

3. **Upload any files first** — `post-v1-story-upload-file`
   `POST /v1/story/upload/file` as `multipart/form-data`. Check
   `get-v1-file-allowed-extensions` (`GET /v1/file/allowed-extensions`) first if
   the file type is unusual; an unsupported extension comes back as a 422.

4. **Publish the story** — `post-v1-2-story-add`
   `POST /v1.2/story/add`. Note the version prefix is **/v1.2**, not /v1.
   Required: `name` and `channels`. `channels` is an array of `{id, is_alias}`.
   Optional: `title`, `description`, `excerpt`, `new_files[]`, `events[]`,
   `tags[]`. Each entry in `new_files[]` requires `filename`, `description` and
   `share_status` (`mandatory`, `optional` or `blocked`).

5. **Tag it, if tags were not inlined** — `post-v1-story-add-tag-by-story-perm-id`
   `POST /v1/story/add/tag/{story_perm_id}`. Use the *permanent* id here, not the
   revision id. Read the available vocabulary from `GET /v1/tags` first.

6. **Verify** — `get-v1-2-story-all`
   `GET /v1.2/story/all?page=1&limit=10` and confirm the story is present.

## Rules that will bite you

- **There is no idempotency.** `POST /v1.2/story/add` has no Idempotency-Key.
  If it times out or returns a 5xx, do **not** re-POST. Run step 6 first and only
  retry if the story is genuinely absent, or you will publish a duplicate.
- **Permanent id vs revision id.** Reads and tagging use `story_perm_id`; edit,
  archive, comment and share use `revision_id`. They are not interchangeable.
- **Version prefixes differ per operation.** Creating a story is /v1.2, reading
  one is /v1.1, listing channels is /v1. Copy the prefix from the operation, never
  assume /v1.
- **Responses are untyped.** No operation in the contract declares a response
  schema, so parse defensively and do not assume a field exists.
- **Errors** come back as
  `{"error":{"scope","code","message"},"trace_id"}` with `application/json` —
  not RFC 9457 problem+json. Capture `trace_id` from the body (it is not a
  header) when raising a support ticket.
- **403 is a role decision, not a scope decision.** Bigtincan publishes no OAuth
  scopes; permission comes from the Hub role of the token owner or the As-User
  target. You cannot pre-flight it from the token.
- **No rate-limit headers.** Nothing tells you how close you are to a limit.
  Back off on your own schedule.
