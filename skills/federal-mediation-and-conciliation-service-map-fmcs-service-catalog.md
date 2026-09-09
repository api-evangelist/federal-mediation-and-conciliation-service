---
name: Map the FMCS service catalog and programme documentation
description: >-
  Read the Federal Mediation and Conciliation Service's own service taxonomy and programme pages out of its
  anonymous WordPress REST API — the seventeen service lines, the pages that describe each one, and the
  training events attached to them — without scraping HTML.
api: openapi/federal-mediation-and-conciliation-service-wp-content-openapi.yml
operations: [listFaqCategories, getFaqCategory, listPages, getPage, search, listEvents, listTypes, listTaxonomies]
---

# Map the FMCS service catalog

FMCS has no product catalog, no pricing page and no developer portal. What it does have — and what nothing
on the site surfaces as a list — is a seventeen-term taxonomy naming every service line the agency offers.
It is reachable in one call.

## 1. Pull the service vocabulary

`listFaqCategories` — `GET /wp/v2/faq_category?per_page=100` — returns all seventeen terms with `name`,
`slug`, `description` and `count`:

arbitration, collective-bargaining-mediation, grievance-mediation, workplace-mediation, facilitation,
dispute-resolution-systems-design, regulatory-negotiations, public-policy-dialogues,
labor-management-partnership, organizational-development, alternative-bargaining-processes,
effective-contract-administration, repairing-broken-relationships, administrative-program-dispute,
notices-and-filings, shared-neutrals, grant.

**Read `count` and expect zero.** Every term currently classifies zero records, because the `avada_faq`
post type the taxonomy binds to is empty (`listFaqs` returns `[]` with `X-WP-Total: 0`). The vocabulary is
real; the content behind it is not in the API. Do not report a service as undocumented on that basis —
resolve it against the pages instead, as below.

## 2. Resolve each service to its programme page

`listPages` — `GET /wp/v2/pages?per_page=100` — is the 117-page programme documentation set, and
`search` — `GET /wp/v2/search?search={term}&per_page=20` — maps a taxonomy slug onto it:

```
GET /wp/v2/search?search=grievance%20mediation&per_page=20
GET /wp/v2/pages/{id}
```

`getPage` returns `title.rendered`, the full `content.rendered`, `parent` and `menu_order`. The `parent`
chain reconstructs the site's information architecture — the `resources/` and `aboutus/` trees — without
crawling a single HTML page.

## 3. Attach the training

`listEvents` — `GET /wp/v2/event?per_page=100` — carries FMCS's training and convening announcements
(Mediation Skills Training, Shared Neutrals Training, the NLMC Conference). `listEspressoEventCategories`
and `listEspressoEventTypes` give the registration taxonomies those events are filed under.

## 4. Verify the model rather than assuming it

`listTypes` (`GET /wp/v2/types`) and `listTaxonomies` (`GET /wp/v2/taxonomies`) are the site's own
declaration of which post types exist and which taxonomies bind to them. Read them first if anything above
looks wrong — the FMCS site carries several plugin post types (portfolio, igmap, espresso venues) that are
registered but unused, and these two endpoints are how you tell a live collection from a vestigial one.

## Conventions that apply to every call

- Anonymous. No key, no header, no signup.
- `per_page` max 100; over that you get `rest_invalid_param` (400).
- Totals in `X-WP-Total` / `X-WP-TotalPages`; follow `Link: rel="next"`.
- `_fields=id,slug,name,count` trims responses substantially on the taxonomy calls.
- Errors are `{code, message, data.status}` — branch on `code`.

## Personal data — do not extend this skill there

The same API exposes `/wp/v2/wpbdp_listing` (121 records naming individuals in the FMCS neutrals directory)
and `/wp/v2/users` (named agency staff). Both are documented in the OpenAPI and both are deliberately
outside every skill and tool in this profile. Do not add a lookup, an enumeration or a pagination helper
for either.
