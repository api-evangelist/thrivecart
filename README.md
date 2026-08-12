# ThriveCart

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

ThriveCart is a hosted shopping cart, checkout and course platform for creators, coaches and
digital-product sellers, operated by ThriveCart LLC. Its public REST API at
`https://thrivecart.com/api/external` covers products, order bumps, upsells, downsells, pricing
options, transactions, customers, subscriptions, affiliates, Learn students and event
subscriptions.

- Website: https://thrivecart.com/
- Developer portal: https://developers.thrivecart.com/
- API reference: https://apidocs.thrivecart.com/ (Postman)
- Status: https://thrivecart.statuspage.io/

## What this profile holds

| Surface | Finding |
|---|---|
| Machine-readable contract | ThriveCart publishes **no OpenAPI**. Its reference is a first-party Postman collection, captured verbatim in `postman/`. `openapi/thrivecart-api-openapi.yml` is **derived** from it — 33 operations, 11 tags. |
| Events | 21 event keys on the Event Subscription API, plus a separate UI-configured webhook surface with a *different* event vocabulary. Both captured in `asyncapi/`; ThriveCart publishes no AsyncAPI. |
| Auth | Bearer token — account API key or OAuth 2.0 authorization code. No scope model; consent is account-wide. The real OAuth endpoints (`/authorization/new`, `/authorization/token`) are **not in the docs** — they were recovered from `src/Oauth.php` in the official PHP SDK and confirmed live. |
| Idempotency | Present on the **webhook** side only (`webhook_id`, stable across retries). The REST API has **no** idempotency key, including on `POST /refund` and `POST /cancelSubscription`. |
| SDKs | One official client: `thrivecart/php-api`, last released **1.0.11 on 2022-01-19** — four years stale against an API that gained fields in April 2026. |
| Sandbox | No sandbox host and no test credential. Test vs live is a per-product toggle read through `mode_int`; one API key touches both. |
| Agent surfaces | No first-party MCP server and no A2A agent card. Every `/.well-known/*` path except `security.txt` answers 200 with the SPA HTML shell. |

Artifacts are indexed from `apis.yml`. Everything carries provenance frontmatter recording whether
it was searched, probed, derived or generated, and from which URL.
