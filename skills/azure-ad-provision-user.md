---
name: azure-ad-provision-user
description: Create a user in Microsoft Entra ID, assign group membership and licences, and reverse the change safely if it was wrong.
api: azure-ad:azure-ad-users-api
generated: '2026-09-06'
method: generated
source: openapi/_original/azure-ad-graph-users-openapi.yml, openapi/_original/azure-ad-graph-groups-openapi.yml, conventions/azure-ad-conventions.yml
operations:
  - user_CreateUser
  - user_GetUser
  - user_ListUser
  - user_UpdateUser
  - user_DeleteUser
  - group_CreateMemberGraphBPreRef
  - group_DeleteMemberGraphBPreRef
  - directory_ListDeletedItem
  - directory.deletedItem_restore
scopes:
  - User.ReadWrite.All
  - GroupMember.ReadWrite.All
---

# Provision a user in Microsoft Entra ID

Base URL `https://graph.microsoft.com/v1.0`. Bearer token only; see
`authentication/azure-ad-authentication.yml`.

## Before you write anything

1. **Check for an existing account first.** There is no idempotency key on this
   API (`conventions/azure-ad-conventions.yml` → `idempotency.coverage: none`).
   A retried create after a network timeout produces a second user. Query first:
   `user_ListUser` with `$filter=userPrincipalName eq '<upn>'`. Add the header
   `ConsistencyLevel: eventual` if you use `$count` or `$search`.
2. **Confirm which permission set you hold.** Delegated tokens are additionally
   bounded by the signed-in user's own privileges; an application token is not.
   `User.ReadWrite.All` is admin-consented in both sets.

## Create

- `user_CreateUser` — `POST /users`. Required: `accountEnabled`,
  `displayName`, `mailNickname`, `userPrincipalName`, `passwordProfile`.
- Read the `id` (a GUID, not prefixed) out of the 201 body. Store it; the UPN is
  mutable and is not a durable key.

## Add to groups

- `group_CreateMemberGraphBPreRef` — `POST /groups/{group-id}/members/$ref` with
  body `{"@odata.id":"https://graph.microsoft.com/v1.0/directoryObjects/{user-id}"}`.
- Reverse with `group_DeleteMemberGraphBPreRef`.
- Group membership collections are polymorphic; when you read them back, switch
  on `@odata.type`, never assume users.

## Verify

- `user_GetUser` — `GET /users/{id}`. Use `$select` to name only the properties
  you need: it lowers the request's ResourceUnit cost by 1 as well as the payload.

## Undo

- `user_DeleteUser` — `DELETE /users/{id}` is a SOFT delete.
- Within **30 days**: `directory_ListDeletedItem` then
  `directory.deletedItem_restore` (`POST /directory/deletedItems/{id}/restore`).
- After 30 days, or after `directory_DeleteDeletedItem`, the object is gone
  permanently and cannot be restored. Treat `directory_DeleteDeletedItem` as
  terminal.

## Failure handling

- `429` — the Identity and Access service does **not** send `Retry-After`. Back
  off exponentially. See `rate-limits/azure-ad-rate-limits.yml`.
- `403` with `error=insufficient_claims` — a Conditional Access policy applies.
  Re-authenticate with the claims challenge; retrying the same token never works.
- `409` `Directory_ConcurrencyViolation` — retry with backoff.
- Branch on `error.code` only. `error.message` is unlocalised and unstable.
