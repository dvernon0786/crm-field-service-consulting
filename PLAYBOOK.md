# Static pSEO site setup — general playbook

Repeatable steps for taking a directory of pre-generated static HTML pages
(pricing/pillar/platform/industry/compare/migrate style pSEO content, or
similar) from "pages exist on disk" to "deployed, navigable, documented."
Derived from doing this once on crm-field-service-consulting.com — apply the
same sequence to any similar project.

---

## 1. Audit before touching anything

Don't assume the directory is complete. Check, in order:

1. `ls -la` the root and every subdirectory. Look for junk — directories with
   literal `{` or `,` in the name are almost always a failed shell
   brace-expansion (`mkdir -p {a,b,c}/{x,y}` typed in a shell that didn't
   expand it). Confirm they're empty (`find <dir> -type f | wc -l`) before
   deleting.
2. `git status` / `git remote -v`. A remote can exist with zero commits —
   don't assume "has a remote" means "has history."
3. Read any validation/build report in the repo (`VALIDATION_REPORT.md`,
   `BUILD_REPORT.md`, etc.). It often references files as "pre-existing" that
   aren't actually there anymore (manifest CSVs, config YAMLs, `llms.txt`).
   Note what's missing; don't assume you should regenerate it yet.
4. Count actual files per content type (`find <dir> -name "*.html" | wc -l`)
   and compare against whatever plan/spec document exists. If there's a
   mismatch between "planned N pages" and "built M pages," that's a decision
   for the user, not something to silently resolve either direction.
5. If there's a sibling "factory" or "pipeline" repo that generated this
   site (keyword research, page-manifest generation, build scripts) — find
   it, read its build spec / content rules, but remember it is very likely a
   **separate git repo** with its own remote. Don't conflate the two.

## 2. Get a real question in front of the user before building

Two decision points come up almost every time:

- **Scope**: if the live page count doesn't match the plan, ask whether to
  trim to plan or keep everything built. Don't decide this yourself — it's a
  business call about which pages are worth keeping live.
- **Missing "pre-existing" artefacts**: if a validation report references
  files that don't exist (manifest, config, llms.txt), ask whether to
  regenerate best-effort versions or skip them. Recreating internal
  bookkeeping with a guessed schema is often wasted effort if nothing on the
  live site depends on it.

Use a real decision tool (don't just pick for the user) when the answer
changes what you build next.

## 3. Fill the structural gaps every static content site needs

Whatever the content generator produced, it's usually missing the pages every
other page links to. Check for and build:

- **Homepage** (`index.html`) — most pSEO generators don't produce one. Link
  every major section from it.
- **Whatever the CTA target is** — a `/quote`, `/contact`, `/demo` page.
  Grep every content page for its primary CTA link target and confirm the
  target file exists.
- **`/privacy`** (or whatever the footer links to) — same check, grep footers
  across a sample of pages for consistent link targets.
- **`llms.txt`** — increasingly expected; a short markdown file describing
  site structure for AI crawlers, in addition to `robots.txt`/`sitemap.xml`.
- **Section index/hub pages** — if the content is grouped into
  categories (`/compare/`, `/industry/`, `/platform/`, etc.), check whether
  `/compare/index.html` etc. actually exist. Breadcrumbs and nav links to
  category roots are a very common silent gap. Generate them
  programmatically (list every file in the category, build a linked grid) —
  don't hand-write 70+ links.

For each, match the existing page's design system (colors, fonts, header/
footer shape) — read 2-3 existing pages first to extract the CSS variables
and structural pattern before writing anything new.

## 4. Check the sitemap against the filesystem, not against itself

`sitemap.xml` is often stale or was hand-maintained. Verify:

```bash
grep -c '<url>' sitemap.xml
find . -name "*.html" -not -path "./.git*" | wc -l
```

These should reconcile (sitemap count = html file count + 1 for the
homepage, minus any `noindex` pages like a quote/contact form). Validate it's
well-formed XML (`python3 -c "import xml.etree.ElementTree as ET; ET.parse('sitemap.xml')"`).
Add any new pages you build (hub pages, privacy, etc.) to it.

## 5. Deploy and fix the platform-specific routing gap

