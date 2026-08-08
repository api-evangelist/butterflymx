# ButterflyMX

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

ButterflyMX is a property-access technology company — smart video intercoms, keypads, smart locks,
elevator controls, vehicle-access readers and package rooms installed in 20,000+ multifamily, commercial,
gated-community, student-housing and senior-living buildings.

## The API surface

- **Developer hub:** <https://apidocs.butterflymx.com/> (ReadMe)
- **API:** ButterflyMX API **v4** — `https://api.butterflymx.com`
- **Sandbox:** `https://api.na.sandbox.butterflymx.com` (NDA + Developer ToS required)
- **OpenAPI:** 3.0.1, 43 paths / **57 operations**, harvested from the provider's own reference pages
  and saved verbatim at [`openapi/_original/butterflymx-api-openapi.json`](openapi/_original/butterflymx-api-openapi.json)
- **Auth:** OAuth 2.0 authorization-code, issuer `https://accounts.butterflymx.com`, with live OIDC and
  RFC 8414 discovery documents
- **Events:** building- and tenant-scoped webhook integrations for `door_release.create` and `call.create`
- **SDKs:** iOS (Swift Package Manager / CocoaPods) and Android (raw AAR); **no server-side SDK in any language**

## What the profile records

| Area | File |
|---|---|
| OpenAPI (verbatim + YAML) | `openapi/` |
| Our enhancements (57 missing operationIds, root security, tag declarations) | `overlays/butterflymx-api-overlay.yaml` |
| OAuth 2.0 / OIDC profile | `authentication/`, `scopes/`, `well-known/` |
| Request/response semantics | `conventions/butterflymx-conventions.yml` |
| Error catalog derived from the spec's own examples | `errors/butterflymx-problem-types.yml` |
| Webhook catalog (no AsyncAPI published) | `asyncapi/butterflymx-webhooks.yml` |
| Entity graph (the spec ships zero reusable schemas) | `data-model/butterflymx-data-model.yml` |
| Sandbox and onboarding | `sandbox/butterflymx-sandbox.yml` |
| Packages / SDKs | `packages/butterflymx-packages.yml` |
| Standards conformance | `conformance/butterflymx-conformance.yml` |
| Domain security, trust center | `security/` |
| Versioning, status page, changelog | `lifecycle/`, `changelog/` |
| Agent skills for the four marquee flows | `skills/` |
| Candidate MCP tool surface (ButterflyMX operates no MCP server) | `mcp/butterflymx-mcp.yml` |

## Notable gaps found in this pass

- **Zero `operationId`s** across all 57 operations, and `components.schemas` is **empty** — nothing in the
  contract is addressable or reusable by name.
- **No idempotency contract**, on an API whose write path physically unlocks doors.
- **No webhook signature** by default; the docs make verification the integrator's problem.
- **No AsyncAPI**, no rate-limit contract, no `403`, no `5xx` declared on any operation.
- **No security.txt** and no vulnerability-disclosure program on any host.
- The developer-hub **changelog holds one entry, a welcome post from 2019**, while the spec itself was
  regenerated in June 2026 — the contract moves silently.

Company reference: <https://forgeglobal.com/butterflymx_stock/>
