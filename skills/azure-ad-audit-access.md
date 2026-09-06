---
name: azure-ad-audit-access
description: Read who has access to what in a Microsoft Entra tenant — group membership, directory roles, app role assignments and Conditional Access policies — without writing anything and without being throttled.
api: azure-ad:azure-ad-directory-api
generated: '2026-09-06'
method: generated
source: openapi/_original/azure-ad-graph-identity-directorymanagement-openapi.yml, openapi/_original/azure-ad-graph-identity-signins-openapi.yml, rate-limits/azure-ad-rate-limits.yml
operations:
  - user_ListUser
  - user_ListMemberGraphOPre
  - group_ListGroup
  - group_ListMember
  - servicePrincipal_ListAppRoleAssignedTo
  - policy_ListConditionalAccessPolicy
  - directory_ListDeletedItem
scopes:
  - Directory.Read.All
  - Policy.Read.All
  - AuditLog.Read.All
---

# Audit access in a Microsoft Entra tenant

This is a **read-only** skill. It touches no write operation, so nothing here
needs a reversal path.

## Budget before you enumerate

Throttling is metered in ResourceUnits per application+tenant pair, and a large
tenant gets 8,000 units per 10 seconds. Cost is shaped by your query:

- `$select` **subtracts** 1 unit — always name the properties you want.
- `$expand` **adds** 1 unit.
- `$top` under 20 subtracts 1.
- `GET /groups/{id}/transitiveMembers` costs **5** units per call. Use it
  deliberately, not in a loop over every group.

On `429`, the Identity and Access service sends **no** `Retry-After`. Back off
exponentially; do not wait on a header that will not arrive.

## Enumerate

1. `user_ListUser` — `GET /users?$select=id,userPrincipalName,accountEnabled`.
   Follow `@odata.nextLink` verbatim until it is absent; do not rewrite it.
2. `user_ListMemberGraphOPre` — `GET /users/{id}/memberOf` returns groups,
   directory roles and administrative units in one polymorphic collection.
   Switch on `@odata.type`.
3. `group_ListMember` — `GET /groups/{id}/members` is likewise polymorphic
   (users, groups, devices, service principals). The `...AsUser` /
   `...AsServicePrincipal` variants exist if you want one type only.
4. `servicePrincipal_ListAppRoleAssignedTo` — which principals hold which app
   role on a resource. This is where non-human access lives, and it is the part
   most access reviews miss.
5. `policy_ListConditionalAccessPolicy` — the policies that can override
   everything above at sign-in time.
6. `directory_ListDeletedItem` — objects deleted in the last 30 days that are
   still restorable, and therefore still part of the tenant's risk surface.

## Prefer change tracking over polling

Repeatedly re-enumerating is the documented route to being throttled. Use delta
query (`https://learn.microsoft.com/en-us/graph/delta-query-overview`) or a
change-notification subscription (`asyncapi/azure-ad-change-notifications-webhooks.yml`)
instead of a polling loop.

## Note on completeness

`policy_ListConditionalAccessPolicy` and directory roles are **not** soft-deleted.
If one is removed there is no restore path, so an audit snapshot is the only
record you will have.
