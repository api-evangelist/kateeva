---
name: Map the Kateeva site, products and company facts
description: >-
  Reconstruct what Kateeva is, what it sells and how its site is organised, working around the fact
  that kateeva.com renders its page bodies client-side and serves an empty shell to a crawler. Uses
  the page hierarchy, the SEO head endpoint and the newsroom archive. No credentials required.
api: openapi/kateeva-pages-api-openapi.yml
operations: [listPages, getPage, getSeoHead, getOEmbed, searchContent, getPost, listMedia]
---

# Map the Kateeva site, products and company facts

Kateeva is a Newark, California capital-equipment maker whose YIELDjet inkjet printing systems
deposit the organic thin film encapsulation layer of OLED displays. Its public site is 29 pages and
152 newsroom items.

Base URL: `https://kateeva.com/wp-json`

## The problem this skill solves

`kateeva.com` is a Vue-rendered site. The `content.rendered` field on almost every **page** is
**empty** — the marketing copy is injected client-side. Fetching the HTML gets you a shell, and
fetching the Pages API gets you a hierarchy with no body. The company's actual substance lives in
two other places: the **SEO head** endpoint (structured, per URL) and the **newsroom posts**
(which *do* carry full `content.rendered`).

## Steps

1. **Reconstruct the site tree** — `listPages`
   `GET /wp/v2/pages?per_page=100&_fields=id,slug,title,link,parent,menu_order&orderby=id&order=asc`
   29 pages. Follow `parent` to rebuild the nav: `overview` (93) → `about` (11) → `company` (95).
   `parent: 0` marks a top-level page.

2. **Get the meaning of each page** — `getSeoHead`
   `GET /yoast/v1/get_head?url=<page link>`
   Returns the rendered `<head>` plus a parsed schema.org `@graph`. This is where the page title,
   description and canonical live when `content.rendered` is empty. Run it over the `link` of every
   page from step 1 to get a machine-readable site description.

3. **Or get a compact embed card** — `getOEmbed`
   `GET /oembed/1.0/embed?url=<page link>`
   Returns `{title, provider_name, author_name, thumbnail_url, html}`. Note `author_name` is the
   WordPress account (`dev`), not a person — do not attribute content to it.

4. **Get the real company facts from the newsroom** — `searchContent` then `getPost`
   The press releases carry full bodies and are the authoritative first-party source for company
   history. Useful queries: `YIELDjet`, `TFE`, `Series`, `CEO`, `Tianma`. Then
   `GET /wp/v2/posts/{id}?_fields=id,date,link,title,content` for the body.
   Worked example: `GET /wp/v2/search?search=YIELDjet&per_page=3` returns page `413`
   ("YIELDjet® Platform") and post `1630` (the Tianma order).

5. **Pull product and facility imagery** — `listMedia`
   `GET /wp/v2/media?per_page=100&_fields=id,slug,mime_type,source_url,alt_text,post`
   236 attachments. `post` links each asset back to the page or article it was uploaded to.
   Requesting a dotted sub-field (`media_details.width`) is silently dropped — ask for
   `media_details` whole.

## What you will NOT find here

There is no product, tool, application, customer or specification **entity** on this API. The
YIELDjet Lassen / Jarvis / Tioga / Kuna line exists only as prose in page titles and post bodies.
Anything structured about the machines themselves — throughput, substrate generation, host
communication (SEMI SECS/GEM, OPC UA) — is not published anywhere on this surface. See
`data-model/kateeva-data-model.yml`.

## Conventions

Always send `_fields`; `per_page` caps at 100; no rate-limit headers; no conditional requests.
Full detail in `conventions/kateeva-conventions.yml`, errors in `errors/kateeva-problem-types.yml`.
