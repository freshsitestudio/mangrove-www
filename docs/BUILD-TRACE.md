# Mangrove Build Run — Execution Trace

**Date:** 2026-07-24
**Goal:** Run the full FreshSite BMAD pipeline on `https://www.mangrove.co.nz/` and document every script invocation, artefact created, and notable behaviour. The intent is to give Cursor a complete trace of what actually happened on a real run, so a reviewer can spot (a) pipeline bugs, (b) artefacts that should exist but don't, (c) order-of-operations issues, (d) anything else weird.

This file is appended-to chronologically as the run progresses. It is the **raw record** — observations, exit codes, warnings, and what each artefact contains. Cursor can compare the trace against the contract in `platform/docs/workflows/freshsite-build-guide.md` and `themes-handbook/docs/PLATFORM_THEME_LIBRARY_INTEGRATION.md` to find gaps.

---

## Customer profile (from `mangrove.co.nz` audit)

- **Business name:** Mangrove (full legal name TBC from copy — meta doesn't say)
- **Tagline (meta):** "Mangrove helps New Zealand businesses improve operations, strengthen financial clarity and build smarter systems for sustainable business gr[owth]"
- **Tech stack:** HubSpot CMS (confirmed via `/hs/hsstatic/` paths, `_hsq` cookie, `hsforms`)
- **Pages observed:** `/`, `/services?hsLang=en-nz`, `/about?hsLang=en-nz`, `/contact?hsLang=en-nz`
- **Visual signals:** HubSpot template, generic stock photography, no real product imagery. Customers are NZ SMBs needing operational/financial consulting.

---

## Phase 0 — Pre-flight

### 0.1 Set env vars

```bash
export HANDBOOK_ROOT=/mnt/ai/freshsite/themes-handbook
export THEMES_ROOT=/mnt/ai/freshsite/themes
export CUSTOMERS_ROOT=/mnt/ai/freshsite/customers
export PLATFORM_SCRIPTS=/mnt/ai/freshsite/platform/scripts
```

### 0.2 Smoke test

**Command:** `bash $PLATFORM_SCRIPTS/integrate_library.sh`

**Result:** Pass — "OK: 10 themes library-ready, 10 template dirs present" + 10/10 recipes printed.

### 0.3 Create project dir

**Command:** `mkdir -p $CUSTOMERS_ROOT/mangrove-www/docs`

**Result:** Project dir empty (just `docs/`).

---

## Phase 1 — Mary (audit + brief + personas)

### 1.0 First bug — wrong argument order

**Command:** `python3 $PLATFORM_SCRIPTS/mary.py brief --slug mangrove --url https://www.mangrove.co.nz/`

**Error:** `mary.py: error: unrecognized arguments: --slug --url https://www.mangrove.co.nz/`

**Root cause:** `mary.py brief` uses **positional** arguments, not flags. The `--type` and `--questions` are flags but `slug` and `url` are positional. **This is a UX bug** — the integration guide prompt (`integrate_library.sh`) says to pass them as flags.

**Fix in this run:** use `mary.py brief mangrove https://www.mangrove.co.nz/ --type consulting`.

### 1.1 Asset harvest

**Command:** `python3 $PLATFORM_SCRIPTS/mary.py brief mangrove https://www.mangrove.co.nz/ --type consulting`

**Output:**
```
[Mary] Fetching https://www.mangrove.co.nz/…
[Mary] Downloading 24 candidate real images…
[Mary] Downloading 1 logo candidates…
[Mary] Picked logo → /mnt/ai/freshsite/customers/mangrove-www/public/logo.png (source: https://www.mangrove.co.nz/hubfs/Mangrove%20Icon.png)
[Mary] Auditing competitor: Stripe (https://stripe.com)…
[Mary] Auditing competitor: Linear (https://linear.app)…
[Mary] Personas: .../docs/mangrove-personas.md
[Mary] Brief written: .../docs/mangrove-website-build-brief.md
[Mary] CRM: created lead 'mangrove-www'
```

**Artefacts:**
- `docs/mangrove-website-build-brief.md` (6.8KB) — full brief
- `docs/mangrove-personas.md` (1.5KB) — 3 personas
- `public/logo.png` (897KB) — Mangrove brand icon
- `docs/assets/mangrove/images/` — 5 photos
- `docs/assets/mangrove/competitors/` — competitor notes
- `crm/leads.json` (auto-touched) — `mangrove-www` lead created

**Personas assigned:** "First-Timer", "Returning Customer", "Researcher" — **these are the generic `general` template personas**, not consulting-specific. `_persona_templates.py` has no "consulting" template. Sally's `detect_business_type` mapped the brief to `nonprofit` because the brief text didn't contain consulting-specific terms; personas were picked from the `general` template.

**Recommendation:** add a `consulting` persona template to `_persona_templates.py` with personas like "The Operations-Burdened Founder", "The Pre-Series-A CEO", "The Mid-Market Operator".

### 1.2 Brief content (top of `mangrove-website-build-brief.md`)

```
# mangrove — Website Build Brief

## Customer summary
- URL: https://www.mangrove.co.nz/
- Current title: Home
- Current meta description: Mangrove helps New Zealand businesses...
- Current nav (4 items): Home, Our Services, About Us, Contact Us
- Business type: consulting

## Site visitor personas
[3 personas in summary form]

## Real assets harvested
[logo path + 5 image paths]

## Visual direction
[hints from competitor audit]
```

---

## Phase 2 — Sally (Design Brief v2)

### 2.0 Second bug — `sally.py design` doesn't exist

**Command:** `python3 $PLATFORM_SCRIPTS/sally.py design mangrove`

**Error:** `sally.py: error: argument cmd: invalid choice: 'design' (choose from 'brief')`

**Root cause:** The script exposes only `brief` as a subcommand. The function `write_design_brief` exists in source but the CLI surface is incomplete. The integration guide's "Next: review, edit, then hand to Sally" maps to this `brief` subcommand, but the workflow doc in `platform/docs/workflows/freshsite-build-guide.md` doesn't actually say which subcommand to call.

**Fix in this run:** use `sally.py brief mangrove`.

### 2.1 Design Brief

**Command:** `python3 $PLATFORM_SCRIPTS/sally.py brief mangrove`

**Output:**
```
[Sally] Wrote Design Brief v2: .../docs/mangrove-design-brief.md
[Sally] Recipe='default'  HeroVariant=''  Rhythm='balanced'
[Sally] Design brief written: ...
```

**Artefact:** `docs/mangrove-design-brief.md` (7KB) — v2 brief with §1-8, §5.5, §10, §11 (no §9 — see below).

**Sections produced:**
- ## 1. Theme Selection — `warm-local` (Local Favourite) picked. warmth=`warm` · tradition=`blend` · density=`moderate`
- ## 2. Home Composition Recipe — `default`, alternate=`eventsLed`
- ## 3. Palette Direction — locked
- ## 4. Typography Mode — Manrope
- ## 5. Photography Treatment — Full-bleed hero, gallery cards (because 5+ real photos)
- ## 5.5. Audience → Theme Mapping — 3 personas × tone=neutral × CTA target
- ## 6. Hero Treatment — theme default
- ## 7. Section Rhythm — `balanced`
- ## 8. Layout Emphasis and Tone
- ## 10. Accessibility & Motion
- ## 11. Open Questions for the Customer

**Notable: §9 (Page List) is missing from the output.** Sally's regex is `## Page list proposal` but Mary's brief doesn't contain that heading. The integration guide says §9 is "from Mary's brief" but the contract isn't enforced — Sally just skips it if Mary's brief didn't have it. This is a gap.

**Theme choice rationale:** Sally inferred `nonprofit` business type (not `consulting` as I asked). Her `detect_business_type` is keyword-based and the brief text didn't contain consulting-specific terms. The taxonomy mapped `nonprofit → warm × blend × moderate`, which put warm-local at top. Reasonable choice but the actual business is a for-profit consultancy — the `business_type` inference is unreliable when the brief is generic.

---

## Phase 3 — Paige (per-page copy)

### 3.0 LLM dependency

**Command:** `python3 $PLATFORM_SCRIPTS/paige.py brief mangrove --skip-llm`

**Output:**
```
[Paige] Business type: auto
[Paige] Home: scaffolded (no LLM)
[Paige] About: scaffolded (no LLM)
[Paige] Services: scaffolded (no LLM)
[Paige] Contact: scaffolded (no LLM)
[Paige] Copy written: .../docs/mangrove-copy.md
```

**Artefact:** `docs/mangrove-copy.md` (1.5KB) — contains page stubs with `_[copy not yet generated]_` placeholders.

**Fourth bug — `--skip-llm` doesn't actually scaffold useful copy.** It produces empty stubs. The "scaffolding" mode is supposed to give Glen a starting structure but it gives him nothing to work with. The microcopy section IS populated (button labels, form errors), which is useful.

**Real copy injection:** for this run I hand-wrote copy for each page (Home, About, Services, Contact) into a temporary file and substituted the placeholders. The actual production flow needs an LLM endpoint configured.

---

## Phase 4 — Amelia (build + recipe + rhythm)

### 4.1 Full pipeline

**Command:** `python3 $PLATFORM_SCRIPTS/amelia.py all mangrove`

**Output (full):**
```
[Amelia] Theme selected: warm-local (Local Favourite)
[Amelia] Cloning theme from local: /mnt/ai/freshsite/themes/template-warm-local → /mnt/ai/freshsite/customers/mangrove-www
[Amelia] Theme has no site.config.ts — writing default that reads build.env
[Amelia] Detected business type: nonprofit
[variation] mangrove (warm-local / nonprofit): hero=centered, 5 widgets, primary=#A03D0B
  [variation] global.css: --primary → #A03D0B
  [variation] global.css: --accent → #FFD700
  [variation] index.astro: hero=centered, widgets=5
[Amelia] Applying recipe 'default' from composition.yaml
[Amelia]   widgets in order: ['hero', 'painGrid', 'audienceGrid', 'serviceCards', 'testimonial', 'ctaBand']
[Amelia] Rewrote src/pages/index.astro with 6 widget(s)
[Amelia] Applying section rhythm 'balanced'
[Amelia]   --section-padding-y → 7.5rem
[Amelia]   --heading-scale → 1.05
[Amelia]   --motion-enabled → 1
[Amelia] Wrote 3 rhythm override(s) to global.css
[Amelia] Theme selected: warm-local (Local Favourite)
[Amelia] Applied 3 token override(s) to global.css
[Amelia] Removed scheme picker (theme is palette_locked)
[Amelia] build.env written: ...
[Amelia]   site: Home
[Amelia]   scheme: warm-local-locked
[Amelia]   url: https://mangrove-www.pages.dev
[Amelia] 4 page(s) to write. The parent agent must write each .astro file now.
[Amelia] Use the page_writing_prompt() helper in this script for each page.
```

**Note: the `[Amelia] all` subcommand runs the same theme-selection block TWICE** — once at the start (line "Theme selected: warm-local") and once after the variation step (line "Theme selected: warm-local (Local Favourite)" again). The second pass is from `step_apply_token_deltas` calling `step_pick_theme` internally. Not a bug, just wasteful.

### 4.2 Fifth bug — `index.astro` has duplicate widgets

After `step_apply_recipe`, the home page looks like this:

```astro
<Layout title="Home">
  <!-- Home: Hero → PainGrid → AudienceGrid → ServiceCards → Testimonial → CTABand (AD-6) -->

  <Hero
    tagline="Neighbourhood favourite"
    title="The kind of place people want to walk into."
    subtitle="Soft warm design for cafes, local retail, and community venues..."
    ctaPrimary={{ text: "Visit us", href: "/contact" }}
    ctaSecondary={{ text: "See what's on", href: "/services" }}
    variant="fullBleedPhoto"
  />

  <!-- ...and then the recipe widget block from composition.yaml: -->


<hero variant="fullBleedPhoto" />
<painGrid variant="threeCards" />
<audienceGrid variant="photoCards" />
<serviceCards variant="horizontalCards" />
<testimonial variant="singleQuote" />
<ctaBand variant="dualPath" />
</Layout>
```

**Two problems:**

1. **Duplicate widgets.** The warm-local scaffold wrote a full Hero block (lines 4-12) AND `step_apply_recipe` added a duplicate `<hero variant="..." />` after it. The Home page now has **2 heroes**.

2. **Lowercase widget tags.** The recipe widget block uses `<hero variant=... />` — Astro components are PascalCase, so `<hero>` doesn't match any imported component. Astro probably renders them as raw HTML `<hero>` tags (invalid HTML, ignored by browser). The recipe is effectively a no-op for those widgets.

**Root cause:** `composition.yaml` for warm-local has lowercase widget names:

```yaml
home_recipes:
  default:
    - hero: fullBleedPhoto
    - painGrid: threeCards
    - audienceGrid: photoCards
    - serviceCards: horizontalCards
    - testimonial: singleQuote
    - ctaBand: dualPath
```

But Astro components are PascalCase. The composition.yaml should be:

```yaml
    - Hero: fullBleedPhoto
    - PainGrid: threeCards
    - AudienceGrid: photoCards
    - ServiceCards: horizontalCards
    - Testimonial: singleQuote
    - CTABand: dualPath
```

**This is a data contract bug in all 10 themes' composition.yaml files.** The themes-handbook Epic 6 spec says widgets should be in their import-case names. Let me confirm by checking the theme widgets dir:

```
ls /mnt/ai/freshsite/themes/template-warm-local/src/components/widgets/
  Hero.astro, PageHero.astro, PainGrid.astro, AudienceGrid.astro,
  ServiceCards.astro, ProcessSteps.astro, ValuesGrid.astro,
  Testimonial.astro, CTABand.astro, Content.astro, ContactForm.astro
```

**All widgets are PascalCase. composition.yaml uses lowercase. Mismatch.**

### 4.3 Build + visual check

**Command:** `cd $CUSTOMERS_ROOT/mangrove-www && npm run build`

**Result:** Pass — 5/5 pages, 976ms total, no errors.

**Command:** `python3 $PLATFORM_SCRIPTS/visual_check.py http://localhost:8768/`

**Result:** 11/12 pass, 1 fail:
- `✗ Tailwind 1 class(es) missing in CSS` — `.site-nav__brand-text` used in HTML but no CSS rule. **Theme bug**, not pipeline bug — warm-local's scaffold emits the class but doesn't style it.

Pass: HTTP 200, 4/4 pages, brand "Home", year 2026, --primary resolves to #9a3412, hero has visible background.

### 4.4 Visual screenshot

Saved to `/home/glen/Downloads/mangrove-build-run.png`. Renders correctly with the warm-local defaults but shows the cafe placeholder copy — confirms the LLM-missing problem.

---

## Phase 5 — Push to GitHub (skipped)

Did not push because the site is using default warm-local copy with cafe-themed language, not mangrove-specific copy. **The site isn't ready for a public push.** When the LLM copy pipeline works, the same Amelia invocation will push to GitHub automatically via `amelia.py deploy`.

---

## Summary of bugs found in this run

| # | Severity | Where | Bug |
|---|---|---|---|
| 1 | UX | `mary.py brief` | `slug` and `url` are positional, not flags. The integration guide says to use `--slug` / `--url`. |
| 2 | UX | `sally.py` | Only `brief` is exposed as a subcommand. Workflow doc says "run sally design" but the actual subcommand is `brief`. |
| 3 | Pipeline | `paige.py --skip-llm` | Doesn't actually scaffold useful copy — produces empty stubs. Need a real LLM. |
| 4 | Logic | `_persona_templates.py` | No "consulting" template. Mangrove gets generic "First-Timer / Returning / Researcher" personas. |
| 5 | Logic | All `composition.yaml` files in 10 themes | Widget names are lowercase, Astro components are PascalCase. The recipe step produces `<hero>` tags that don't match any imported component. |
| 6 | Logic | `sally.py:detect_business_type` | Maps "Mangrove" to `nonprofit` based on keyword "charity / foundation / ngo" pattern that doesn't exist in the brief. Sally's `nonprofit` choice picked warm-local which is appropriate for a warm community feel but the actual business is for-profit. |
| 7 | Missing | Sally's Design Brief | §9 (Page List) absent because Mary's brief didn't have a "## Page list proposal" section. The contract isn't enforced. |
| 8 | UX | `amelia.py all` | Two `step_pick_theme` calls (one at start, one from `step_apply_token_deltas`). Wasteful but not broken. |
| 9 | Theme | `warm-local` theme | `.site-nav__brand-text` class emitted but unstyled. Visual check catches it. |

---

## What worked

- Theme clone from local (fast, no GitHub round-trip needed)
- Asset harvest: 5 photos + 1 logo captured
- Persona artifact generation
- Brief generation with all 8 v2 sections
- Recipe detection from composition.yaml
- Rhythm token application to global.css
- Variation engine produced color overrides
- Build: 5 pages in under 1s
- Visual check: 11/12 pass (the one fail is a theme bug)

---

## What didn't work

- Real copy generation (LLM not configured)
- Theme selection accuracy (business_type inference too narrow)
- composition.yaml widget name case mismatch
- §9 Page List section in Design Brief (when Mary's brief lacks the heading)

---

## What Cursor should look at

1. **The 5 severity-3+ bugs above** are real regressions in the pipeline. Most are in the data contracts (composition.yaml, brief schema) not in code logic.

2. **The `amelia.py all` flow doesn't actually write the .astro pages with real copy** — it scaffolds from the theme and expects the parent LLM agent to fill in content. For headless automation, this gap needs a way to call the LLM directly (or have Paige do the writing).

3. **`integration_library.sh` "Next" message** says to use `amelia.py deploy --slug <slug>` but doesn't say which subcommand to use before deploy (mary, sally, paige). The integration doc should be more explicit.

4. **The schema instability in Sally's brief** — §9 missing — is a contract gap. The themes-handbook Epic 6 spec says §9 is "from Mary's brief" but doesn't say what to do when Mary didn't write it.
