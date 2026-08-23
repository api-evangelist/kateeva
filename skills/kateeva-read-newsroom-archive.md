---
name: Read the Kateeva newsroom archive
description: >-
  Search and retrieve Kateeva's published press releases, in-the-news coverage, event notices and
  blog articles on OLED inkjet printing, thin film encapsulation and the YIELDjet platform, from
  the public read-only content API. No credentials required.
api: openapi/kateeva-posts-api-openapi.yml
operations: [searchContent, listCategories, listPosts, getPost, listTags, getSeoHead]
---

# Read the Kateeva newsroom archive

152 published items on OLED inkjet deposition, thin film encapsulation, the YIELDjet platform and
the company itself, available anonymously as JSON.

Base URL: `https://kateeva.com/wp-json`

## Authentication

None. Anonymous HTTPS. The collection answers `Allow: GET`.

## Steps

1. **Get the category map first** — `listCategories`
   `GET /wp/v2/categories?per_page=20&_fields=id,name,slug,count,parent`
   This is the only reliable retrieval axis on this surface. At capture: Press releases `9` (53),
   In the news `11` (71), Kateeva Blog `13` (19) with children Spotlight on People `14` (10),
   Spotlight on Technology `15` (11), Spotlight on Business `16` (5), Events `12` (3).
   Do **not** rely on tags — 26 exist, the highest count is 5, and several are empty or misspelled.

2. **Search across every content type** — `searchContent`
   `GET /wp/v2/search?search=<terms>&per_page=20`
   Returns lightweight `{id, title, url, type, subtype}` records across posts and pages. `type` is
   always `post`; branch on **`subtype`** — `post` for a newsroom item, `page` for a site page.
   Each result's `_links.self` gives the correctly typed URL to fetch.

3. **Or walk one category by date** — `listPosts`
   `GET /wp/v2/posts?categories=9&per_page=100&orderby=date&order=desc&_fields=id,date,slug,title,link,categories`
   Read `X-WP-Total` and `X-WP-TotalPages` off the response headers rather than counting.
   Narrow with `after=<ISO8601>` / `before=<ISO8601>`, or `modified_after` to pick up edits.

4. **Retrieve the article body** — `getPost`
   `GET /wp/v2/posts/{id}?_fields=id,date,modified,title,content,excerpt,link,categories,tags`
   `content.rendered` is HTML, not markdown or plain text. `title.rendered` and `excerpt.rendered`
   are HTML-escaped — unescape before display (`R&#038;D` is `R&D`).

5. **Get structured metadata instead of scraping** — `getSeoHead`
   `GET /yoast/v1/get_head?url=<article url>`
   Returns the rendered head plus the parsed schema.org `@graph` for the URL.

## Conventions that will bite you

- **Always send `_fields`.** Two unfiltered posts are 10.8KB; the same two with
  `_fields=id,date,slug,link,title` are 465 bytes.
- **`per_page` is capped at 100.** 101 or more returns `400 rest_invalid_param` with the bound in
  `data.params.per_page`.
- **The archive is stale.** The most recent post is dated **2024-07-01**. This is a corpus, not a
  stream — do not build a "latest news" flow that assumes recency.
- **No rate-limit headers exist.** Nothing tells you when to back off. Self-throttle.
- **No `ETag` or `Last-Modified`.** Conditional requests are not available; a poller re-transfers
  the whole collection every time. Use `modified_after` to reduce the payload instead.

## Errors

Match on `code`, never on `message`. Full catalog: `errors/kateeva-problem-types.yml`.

| status | code | meaning |
|---|---|---|
| 400 | `rest_invalid_param` | a query parameter is out of bounds; read `data.params` |
| 401 | `rest_forbidden` | privileged read; no public credential path exists |
| 404 | `rest_no_route` | the path is not registered |
| 404 | `rest_post_invalid_id` | the id does not resolve to a *published* post |

## Reversibility

Nothing to reverse — every operation here is a GET. See `conventions/kateeva-conventions.yml`.
