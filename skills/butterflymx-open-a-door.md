---
name: Open a ButterflyMX door programmatically
description: >-
  Collect the tenant and target ids and issue a door release request so an application can offer
  swipe-to-open or tap-to-open for an access point or a unit-level smart lock.
api: openapi/butterflymx-api-openapi.yml
operations:
  - listTenants
  - listAccessPoints
  - listDevices
  - showSchedules
  - createDoorReleaseRequests
  - listAccessLogs
method: generated
generated: '2026-08-08'
source: https://apidocs.butterflymx.com/docs/grant-access
---

# Open a ButterflyMX door programmatically

`POST /v4/door_release_requests` is the only operation in this API with a **physical** side effect.
Treat it accordingly.

> `operationId`s come from `overlays/butterflymx-api-overlay.yaml`; the published spec declares none.

## Guardrails — read before wiring this up

- **No idempotency key exists.** ButterflyMX documents none, and there is no `Idempotency-Key` parameter
  anywhere in the spec. A retry after a timeout **opens the door again**. Never put this call behind a
  blind retry or an exponential-backoff wrapper.
- The API declares **no 429 and no 5xx** on this operation, so failure modes are undocumented. Treat any
  non-2xx as "state unknown" and confirm via `listAccessLogs` rather than retrying.
- Authorization is per-user: a tenant may release only their own access points; an admin may release any
  access point in a building they manage. There is **no 403** — insufficient permission surfaces as `401`
  or as an empty collection.
- An agent driving this operation should require human confirmation and a short-lived, purpose-bound
  token. See `mcp/butterflymx-mcp.yml` (`consequence: safety-critical`).

## Steps

1. **Resolve the tenant.** `listTenants` — `GET /v4/tenants` (`scope=self` for the authenticated user).
   Keep `id` → `tenant_id`, and `building_id`.

2. **Resolve the target.** Exactly one of:
   - **Access point** (intercoms, ACS controllers, smart keypads, common-area smart locks):
     `listAccessPoints` — `GET /v4/access_points?q[building_id_eq]={building_id}` → `access_point_id`.
   - **Device** (unit-level smart locks): `listDevices` — `GET /v4/devices?q[building_id_eq]={building_id}` → `device_id`.

3. **Optional: check the window.** `showSchedules` — `GET /v4/access_points/{id}/schedules` returns the
   `weekdays` / `from` / `to` windows during which the current user may use that access point.

4. **Release.** `createDoorReleaseRequests` — `POST /v4/door_release_requests`.

   ```json
   { "door_release_request": { "access_point_id": 123, "tenant_id": 567 } }
   ```

   or

   ```json
   { "door_release_request": { "device_id": 115677, "tenant_id": 531232 } }
   ```

   The response carries `id`, `guid` and `release_method`. **Record the `guid`** — it is the only handle
   you have for reconciling a request whose response you never received.

5. **Confirm, do not retry.** `listAccessLogs` —
   `GET /v4/buildings/{building_id}/access_logs?q[logged_at_gteq]={t}` and match on `access_point`,
   `entry_method` and `release_status`. A `door_release.create` webhook delivers the same record in real
   time if a building or tenant integration is configured — see `asyncapi/butterflymx-webhooks.yml`.

## Environment

Sandbox base URL is `https://api.na.sandbox.butterflymx.com`; production is
`https://api.butterflymx.com`. There are no test-mode key prefixes — the environment is the hostname plus
the credential pair you were issued. See `sandbox/butterflymx-sandbox.yml`.
