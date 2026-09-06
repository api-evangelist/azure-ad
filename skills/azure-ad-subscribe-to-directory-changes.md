---
name: azure-ad-subscribe-to-directory-changes
description: Subscribe to created/updated/deleted change notifications for Entra ID users and groups, keep the subscription alive, and interpret the events correctly.
api: azure-ad:azure-ad-change-notifications-api
generated: '2026-09-06'
method: generated
source: openapi/_original/azure-ad-graph-changenotifications-openapi.yml, asyncapi/azure-ad-change-notifications-webhooks.yml
operations:
  - subscription_CreateSubscription
  - subscription_ListSubscription
  - subscription_GetSubscription
  - subscription_UpdateSubscription
  - subscription_DeleteSubscription
  - subscription_reauthorize
scopes:
  - User.Read.All
  - Group.Read.All
---

# Subscribe to Entra directory changes

## Create the subscription

`subscription_CreateSubscription` — `POST /subscriptions` with `changeType`,
`notificationUrl`, `resource` (`/users`, `/users/{id}`, `/groups`,
`/groups/{id}/members`, ...), `expirationDateTime` and a `clientState` secret.

Your `notificationUrl` must answer the validation handshake first: Graph POSTs a
`validationToken` and your endpoint must echo it back as `text/plain` within the
timeout, or the subscription is refused.

## Keep it alive — this is the part that breaks

Subscriptions **expire**. For user, group and other directory resources the
maximum lifetime is **41,760 minutes (under 29 days)**.

- Renew with `subscription_UpdateSubscription`, extending `expirationDateTime`,
  before it lapses.
- `subscription_reauthorize` when Graph asks you to reauthorize.
- Subscribe to **lifecycle notifications** so you are told you are about to miss
  events rather than discovering it later.
- Events that occur while no subscription exists are **not replayed**. Deleting
  and recreating a subscription is a gap, not a reset.

## Read the events correctly

- Basic notifications carry only the resource `id`; re-query Graph for the data.
- Rich notifications carry encrypted resource data and need a public key at
  subscription time.
- **Creation and soft-deletion of a user or group both arrive as `updated`, not
  as `created`/`deleted`.** Code that switches on `changeType` alone will
  misclassify both. Re-read the object, or track `deletedDateTime`.
- Verify the echoed `clientState` on every notification before acting on it.

## Quotas

Per app across all tenants: 50,000 subscriptions. Per tenant across all apps:
1,000. Per app-and-tenant pair: 100. Exceeding one returns `403 Forbidden` with
the breached limit named in the message. Not supported in Azure AD B2C tenants.

## Latency

Microsoft publishes "Unknown" for both user and group notification latency. You
cannot bound your own staleness from the published data — if freshness matters,
reconcile periodically with a delta query.

## Undo

`subscription_DeleteSubscription` stops delivery. There is no restore; you create
a new subscription, and you will not receive what happened in between.
