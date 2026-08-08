---
name: Issue a ButterflyMX visitor pass
description: >-
  Issue a time-limited virtual key so a guest, contractor or delivery driver can open a building's doors
  without the ButterflyMX app, then retrieve the PIN/QR credential or revoke it.
api: openapi/butterflymx-api-openapi.yml
operations:
  - listTenants
  - showBuilding
  - listAccessPoints
  - listDevices
  - createAOneTimeKeychain
  - createACustomKeychain
  - createADeliveryPassKeychain
  - listVirtualKeys
  - deleteKeychain
method: generated
generated: '2026-08-08'
source: https://apidocs.butterflymx.com/docs/virtual-keys
---

# Issue a ButterflyMX visitor pass

Visitor passes are **keychains**. A keychain is the configuration object (who, where, when); the
**virtual keys** attached to it are the credentials a guest actually uses.

> `operationId`s in this skill come from `overlays/butterflymx-api-overlay.yaml`. The published
> ButterflyMX OpenAPI declares none, so every step also names the literal method + path.

## Before you start

- Auth is an OAuth 2.0 bearer token (`Authorization` header), valid **24 hours**. See
  `authentication/butterflymx-authentication.yml`.
- Test against `https://api.na.sandbox.butterflymx.com`, not production. See `sandbox/`.
- **There is no idempotency key.** Do not blindly retry a keychain POST — a timeout may already have
  created the pass. Re-`GET /v4/keychains` filtered by `q[name_eq]` before retrying.

## Steps

1. **Find the tenant.** `listTenants` — `GET /v4/tenants`.
   Add `scope=self` to restrict to the authenticated tenant, or `q[email_eq]` / `q[full_name_start]` to
   search. Keep `id` (the `tenant_id`) and `building_id`.

2. **Get the building's time zone.** `showBuilding` — `GET /v4/buildings/{id}`.
   Read `time_zone` (e.g. `America/New_York`). **All keychain timestamps must be sent in UTC** — convert
   the visitor's local window before you post it. This is the single most common failure in this flow.

3. **Choose the access targets.**
   - Common doors, gates, intercoms, keypads: `listAccessPoints` — `GET /v4/access_points?q[building_id_eq]={building_id}` → `access_point_ids`.
   - Unit-level smart locks: `listDevices` — `GET /v4/devices?q[building_id_eq]={building_id}` → `device_ids`.

4. **Create the keychain** with the type that matches the visit:

   | Need | Operation | Path |
   |---|---|---|
   | Scheduled, multi-use window | `createACustomKeychain` | `POST /v4/keychains/custom` |
   | Single use (15-min grace after first use) | `createAOneTimeKeychain` | `POST /v4/keychains/one_time` |
   | Delivery (5-min grace, exactly one key) | `createADeliveryPassKeychain` | `POST /v4/keychains/delivery_pass` |
   | Recurring weekday schedule | `createARecurringKeychain` | `POST /v4/keychains/recurring` |

   ```json
   {
     "keychain": {
       "name": "Visitor - Electrician",
       "tenant_id": 377389742,
       "starts_at": "2025-05-01T13:00:00Z",
       "ends_at": "2025-05-01T15:00:00Z",
       "recipients": ["guest@example.com", "+12345678900"],
       "access_point_ids": [842395203],
       "device_ids": [708353660]
     }
   }
   ```

   You must supply **either** `tenant_id` **or** `unit_id`, plus at least one access target — otherwise
   you get a `422` with `must_have_an_associated_unit_or_tenant`.

5. **Deliver the credential.**
   - Populate `recipients` and ButterflyMX emails/SMSes the key link itself.
   - Send an **empty `recipients` array** to suppress that, then read the PIN and QR from
     `listVirtualKeys` — `GET /v4/virtual_keys` (fields `pin_code`, `qr_code_url`, `instructions_url`) and
     deliver through your own channel.

6. **Revoke when needed.** `deleteKeychain` — `DELETE /v4/keychains/{id}`. This immediately deactivates
   every virtual key attached to the keychain.

## Errors to handle

From `errors/butterflymx-problem-types.yml`. The envelope is `{"errors":[{field, code, message}]}` — not
RFC 9457.

| Code | Meaning | Fix |
|---|---|---|
| `start_time_must_be_before_the_end_time` | `starts_at` >= `ends_at` | Order and re-send |
| `end_time_must_be_after_the_start_time` | same, reported on `ends_at` | Order and re-send |
| `must_have_an_associated_unit_or_tenant` | neither `tenant_id` nor `unit_id` supplied | Supply one |
| `not_found` on `tenant_ids` / `unit_ids` | id is not visible to this token | Re-run step 1 |
| `unauthorized` (401) | token expired (24h) or wrong environment | Refresh the token |

Note there is **no 403** in the contract — a permission failure surfaces as `401` or as an empty list.
