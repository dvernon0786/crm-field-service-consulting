# Handoff — crm-field-service-consulting.com

## What this is

Static pSEO site: 258 content pages (pricing, pillar, platform, industry,
compare, migrate) targeting field service management software keywords,
originally planned in `~/Desktop/mircosite/enterprise_sites/crm-field-service-consulting.md`
(a separate repo, `dvernon0786/mircosite`) but scoped up during build to
258 pages instead of the plan's original 4. Deployed on Vercel as static HTML,
no build step.

Live: https://crm-field-service-consulting.vercel.app (also configured for
the apex domain per canonical tags: `crm-field-service-consulting.com`)

## Status: deployed, nav complete, one open content gap

### Session 1 (2026-08-24) — initial commit and infra

The repo had 258 HTML content pages, `robots.txt`, and `sitemap.xml` on disk
but **no git history** (despite `origin` already pointing at
`dvernon0786/crm-field-service-consulting`) and several things missing that
every page linked to:

- No homepage (`index.html`)
- No `/quote` page (every page's primary CTA target)
- No `/privacy` page (every footer linked here)
- No `llms.txt` (referenced in `VALIDATION_REPORT.md` as pre-existing, but absent)
- Two empty junk directories, `{platform` and `{pricing,pillar}`, left by a
  shell brace-expansion mistake — deleted, they held nothing

Built all of the above and made the first commit (`3551ce2`), then pushed to
`origin/main`.

**Decision made with the user:** the plan at
`~/Desktop/mircosite/enterprise_sites/crm-field-service-consulting.md` specs
only 4 pages (3 pricing + 1 pillar), gated on measured keyword volume. The
live repo has 258. User chose to **keep all 258** rather than trim to the
plan — treat the extra platform/industry/compare/migrate pages as accepted
scope, not a bug to fix.

### Session 1 continued — 404s on Vercel

User reported every page 404ing on the Vercel deployment
(`/pillar/work-order-management-software` → `404: NOT_FOUND`). Root cause: no
`vercel.json` existed, so Vercel served static `.html` files literally —
`/pillar/work-order-management-software.html` would have resolved, but every
internal link, canonical tag, and sitemap entry uses the extensionless form.

Fix: added `vercel.json` with `"cleanUrls": true, "trailingSlash": false`
(commit `f2c117f`). This must stay in place — removing it breaks every link
on the site.

### Session 2 (2026-08-24/25) — nav bar, footer, section hubs

User asked for a real header/footer/nav bar — until this point every page had
only a bare logo + breadcrumb, no site-wide navigation, and the breadcrumbs on
`compare/`, `industry/`, `migrate/`, and `platform/` pages linked to section
index pages (`/compare/`, `/industry/`, etc.) that didn't exist.

Built a Python script (run from the session scratchpad, not committed to this
repo) that:

1. Replaced the `<header>` block on all 261 then-existing pages with a
   `.site-nav` bar (brand + Guide/Pricing/Platforms/Industries/Compare/Migrate/
   Get-a-quote links), preserving each page's original breadcrumb `<nav>`
   underneath it.
2. Replaced the `<footer>` block with one linking every section.
3. Injected the nav/breadcrumb/footer CSS before `</style>`.
4. Was idempotent (skipped files already containing the nav marker) and was
   tested on copies of one page from each section type before running
   site-wide, after an initial dry-run-that-wasn't-actually-dry surfaced two
   bugs (duplicate `class="bc-row"` attribute on pages whose original
   breadcrumb already had a `class="bc"`, and a missing newline before
   `<main>`). Fixed, re-verified on isolated copies, then re-run for real.

Also generated 4 new hub pages via a second script:

- `/platform/index.html` — links to all 8 platforms
- `/industry/index.html` — links to all 5 industries
- `/compare/index.html` — links to all 73 comparison pages
- `/migrate/index.html` — links to all 86 migration pages

Added those 4 URLs to `sitemap.xml` (259 → 264 entries).

**Verification performed before commit:**
- All 265 HTML files parse without error (`html.parser.HTMLParser`).
- Every internal `href` in every page checked against files actually on disk.
  Found 215 broken links, but traced every one to pre-existing lateral
  cross-links inside the original `compare/`/`migrate`/`platform`/`industry`
  content (e.g. `migrate/odoo-to-ifs.html` linking to
  `/compare/ifs-vs-odoo`, which was never built) — **zero** were caused by
  the new nav or hub pages. Left as a known, separate gap (see below);
  flagged to the user rather than silently expanding scope to fix it.
- `sitemap.xml` re-validated as well-formed XML.

Committed as `0ff7096`. Push initially failed with `HTTP 408` / RPC
disconnect (payload was only ~224KB, so not a size issue — looked like a
transient proxy/HTTP2 problem). Fixed by `git config http.version HTTP/1.1`,
then the push succeeded on retry. If this recurs, try that before assuming
anything is actually wrong with the commit.

## Open gaps

1. **`/api/quote` doesn't exist.** `quote.html` POSTs there; nothing handles
   it. The form has a honeypot field and CSRF-token placeholder wired up
   client-side per BUILD_SPEC §8, but there's no backend. Needs a real
   endpoint (serverless function on Vercel, or an external form service)
   before the site can actually capture a lead.
2. **215 broken internal links** inside the original compare/migrate content,
   pointing at comparison-page combinations that were never generated (e.g.
   IFS-vs-Odoo, Infor-vs-NetSuite, Microsoft-vs-Oracle — run the verification
   script in this doc's history to regenerate the current list). Two ways to
   close this: build the missing comparison pages, or edit the dangling links
   to point only at pages that exist. Neither has been decided.
3. **No manifest CSV / site config YAML in this repo.** `VALIDATION_REPORT.md`
   calls them "pre-existing artefacts" but they've never been present here.
   Skipped recreating them (session 1) since nothing on the live site depends
   on them and their original schema is unknown — would be fabricating
   structure, not restoring it.
4. **Domain registration/DNS unconfirmed.** Per the original plan
   (`BUILD_ORDER.md` in the mircosite repo), nobody had checked whether
   `crm-field-service-consulting.com` was available/registered as of the plan
   date. The Vercel deployment is live at the `.vercel.app` subdomain; whether
   the apex domain is actually pointed at it is unverified from this session.

## Key files

- `vercel.json` — `cleanUrls: true`. Do not remove.
- `sitemap.xml` — 264 URLs, must be kept in sync with pages on disk.
- `llms.txt` — AI-citation context file, describes site structure for LLM
  crawlers (see `robots.txt` for which bots are explicitly allowed/blocked).
- `CLAUDE.md` — project conventions and page-template documentation for future
  sessions working in this repo.
