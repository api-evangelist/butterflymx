---
name: Manage ButterflyMX resident access credentials
description: >-
  Onboard a tenant into a unit and provision, rotate or revoke their permanent access credentials — PIN
  codes and RFID tags — plus access-group membership.
api: openapi/butterflymx-api-openapi.yml
operations:
  - listBuildings
  - createUnit
  - createTenant
  - listTenants
  - resendConfirmationEmail
  - listAccessTools
  - createAPinAccessTool
  - changeThePinOfAnAccessTool
  - createARfidTagAccessTool
  - deleteAccessTool
  - listAccessGroups
  - addTenantsToAccessGroup
  - removeTenantsFromAccessGroup
method: generated
generated: '2026-08-08'
source: https://apidocs.butterflymx.com/docs/manage-resident-pin
---

# Manage ButterflyMX resident access credentials

Permanent access is an **access tool**: a PIN or an RFID tag bound to a tenant. Temporary access is a
keychain — see `butterflymx-issue-visitor-pass.md`.

> `operationId`s come from `overlays/butterflymx-api-overlay.yaml`; the published spec declares none.

## Onboard

1. `listBuildings` — `GET /v4/buildings` → `building_id`.
2. `createUnit` — `POST /v4/buildings/{building_id}/units` if the unit does not exist (`label`, `floor`).
3. `createTenant` — `POST /v4/tenants` with the tenant's name, email and unit. ButterflyMX emails an
   invitation; the tenant's `registration_status` stays `invited` until they accept.
4. `resendConfirmationEmail` — `POST /v4/tenants/{tenant_id}/resend_confirmation` re-sends that invitation.
   Only valid while the tenant is still `invited`.

## PINs

- **List:** `listAccessTools` — `GET /v4/access_tools?q[tenant_id_eq]={tenant_id}`. PIN entries have
  `type: "pin"` and a `code`.
- **Create:** `createAPinAccessTool` — `POST /v4/access_tools/pins`

  ```json
  { "pin": { "code": "235704", "tenant_id": 175394 } }
  ```

  **Only one PIN is allowed per tenant.** ButterflyMX generates an initial PIN at registration, so a
  create usually fails on an existing tenant — list first, then update instead.

- **Rotate:** `changeThePinOfAnAccessTool` — `PUT /v4/access_tools/pins/{id}` with `{ "code": "235708" }`.

- **Complexity rules** (enforced server-side, surfaced as `422`):
  - `wrong_length` — the code must be 6 characters.
  - `low_entropy` — no sequences (`123456`, `654321`) and no repeated digits (`111111`).
  - 4-digit PINs are grandfathered and exempt, but require the resident to enter a unit label alongside
    the PIN.

## RFID tags (keyfobs, keycards, vehicle stickers)

- **Create:** `createARfidTagAccessTool` — `POST /v4/access_tools/rfid_tags`.
- **RFID tags cannot be updated.** To change one, `deleteAccessTool`
  (`DELETE /v4/access_tools/{id}`) and create a new one.
- Identifiers accept only hex characters — a bad value returns `422` `invalid` on `item.identifier` with
  *"The fobs, cards, and tags do not accept special characters or letters outside of a-f"*.

## Access groups

Access groups grant sets of tenants and units access to parts of a property on a schedule.

- `listAccessGroups` — `GET /v4/access_groups`
- `addTenantsToAccessGroup` — `POST /v4/access_groups/{id}/tenants`
- `removeTenantsFromAccessGroup` — `DELETE /v4/access_groups/{id}/tenants/bulk_destroy`
- `addUnitsToAccessGroup` / `removeUnitsFromAccessGroup` — the same pair under `/units`.

A bulk add that references ids the token cannot see returns `422` with `not_found` on `tenant_ids` or
`unit_ids` — the whole call fails rather than partially applying, so validate ids first with
`listTenants`.

## Offboard

Revoke in this order so no window is left open:

1. `deleteAccessTool` — `DELETE /v4/access_tools/{id}` for every PIN and RFID tag.
2. `deleteKeychain` — `DELETE /v4/keychains/{id}` for any outstanding visitor pass.
3. `removeTenantsFromAccessGroup`.
4. `deleteTenant` — `DELETE /v4/tenants/{id}`.

`is_protected` (`422`) on a delete means the resource is system-protected and cannot be removed via the
API. Reads are permission-scoped; there is no 403, so an unexpected empty list usually means the token
lacks scope rather than that the data is absent.
