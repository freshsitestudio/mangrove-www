# Local Favourite — Technical Spec

<!-- Handbook contract: docs/THEME_DEVELOPER_HANDBOOK.md §18 -->

## CSS Custom Properties

All 18 fixed token names (AD-4). Names are immutable; these are the Warm Local locked-palette defaults.

```css
:root {
  --bg:              #FFF7ED;
  --bg-alt:          #FEF3C7;
  --bg-dark:         #431407;
  --surface:         #FFFFFF;
  --text:            #1C1A18;
  --text-muted:      #706B66;
  --primary:         #9A3412;
  --primary-dark:    #7C2D12;
  --accent:          #84CC16;
  --accent-warm:     #F59E0B;
  --border:          rgba(154, 52, 18, 0.10);
  --gradient-start:  #FFF7ED;
  --gradient-end:    #FEF3C7;
  --nav-height:      4rem;
  --radius:          1rem;
  --radius-lg:       1.25rem;
  --shadow:          0 4px 12px rgba(0, 0, 0, 0.08);
  --shadow-lg:       0 12px 40px -8px rgba(0, 0, 0, 0.12);
}
```

Token deltas for this theme vs strict base (from `references/theme-validation.md §6`):

| Token | Value |
|---|---|
| `--bg` | `#FFF7ED` |
| `--primary` | `#9A3412` |
| `--accent` | `#84CC16` |

## Google Fonts

CDN only — no self-hosted fonts. Loaded in `Layout.astro`:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;500;600;700;800&family=Lora:wght@400;500;600&display=swap" rel="stylesheet">
```

**Families shipped:** Nunito (400–800) for headings and UI; Lora (400–600) for body copy.

## Color Scheme Selector

**NO** — Palette is locked. `warm-local` sets `palette_locked: true` in handbook metadata. Do **not** ship a scheme picker, `[data-scheme]` CSS blocks, `DEFAULT_SCHEME` in `build.env`, or `localStorage` scheme persistence.

## Component Props

All 11 required widgets documented below. Themes may add optional props; this documents the actual shipped interface.

### Hero.astro

```typescript
interface Props {
  tagline?: string;
  title: string;
  subtitle?: string;
  ctaPrimary?: { text: string; href: string };
  ctaSecondary?: { text: string; href: string };
}
```

Soft warm gradient hero — `linear-gradient(180deg, var(--gradient-start), var(--gradient-end))`; primary CTA uses `--accent` (lime) with dark text `#1C1A18` for WCAG AA.

### PageHero.astro

```typescript
interface Props {
  title: string;
  subtitle?: string;
}
```

### PainGrid.astro / AudienceGrid.astro / ValuesGrid.astro

```typescript
interface Props {
  items: { title: string; description: string }[];
}
```

### ServiceCards.astro

```typescript
interface Props {
  services: {
    name: string;
    tagline: string;
    price?: string;
    features: string[];
  }[];
}
```

### Testimonial.astro

```typescript
interface Props {
  quote: string;
  author: string;
  role?: string;
  company?: string;
}
```

### CTABand.astro

```typescript
interface Props {
  title: string;
  subtitle?: string;
  ctaPrimary: { text: string; href: string };
  ctaSecondary?: { text: string; href: string };
}
```

### ProcessSteps.astro

```typescript
interface Props {
  steps: { number: number; title: string; description: string }[];
}
```

### ContactForm.astro

```typescript
interface Props {
  phone?: string;
  address?: string;
}
```

### Content.astro

```typescript
interface Props {
  title: string;
  body: string; // HTML string — use set:html
}
```

## Build Commands and Stack

- **Framework:** Astro 7 (`^7.0.7`)
- **CSS:** Tailwind v4 via `@tailwindcss/vite` registered in `vite.plugins` — **not** legacy `@astrojs/tailwind`
- **Node:** `>=22.12.0` (enforced in `engines.node`)
- **CSS import:** frontmatter `import "../styles/global.css"` in `Layout.astro` — no `public/styles/` copy step
- **Fonts:** Google Fonts CDN `<link>` in `Layout.astro` only — not in `build.env`, not self-hosted
- **Deploy:** Cloudflare Pages via git-push — no Wrangler CLI in theme repo
- **Config:** `build.env` committed public config (identity/nav only); `npm run build` and `npm run dev` source it via `[ -f build.env ] && set -a && . ./build.env && set +a; astro build`. Layout reads `SITE_*` / `NAV_*` from `process.env` (shell-sourced — not Vite `PUBLIC_*`).
- **No:** `@astrojs/tailwind`, Wrangler, Netlify Forms, self-hosted fonts, `data-netlify`, scheme picker

## Contact Form Contract

```html
<form name="contact" method="POST" action="/contact">
  <!-- fields… -->
</form>
```

Required attributes: `name="contact"`, `method="POST"`, `action="/contact"`.

Also required (§10 / §18 contract):

- Visible `<label>` elements with `for` pointing to each input `id`
- `required` attribute on all mandatory fields
- Native HTML5 `required` validation (no `novalidate` — Round 42 native fallback); JS enhances whitespace-only / invalid-email cases with `aria-invalid` + focus-first when submit fires
- No `data-netlify`, no Netlify Forms, no other third-party form backends

## Implementation Notes

- **Mobile nav:** Must have solid background on mobile expansion — no transparent overlays. Background is `var(--bg)` (warm off-white) on the dropdown panel. Collapse sets `display: "none"`; resize resets `aria-expanded`.
- **Scheme picker absent:** `warm-local` is palette-locked from day one — no picker shipped. No `[data-scheme]` CSS, no picker UI, no `DEFAULT_SCHEME` in `build.env`.
- **Soft warm hero:** Hero uses a gentle warm gradient (`--gradient-start` → `--gradient-end`) — not `linear-gradient()` with cool blues. Flat heritage cream is reserved for `thorndon-traditional`.
- **Lime accent CTAs:** `--accent: #84CC16` is for primary CTAs and highlights only. Dark text `#1C1A18` on lime clears WCAG AA (white-on-lime fails 4.5:1).
- **Rounded geometry:** `--radius: 1rem` and `--radius-lg: 1.25rem` — inviting, community-feel cards and buttons.
- **ValuesGrid / PainGrid / AudienceGrid prop name:** Always `items[]` — reject `values[]` / `cards[]` patterns.
- **Contact page structure:** `PageHero → ContactForm` only. Do **not** insert `ContactInfo` between them.
- **About page Content widget:** `Content.astro` uses `set:html` to render a body HTML string. Keep body content as reusable placeholder copy.
- **`focus-visible` outlines:** All interactive elements inherit `outline: 2px solid var(--primary)` from `global.css`.
- **WCAG AA:** `--text-muted: #706B66` on `--bg` (≈ 4.96:1) and `--bg-alt` (≈ 4.73:1) meets 4.5:1 minimum.
- **Decorative images:** Use `alt=""` for purely decorative images; provide meaningful alt text for content images.
- **Typography split:** Nunito for `h1`–`h6` and UI; Lora on `body`. Logo link uses Nunito explicitly (weight ≤ 800).
