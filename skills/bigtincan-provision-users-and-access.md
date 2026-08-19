---
name: bigtincan-provision-users-and-access
description: Onboard a user into Bigtincan Hub and grant them content access through groups, channels and tabs, using the correct version prefix for each admin operation.
api: Bigtincan Hub API
base_url: https://pubapi.bigtincan.com
spec: openapi/bigtincan-hub-api-openapi.json
generated: '2026-08-14'
method: generated
source: openapi/bigtincan-hub-api-openapi.json + https://pubapi.bigtincan.com/doc/interactive/
operations:
  - get-v1-3-admin-user-all
  - post-v1-1-user
  - patch-v1-1-user
  - get-v1-1-admin-user-get-by-user-id
  - get-v1-1-admin-group-all
  - post-v1-group
  - put-v1-group-by-group-id-users
  - get-v1-admin-tabs
  - put-v1-tabs-by-tab-id-channels-by-channel-id-groups
  - delete-v1-admin-groups-by-group-id-users-by-user-id
  - delete-v1-user-by-user-id
---

# Provision a Bigtincan Hub user and grant them access

Access in Bigtincan flows **user → group → (channel, tab)**. You do not grant a
user access to content directly; you put the user in a group and grant the group.

## Before you start

Authenticate with the password grant. **`As-User` is not honoured by admin
operations** — every step below runs as the token owner, who must hold an
administrator role.

## Steps

1. **Check the user does not already exist** — `get-v1-3-admin-user-all`
   `GET /v1.3/admin/user/all?page=1&limit=100`. Note the **/v1.3** prefix. There
   is no idempotency key on user creation, so this read is the only thing
   standing between you and a duplicate account.

2. **Create the user** — `post-v1-1-user`
   `POST /v1.1/user`. Note the **/v1.1** prefix — different from step 1, for the
   same resource. Required: `first_name`, `last_name`, `email`. Useful optional
   fields:
   - `access` — `0` User, `2` Structure Admin, `4` Administrator.
   - `send_invite` — boolean; sends the invitation email.
   - `digest_email` — `0` Never, `1` Daily, `2` Weekly, `3` Monthly.
   - `configuration_bundle` — configuration bundle id.

3. **Read the user back** — `get-v1-1-admin-user-get-by-user-id`
   `GET /v1.1/admin/user/get/{user_id}`. Capture the id.

4. **Pick or create the group** — `get-v1-1-admin-group-all` /
   `post-v1-group`
   `GET /v1.1/admin/group/all` to list, `POST /v1/group` to create. Again note
   the prefixes differ between the read and the write.

5. **Add the user to the group** — `put-v1-group-by-group-id-users`
   `PUT /v1/group/{group_id}/users`. This is a PUT and safe to retry.

6. **Grant the group access to content** —
   `put-v1-tabs-by-tab-id-channels-by-channel-id-groups`
   `PUT /v1/tabs/{tab_id}/channels/{channel_id}/groups` with an array of
   `{id, permissions}`. `permissions` is an integer mask: **1 read, 2 write,
   3 read and write**. Use `get-v1-admin-tabs` (`GET /v1/admin/tabs`) to find the
   tab. Grant the narrowest mask that works — there are no OAuth scopes, so this
   mask and the user's `access` level are the entire authorization story.

## Deprovisioning

- `delete-v1-admin-groups-by-group-id-users-by-user-id` —
  `DELETE /v1/admin/groups/{group_id}/users/{user_id}` removes a user from one
  group. Prefer this for a role change.
- `delete-v1-user-by-user-id` — `DELETE /v1/user/{user_id}` deletes the user.
  Irreversible through the API. Confirm with a human first.

## Rules that will bite you

- **The version prefix changes between operations on the same resource.** Users
  are listed at /v1.3, created and updated at /v1.1, deleted at /v1. Groups are
  listed at /v1.1/admin, created at /v1, modified at /v1/admin. Copy the prefix
  from the operation every time.
- **Admin operations ignore As-User.** Anything under an `Admin` tag runs as the
  token owner regardless of the header.
- **No idempotency on `POST /v1.1/user`.** Always list first (step 1); never
  blind-retry a create.
- **Password operations are separate**: `patch-v1-user-change-password`
  (`PATCH /v1/user/change-password`) changes the current user's password, and
  `patch-v1-user-forgot-password` (`PATCH /v1/user/forgot-password`) sends a
  reset email. Neither is an admin reset of another user's password.
- **Errors** are `{"error":{"scope","code","message"},"trace_id"}` with
  `application/json`, not RFC 9457. A 401 carries
  `code: INVALID_TOKEN`; refresh at `/services/oauth2/token` with
  `grant_type=refresh_token`.
