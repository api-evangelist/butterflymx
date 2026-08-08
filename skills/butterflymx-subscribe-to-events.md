---
name: Subscribe to ButterflyMX access and call events
description: >-
  Register a building-scoped or tenant-scoped webhook integration so an application receives door_release
  and call events in real time, and reconcile against the log endpoints when a delivery is missed.
api: openapi/butterflymx-api-openapi.yml
operations:
  - listBuildings
  - listBuildingIntegrations
  - createBuildingIntegration
  - updateBuildingIntegration
  - deleteBuildingIntegration
  - createTenantIntegration
  - listAccessLogs
  - listCalls
method: generated
generated: '2026-08-08'
source: https://apidocs.butterflymx.com/docs/webhooks
---

# Subscribe to ButterflyMX access and call events

ButterflyMX has no AsyncAPI and no standalone webhook resource. Subscriptions are **integrations**
created against a building or a tenant.

> `operationId`s come from `overlays/butterflymx-api-overlay.yaml`; the published spec declares none.

## Choose the scope first

| | Building integration | Tenant integration |
|---|---|---|
| Path | `POST /v4/buildings/{building_id}/integrations` | `POST /v4/tenants/{tenant_id}/integrations` |
| Configured by | Building admin | Tenant |
| Delivers | Every event in the building | Only events involving that tenant's unit |
| Typical use | Security dashboards, delivery monitoring | Personal apps, smart-home automations |

## Steps

1. **Find the building.** `listBuildings` — `GET /v4/buildings` → `building_id`.

2. **Check for an existing subscription.** `listBuildingIntegrations` —
   `GET /v4/buildings/{building_id}/integrations`. Creating a duplicate is easy: there is **no
   idempotency key** on this API, so always list before you create.

3. **Create the integration.** `createBuildingIntegration` —
   `POST /v4/buildings/{building_id}/integrations`:

   ```json
   {
     "data": {
       "type": "integrations",
       "attributes": {
         "integrator": "webhook",
         "configuration": {
           "url": "https://your-webhook-url.com/building-events",
           "method": "post"
         },
         "bindings": [
           { "resource_type": "door_release", "actions": ["create"] },
           { "resource_type": "call",         "actions": ["create"] }
         ]
       }
     }
   }
   ```

   Only two resource types exist: `door_release` and `call`, each with the single action `create`.

4. **Handle the delivery.** Payload shape:

   ```json
   {
     "event": {
       "resource_type": "door_release",
       "action": "create",
       "data": {
         "id": 123456789,
         "logged_at": "2025-04-24T10:24:35Z",
         "access_point": 22177636,
         "entry_method": "Swipe to open",
         "release_status": "Unlocked",
         "name": "John Tenant",
         "image_url": "https://cdn.butterflymx.com/snapshot.png"
       }
     }
   }
   ```

   - **Return `200 OK` within 5 seconds** or the delivery is retried. Acknowledge first, process async.
   - **Deduplicate yourself.** ButterflyMX states the consumer must tolerate duplicate deliveries, and the
     payload carries no delivery id — key on `event.data.id` + `logged_at`.
   - **Verify origin yourself.** There is **no signature by default**; the docs describe verification as
     optional and suggest IP filtering or a shared secret you agree out of band. Do not treat an unsigned
     `door_release` event as authoritative for anything security-sensitive.

5. **Reconcile gaps.** Webhooks are at-least-once with no replay endpoint. Backfill with
   `listAccessLogs` — `GET /v4/buildings/{building_id}/access_logs?q[logged_at_gteq]=…&q[logged_at_lteq]=…`
   and `listCalls` — `GET /v4/buildings/{building_id}/calls`, both paginated with `page`/`per`.

6. **Maintain.** `updateBuildingIntegration` (`PUT .../integrations/{id}`) to change the URL or bindings;
   `deleteBuildingIntegration` (`DELETE .../integrations/{id}`) to stop delivery.

## Testing

Point the integration at `https://webhook.site` in the sandbox, or run the provider's reference receiver:
<https://github.com/runslikebutter/webhooks-demo-app>. Note that video call **media** is not delivered by
webhook — the `call.create` event is the trigger, and the ButterflyMX mobile SDK handles the stream.