If deploying static HTML with **extensionless internal links**
(`/pillar/foo` instead of `/pillar/foo.html` — check by grepping a sample
page's `href`s) to Vercel/Netlify/etc., the host needs to be told to map
clean URLs to `.html` files. This is an extremely common "everything 404s"
bug because it's invisible until you actually load a page.

- Vercel: `vercel.json` with `{"cleanUrls": true, "trailingSlash": false}`.
- Netlify: equivalent is usually automatic for `.html` under `pretty URLs`,
  but verify; may need `_redirects`.
- Any host: confirm by checking one working URL and one directory-style
  category URL after first deploy, don't assume it worked.

## 6. Build the site-wide nav/footer as a script, not by hand

If pages only have a bare logo + breadcrumb (common for machine-generated
pSEO pages — the generator does per-page metadata well but skips site-wide
chrome), do NOT hand-edit each file. Write a script:

1. Regex-match the existing `<header>...</header>` block.
2. Extract and preserve the page-specific part (usually the breadcrumb nav)
   — don't discard per-page context when adding the site-wide part.
3. Build the new header = site nav bar + preserved breadcrumb.
4. Regex-match and replace `<footer>...</footer>` with a footer that links
   every section.
5. Inject the new CSS classes before `</style>`.
6. Make it **idempotent** — check for a marker class/string and skip files
   that already have it, so re-running is always safe.

**Test on 2-3 copies of different page types first** (a page whose
breadcrumb had one CSS class, a page whose breadcrumb had none, a page with
no breadcrumb at all) before running site-wide. Diff or `sed -n` the output
and check specifically for:
- Duplicate attributes (e.g. `class="x" class="x"`) if your regex both
  reuses an existing class attribute and adds a new one.
- Missing whitespace/newlines where regex-replaced blocks get glued to
  adjacent tags (`</header><main>` with no line break is a common tell).
- Exactly one `<header>` and one `<footer>` per file after the run.

If the first run has bugs, `git checkout -- .` to revert cleanly (safe only
if the run was uncommitted — verify `git status` first) and re-run the fixed
script rather than patching the already-mutated files.

## 7. Verify before committing, every time

After any site-wide scripted edit:

```bash
# Every file still parses as valid HTML
python3 -c "
from html.parser import HTMLParser
import glob
bad = []
for f in glob.glob('**/*.html', recursive=True):
    if '.git' in f: continue
    try: HTMLParser().feed(open(f).read())
    except Exception as e: bad.append((f, str(e)))
print('parse errors:', len(bad))
"

# Every internal href resolves to a real file on disk
# (write a small script: walk all .html files, regex out href="/...",
#  check the target exists as file.html or file/index.html)
```

When the link-check turns up broken links, **trace each one back to its
source file** before assuming you broke something — cross-links inside
pre-existing content (e.g. one comparison page linking to a sibling
comparison that was never generated) are common and are not something your
current change caused. Report them to the user as a separate, flagged gap
rather than silently expanding your change's scope to fix unrelated content
debt.

## 8. Commit and push in small, explained increments

- One commit per logical change (infra, deploy-config fix, nav/footer,
  docs) — not one giant commit for everything.
- Write commit messages that explain *why*, not just *what* — e.g. "add
  vercel.json with cleanUrls to fix 404s on extensionless routes" beats "add
  vercel.json."
- If `git push` fails with `HTTP 408` or "empty reply from server" on a
  small payload (a few hundred KB), it's very likely transient — before
  assuming anything is broken:
  1. Confirm the commit landed locally (`git log`, `git status` clean).
  2. Confirm `git fetch` still works (isolates push-specific vs. general
     connectivity/auth failure).
  3. Try `git config http.version HTTP/1.1`, then retry.
  4. If it still fails, just wait ~10-20s and retry again — these often
     self-resolve without any config change.

## 9. Write down what you did and what's still open

At the end, produce (or update) two documents in the repo itself:

- **`CLAUDE.md`** (or equivalent project-instructions file): stack, file
  structure, the page-template pattern (so a future edit copies the existing
  convention instead of inventing a new one), content rules that must not be
  violated, and known gaps. This is for whoever/whatever works in the repo
  next — write it assuming zero memory of this session.
- **`HANDOFF.md`**: a chronological build log — what was broken, what was
  built, what was verified and how, what's still open. Include enough detail
  that a broken-link count or a "why does this file exist" question can be
  answered by reading this file instead of re-deriving it.

Before writing to either, check whether they already exist **in this repo
specifically** — a sibling/parent "factory" repo often has its own
`CLAUDE.md`/`HANDOFF.md` documenting a different tool. Don't edit the wrong
repo's docs by mistake; confirm the target with the user if ambiguous.

---

## Quick checklist (copy this into a new project)

- [ ] Audit: junk dirs, git history, validation-report claims vs. reality,
      planned vs. actual page count
- [ ] Resolve scope/missing-artefact questions with the user before building
- [ ] Build: homepage, CTA target page, footer-linked pages (privacy etc.),
      llms.txt, section hub/index pages
- [ ] Sitemap reconciled against filesystem, valid XML
- [ ] Deploy-host clean-URL config in place, verified on one nested route
- [ ] Nav/footer injected via idempotent script, tested on samples first
- [ ] Post-edit verification: HTML parses, internal links resolve
- [ ] Broken links traced to source and reported, not silently "fixed" by
      scope creep
- [ ] Small commits, real messages, push retried on transient failures
- [ ] CLAUDE.md + HANDOFF.md written/updated in the correct repo
