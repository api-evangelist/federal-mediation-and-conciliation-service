---
name: Harvest FMCS F-7 collective bargaining notice workbooks
description: >-
  Find, page through and download the Federal Mediation and Conciliation Service's monthly F-7 collective
  bargaining notice workbooks (and its FOIA logs, audit reports and per-diem sheets) from the anonymous
  FMCS WordPress REST API, and poll correctly for the new one each month.
api: openapi/federal-mediation-and-conciliation-service-wp-content-openapi.yml
operations: [listMedia, getMedia, listPages, getPage, search]
---

# Harvest FMCS F-7 notice workbooks

FMCS's most valuable public dataset is the **F-7 notice** — the statutory notice a party to a collective
bargaining agreement must file under 29 U.S.C. 158(d) before terminating or modifying that agreement. FMCS
publishes them as one Excel workbook per month. There is no dataset API, no CSV feed and no bulk endpoint:
the workbooks are WordPress media uploads, which means the media collection of the site's REST API is the
only programmatic way to enumerate them.

## Before you start

- Base URL: `https://www.fmcs.gov/wp-json`
- No credential. No signup. No API key. Do not send an `Authorization` header.
- No rate limits are published and none are signalled. Be a good citizen: page at `per_page=100`, do not
  parallelise hard, and cache — every response is `Cache-Control: no-store` with no ETag, so the server
  gives you nothing to revalidate against.

## 1. Find the workbooks

`listMedia` — `GET /wp/v2/media` — is the document library, 1,122 items at the time of writing. Filter it
rather than walking it:

```
GET /wp/v2/media?search=F7%20Notices&per_page=100&orderby=date&order=desc
GET /wp/v2/media?mime_type=application/vnd.openxmlformats-officedocument.spreadsheetml.sheet&per_page=100
```

The first narrows to the F-7 workbooks by title; the second returns every `.xlsx` on the site, which is the
F-7 series plus the FOIA logs and the FEVS reports. Read `X-WP-Total` and `X-WP-TotalPages` from the
response headers to know how far to page, and follow the `Link: rel="next"` header rather than
incrementing `page` yourself.

## 2. Read the download URL off the record

Each media record from `listMedia` or `getMedia` (`GET /wp/v2/media/{id}`) carries:

- `source_url` — the direct file URL under `/wp-content/uploads/`. This is the actual download; there is no
  redirect or download-counter indirection.
- `filesize` — byte size, returned directly. Use it to sanity-check a download.
- `mime_type`, `filename`, `title.rendered`, `date`, `modified_gmt`.
- `post` — the id of the page the file is attached to, if any. `getPage` on that id gives you the human
  context FMCS published around the file.

## 3. Get the file layout

The workbooks have a fixed column layout that FMCS documents in prose, not as a schema. Find that document
the same way:

```
GET /wp/v2/search?search=file%20layout&per_page=10
GET /wp/v2/pages/{id}
```

`/resources/documents-and-data/` is the page that publishes both the workbooks and the layout document;
`listPages?search=documents and data` finds it.

## 4. Poll for next month's workbook

FMCS publishes monthly. Do not re-list the whole library:

```
GET /wp/v2/media?modified_after=2026-08-01T00:00:00&per_page=100&orderby=modified&order=desc
```

`modified_gmt` / `modified_after` is the only change signal this API offers — there is no ETag, no
`Last-Modified` on JSON responses, no changelog and no webhook. The RSS feed at `/feed/` is dormant (newest
item 2020) and must not be used as a freshness signal.

## Errors you will actually hit

The envelope is WordPress's, not RFC 9457 — `{"code": "...", "message": "...", "data": {"status": 400}}`.
Branch on `code`, not on the HTTP status:

- `rest_invalid_param` (400) — almost always `per_page` above 100. `data.params` names the offender.
- `rest_post_invalid_id` (404) — the id is real but belongs to another post type. Ids are shared across
  types on this site.
- `rest_forbidden` (401) — you asked for an administrative route. Nothing on the public surface returns it.

## Out of scope

This skill does not file an F-7 notice. Filing is online-only since 5 April 2022 through a login-gated
portal with no public contract, and nothing in this API can submit one.
