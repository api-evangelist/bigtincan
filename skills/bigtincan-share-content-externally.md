---
name: bigtincan-share-content-externally
description: Share Bigtincan content outside the tenant — email a story to buyers, or create an expiring public file share — with the confirmation gates these irreversible actions require.
api: Bigtincan Hub API
base_url: https://pubapi.bigtincan.com
spec: openapi/bigtincan-hub-api-openapi.json
generated: '2026-08-14'
method: generated
source: openapi/bigtincan-hub-api-openapi.json + https://pubapi.bigtincan.com/doc/interactive/
operations:
  - get-v1-user-me
  - get-v1-2-story-all
  - post-v1-story-share-by-revision-id
  - post-v1-shares
  - put-v1-shares-by-share-id-files-by-file-id
  - get-v1-shares-by-share-id
  - get-v1-shares
  - delete-v1-shares-by-share-id
  - delete-v1-shares-by-share-id-files-by-file-id
---

# Share Bigtincan content externally

Two distinct surfaces leave the tenant:

- **Story share** sends email to named recipients.
- **Public file share** publishes files to an unauthenticated URL with an expiry.

Both have effects that cannot be recalled by the API. Treat each as a
human-confirmation gate.

## Before you start

Authenticate as in `bigtincan-publish-story`. `As-User` works here (password
grant only) and determines who the recipient sees as the sender.

## Path A — email a story to buyers

1. **Identify the story revision** — `get-v1-2-story-all`
   `GET /v1.2/story/all`. This operation needs the **revision id**, not the
   permanent id.

2. **Confirm with a human.** Enumerate the exact recipient addresses and the
   subject back to the user. This step sends real email.

3. **Share** — `post-v1-story-share-by-revision-id`
   `POST /v1/story/share/{revision_id}`. Required: `subject` and `emails[]`.
   Optional: `note`, `cc_emails[]`, `lang_code`, `file_id[]` to restrict the
   share to specific files.

4. **Do not retry on ambiguity.** There is no idempotency key. A retry after a
   timeout can send the email twice, and there is no unsend.

## Path B — create an expiring public file share

1. **Confirm with a human.** The resulting URL is reachable without
   authentication by anyone who has it.

2. **Create the share** — `post-v1-shares`
   `POST /v1/shares`. Required: `allow_downloads` (boolean). Strongly
   recommended: `expires_at`, an RFC 3339 timestamp
   (`2019-07-17T06:02:43+00:00` is the format in the contract). **Omitting
   `expires_at` creates a share with no stated expiry** — always set one unless
   the user explicitly asks otherwise.

3. **Add files** — `put-v1-shares-by-share-id-files-by-file-id`
   `PUT /v1/shares/{share_id}/files/{file_id}`, once per file. This is a PUT and
   is naturally idempotent, so it is safe to retry.

4. **Read it back** — `get-v1-shares-by-share-id`
   `GET /v1/shares/{share_id}` and report the contents and expiry to the user.

## Cleaning up

- `delete-v1-shares-by-share-id-files-by-file-id` —
  `DELETE /v1/shares/{share_id}/files/{file_id}` removes one file.
- `delete-v1-shares-by-share-id` — `DELETE /v1/shares/{share_id}` revokes the
  whole share. This is the only way to withdraw published content.
- `get-v1-shares` — `GET /v1/shares` lists the principal's shares. Run this
  before creating a new one to avoid duplicating an existing share.

## Rules that will bite you

- **Nothing here is idempotent on create.** POST /v1/story/share and
  POST /v1/shares have no dedupe key. Read back before retrying.
- **Email cannot be recalled.** Revoking a share stops future access to the
  files; it does not unsend the message.
- **Errors** are `{"error":{"scope","code","message"},"trace_id"}`, not RFC 9457.
- **A 403 means the Hub role lacks the permission**, not that a scope is missing —
  there are no OAuth scopes to widen.
