# crm-field-service-consulting.com

Static pSEO content site: field service management software pricing, platform
guides, industry guides, platform comparisons, and ERP migration guides.
SMB track — the reader is an owner/ops lead deciding in weeks, price-sensitive,
reading on a phone.

## Stack

Plain static HTML, CSS-in-`<style>` per page, no build step, no framework, no
JS dependencies. Deployed on Vercel as static files. `vercel.json` sets
`"cleanUrls": true` — every internal link, canonical tag, and sitemap entry
uses extensionless URLs (`/pillar/work-order-management-software`, not
`/pillar/work-order-management-software.html`). Do not remove `cleanUrls` or
add a framework/build step without updating every link in every page.

## Structure

```
/index.html                          homepage
/quote.html                          lead form → POST /api/quote (not implemented)
/privacy.html
/pillar/work-order-management-software.html
/pricing/*.html                      3 pages
/platform/{sap,oracle,dynamics,odoo,peoplesoft,servicenow,microsoft,fusion}/*.html
/industry/{construction,manufacturing,nonprofit,telecom,transportation}/*.html
/compare/*.html                      73 "X vs Y" pages
/migrate/*.html                      86 "X to Y" migration pages
/platform/index.html, /industry/index.html, /compare/index.html, /migrate/index.html
                                      section hub pages, link to everything under them
/sitemap.xml                         264 URLs, must match what's on disk
/robots.txt
/llms.txt                            AI-citation context file
```

258 original content pages + homepage + quote + privacy + 4 section hubs = 265
HTML files total.

## Page template

Every content page shares one shape — copy the pattern from an existing page
in the same section rather than inventing a new one:

- `<head>`: title (≤60 chars), meta description (≤155 chars), canonical,
  og:title/description/type, twitter:card, then JSON-LD (`BreadcrumbList` on
  every page; `Article`/`FAQPage`/`Service`/`Organization` per BUILD_SPEC).
- `<header>`: `.site-nav` (brand + primary nav: Guide, Pricing, Platforms,
  Industries, Compare, Migrate, Get a quote) then a `.bc-row` breadcrumb nav
  specific to that page.
- `<main>`: page content.
- `<footer>`: links to every section + Privacy + Sitemap.

The nav/footer markup and its CSS (`.site-nav`, `.bc-row`) is identical byte-for-byte
across all 265 pages. If you need to change the nav, do it with a script across
every file, not by hand-editing one page at a time — see
`git log` for the `inject_nav.py`-style approach used previously (script was
run from a scratch location, not committed to this repo).

## Content rules (do not violate)

- **Nothing fabricated.** No invented staff, testimonials, client names, case
  studies, or certifications. Every number traces to a measured keyword/pricing
  data layer, not estimation.
- SMB track CTA is `pricing-quote` (→ `/quote`) everywhere. Never mix in an
  enterprise CTA like "book an assessment."
- Primary keyword goes in `<h1>`, `<title>`, and the first ~100 words. Once.
- Zero-volume or navigational keywords are not a reason to build a new page.
- Never link across domains — this site must not reference sibling microsites.

Full rules: see the source spec at `~/Desktop/mircosite/enterprise_sites/`
(`BUILD_SPEC.md`, `CONTENT_RULES.md`, `crm-field-service-consulting.md`) — that
directory is the planning source for this site but is a **separate git repo**
(`dvernon0786/mircosite`), not part of this one.

## Known gaps (see HANDOFF.md for detail)

- `/api/quote` does not exist — the quote form has no backend yet.
- 215 internal links inside the pre-existing `compare/` and `migrate/` pages
  point to comparison pages that were never generated (e.g.
  `migrate/odoo-to-ifs.html` links to `/compare/ifs-vs-odoo`, which doesn't
  exist). Pre-existing content debt, not something recent changes caused.
- No manifest CSV / site config YAML in this repo — `VALIDATION_REPORT.md`
  references them as "pre-existing artefacts" but they were never present
  here and their original schema is unknown. Do not fabricate one; regenerate
  from the mircosite pipeline if actually needed.

## Working conventions

- This is a real GitHub repo (`dvernon0786/crm-field-service-consulting`) with
  commit history — commit and push real changes, don't leave work uncommitted.
- If `git push` fails with `HTTP 408` / RPC failure on a large-ish push, try
  `git config http.version HTTP/1.1` before assuming the payload is too big —
  this fixed a transient failure previously on a 224KB push.
- Verify HTML parses (`python3 -c "from html.parser import HTMLParser; ..."`)
  and that internal `href`s resolve to real files before committing site-wide
  script-driven edits — see HANDOFF.md for the verification approach used.
