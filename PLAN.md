# Website Refresh Plan: CV & Expertise Pages

## Context

palaio.eu is currently a single-page Hugo site (theme: `hugo-split-theme`,
a git submodule of a personal fork), deployed by FTPS from `main` via
GitHub Actions (Hugo 0.108.0 extended). The only downloadable CV is
`static/resume.pdf`, a LinkedIn export from early 2023 that is now stale
and disconnected from the site's content.

Goals of this refresh:

1. A CV written in **markdown/YAML**, rendered by Hugo as a clean,
   readable, **two-column A4 page**, and **reliably exported to PDF**.
2. **More pages** presenting the various fields of expertise.
3. The **main page stays untouched** for now; its redesign is a later,
   separate step.

Decisions taken (2026-07-03, with Victor):

- **PDF output**: automated in CI with headless Chromium — the PDF is
  rebuilt from the CV page on every deploy, so it can never drift.
- **CV engine**: [mertbakir/resume-a4](https://github.com/mertbakir/resume-a4)
  (MIT, min Hugo 0.41 — compatible with our 0.108). A4-sized,
  print-friendly, optional two-column layout, content in YAML `data/`
  files with markdown support in descriptions.
- **Theming**: new pages (CV, expertise) get their own clean styling and
  coexist with the split-theme home page during the transition.
- **Language**: English only for now (i18n possible later).

## Architecture

### CV rendering

`resume-a4` is designed as a standalone site whose *home page* is the
resume (`layouts/home.html` + partials + SCSS). Since our home page must
remain on `hugo-split-theme`, we **vendor** the needed parts of
resume-a4 into this repo instead of adding a second theme (avoids
partial-name collisions between themes and a second submodule):

- `layouts/cv/single.html` — adapted from resume-a4's `home.html`
  (MIT attribution comment at the top).
- `layouts/partials/cv/…` — the theme's partials, namespaced under
  `cv/` so they cannot collide with hugo-split-theme partials.
- `assets/scss/cv.scss` (+ imports) — the theme's SCSS, compiled via
  Hugo Pipes (extended Hugo already in CI).

The CV page itself:

- `content/cv/_index.md` (or `cv.md`) with `layout: cv`, served at
  `/cv/`.
- CV data in `data/cv/*.yaml`, following resume-a4's schema:
  `features.yaml` (page/column/section arrangement), `experience.yaml`,
  `education.yaml`, plus skills, languages, certifications,
  publications. Section order and side-vs-main column placement are
  configured per page in the features file — two-column on page 1
  (side: skills, education, languages, certifications; main: summary,
  experience).

Initial CV content is drafted from the existing `static/resume.pdf`
(LinkedIn export, state of ~March 2023): PhD at ISIR/Sorbonne
Université, SoftBank Robotics Europe / Aldebaran 2010–2021, Semio 2022,
Palaio Logic 2022–present, IMT Atlantique, DeepLearning.AI and HarvardX
TinyML certificates, skills list. **Gaps needing Victor's input are
marked `TODO` in the YAML** (see "Open content questions").

### PDF generation in CI

In `.github/workflows/hugo.yml`, after `hugo --minify`:

1. Install Chromium (e.g. `browser-actions/setup-chrome` or the
   distro package) + fonts.
2. `chromium --headless=new --no-pdf-header-footer \
   --print-to-pdf=public/resume.pdf public/cv/index.html`
   (print CSS in the vendored SCSS defines the A4 layout, so the PDF is
   identical to browser print output, deterministically).
3. The PDF lands at `/resume.pdf` — same URL as today, so the existing
   home-page link keeps working. `static/resume.pdf` is then deleted
   from the repo.

While fixing the workflow, also fix the latent **baseURL bug**: the
build step references `steps.pages.outputs.base_url`, which is never
defined (no `configure-pages` step) — replace with the configured
`https://palaio.eu/` from `config.toml`.

### Expertise section

- `content/expertise/_index.md` + one page per field, seeded from the
  home page's existing service copy:
  - `virtual-and-embodied-agents.md`
  - `decision-making-dialogue-and-ai.md`
  - `embedded-development.md`
  - (optional) `machine-learning-and-tinyml.md`
- Site-level layouts `layouts/expertise/{list,single}.html` with a
  minimal, readable style (`assets/scss/pages.scss`) and a small shared
  header partial (`layouts/partials/site-nav.html`) linking
  Home / Expertise / CV. The home page does **not** get this nav yet.
- These layouts are deliberately simple: they are placeholders for the
  future full redesign, not the redesign itself.

## Implementation steps

**Phase 1 — CV page** (`content/cv/`, `data/cv/`, vendored layouts)
1. `git submodule update --init --recursive` so the site builds locally.
2. Vendor resume-a4 layout/partials/SCSS as described above.
3. Write `data/cv/*.yaml` from the 2023 resume content, with `TODO`
   markers for 2023–2026.
4. Verify: `hugo server`, check `/cv/` rendering and browser print
   preview (A4, two columns, page breaks).

**Phase 2 — PDF automation & workflow fixes**
5. Add the Chromium print step to `hugo.yml`; fix the baseURL bug;
   delete `static/resume.pdf`.
6. Verify: run the workflow on this branch (deploy step skipped or
   dry-run), download the built `public/` and inspect `resume.pdf`.

**Phase 3 — Expertise pages**
7. Add layouts, nav partial, and the seed pages.
8. Verify: `hugo server`, check `/expertise/` list and each page.

**Phase 4 — later, out of scope here**
- Main page redesign (integrate nav, unify styling or adopt a
  multi-page theme).
- French translation via Hugo i18n.
- Possibly upgrading Hugo beyond 0.108.

## Open content questions (for Victor)

- **2023–2026 at Palaio Logic**: clients, missions, and outcomes since
  the March 2023 LinkedIn export (the PDF says "Jan 2022 – Present,
  1 yr 3 mo"). LinkedIn blocks anonymous scraping, so this must come
  from you (or an updated LinkedIn PDF export dropped in the repo).
- The 2023 export shows company icons but no names for the
  2010–2021 roles — confirm they should be labeled Aldebaran /
  SoftBank Robotics Europe explicitly.
- Contact block: which email on the CV (`victor@paleologue.fr` vs
  `contact@palaio.eu`)? Phone? Photo or logo?
- Review/rewrite of the expertise page texts (seeded from existing
  home-page copy).

## Verification (end-to-end)

- `hugo server` → `/`, `/cv/`, `/expertise/` all render; home page
  byte-identical to before (no layout overrides leak into it).
- Browser print preview of `/cv/` fits A4 with clean page breaks.
- CI run on this branch produces `public/resume.pdf`; open and check
  layout, fonts, links.
- After merge to `main`: live check of palaio.eu/cv/ and
  palaio.eu/resume.pdf.
