---
name: azure-ad-manage-app-registration
description: Register an application in Microsoft Entra ID, create and rotate its credentials, and grant it an app role — without stranding a secret or an orphaned service principal.
api: azure-ad:azure-ad-applications-api
generated: '2026-09-06'
method: generated
source: openapi/_original/azure-ad-graph-applications-openapi.yml, scopes/azure-ad-scopes.yml
operations:
  - application_CreateApplication
  - application_GetApplication
  - application_ListApplication
  - application_UpdateApplication
  - application_DeleteApplication
  - application_addPassword
  - application_removePassword
  - application_addKey
  - application_removeKey
  - servicePrincipal_CreateServicePrincipal
  - servicePrincipal_ListServicePrincipal
  - servicePrincipal_CreateAppRoleAssignedTo
  - servicePrincipal_DeleteAppRoleAssignedTo
  - directory.deletedItem_restore
scopes:
  - Application.ReadWrite.All
  - AppRoleAssignment.ReadWrite.All
---

# Manage an application registration

## The two-object trap

An `application` and a `servicePrincipal` are two different directory objects for
one logical app, and both have their own `id`. The application also carries an
`appId` which is **not** its object id. Getting these confused is the single most
common integration bug on this surface — see
`data-model/azure-ad-data-model.yml`.

1. `application_CreateApplication` — `POST /applications`. Returns both `id`
   (object id) and `appId` (client id).
2. `servicePrincipal_CreateServicePrincipal` — `POST /servicePrincipals` with
   `{"appId":"<appId from step 1>"}`. Without this the app exists but cannot be
   assigned anything in this tenant.

## Credentials

- `application_addPassword` — `POST /applications/{id}/addPassword`. The secret
  value is returned **once, in this response only**. There is no way to read it
  back. Capture it or you must rotate.
- `application_addKey` / `application_removeKey` — certificate credentials.
  Prefer these (or a federated identity credential) over a shared secret for any
  unattended workload; the token endpoint advertises `private_key_jwt`.
- Rotate by adding the new credential, cutting traffic over, then
  `application_removePassword`. Never remove first.

## Grant an app role

- `servicePrincipal_CreateAppRoleAssignedTo` — `POST
  /servicePrincipals/{resource-sp-id}/appRoleAssignedTo` binding a principal
  (user, group or service principal) to an `appRoleId` published by the resource.
- Reverse with `servicePrincipal_DeleteAppRoleAssignedTo`.

## Undo

- `application_DeleteApplication` soft-deletes; restore within **30 days** with
  `directory.deletedItem_restore`. The same applies to a deleted service
  principal.
- Deleting an application does **not** delete its service principal, and vice
  versa. Clean up both, and check `servicePrincipal_ListServicePrincipal` with
  `$filter=appId eq '<appId>'` afterwards.

## Failure handling

Same as every Graph call: `error.code` only, exponential backoff on 429 (no
`Retry-After` here), claims challenge on `403 insufficient_claims`.
