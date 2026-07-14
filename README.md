# Local Favourite

<!-- Handbook contract: docs/THEME_DEVELOPER_HANDBOOK.md §18 -->

## Purpose

Local Favourite (`warm-local`) is a fixed-palette, friendliness-first, soft-warm FreshSite Studio theme for cafes, local retail, community organisations, family businesses, markets, and neighbourhood venues. It translates approachable community connection into a website: warm off-white surfaces, rounded geometry, Nunito + Lora typography, and a tone that says "I like them and want to visit" — not heritage reliability or cool contemporary polish.

## Fit

- Cafes, bakeries, and neighbourhood food venues
- Local retail and family-run shops
- Community organisations and markets
- Health-adjacent and family services where comfort drives conversion
- Clients who need to feel human, grounded, and easy to approach

## Anti-fit

- Heritage trades and barbershops where reliability matters more than likability (`thorndon-traditional` leads with "I can trust them")
- Cool sparse advisor aesthetics (`minimal-professional`)
- B2B procurement and infrastructure clients (`corporate-tech`)
- Consumer brands wanting gradient hero aesthetics (`fresh-modern`)
- Clients who need multiple swappable colour schemes — this theme locks the palette

## Visual Personality

Friendly and community-feel — this theme reads like a neighbourhood spot you want to walk into, not a corporate brochure or heritage shopfront. Soft warm off-white hero surfaces, Nunito rounded headlines, Lora body copy, and lime accent CTAs signal approachability and warmth. No consumer blue gradients; the hero is a soft warm gradient from cream-orange to warm yellow-white with inviting rounded cards.

## Colour Palette

Locked warm palette: off-white background `#FFF7ED`, brick primary `#9A3412`, stone muted `#78716C`, and lime accent `#84CC16` reserved for CTAs and highlights only. **No colour scheme picker** — `palette_locked: true`.

Colours that feel wrong here: cool blues, stark corporate greys, dark-mode neon, heritage walnut creams, high-formality serif-only layouts.

## Typography

**Heading font:** Nunito (Google Fonts CDN, weights 400–800). Rounded sans for friendly headlines and section titles.

**Body font:** Lora (Google Fonts CDN, weights 400–600). Warm serif for readable body copy; Nunito also used for UI labels.

## Layout Notes

- **Hero:** Full-width soft warm gradient hero (`--gradient-start: #FFF7ED` → `--gradient-end: #FEF3C7`) — approachable, not flat heritage cream or blue gradient. Responsive text sizing via `clamp()`.
- **Navigation:** Sticky top bar with solid warm off-white background (`--bg`) — no transparent overlays on mobile. Hamburger toggle on viewports < 768px with a solid warm dropdown. **No scheme picker.**
- **Card grids:** CSS `auto-fit / minmax` — rounded corners (`--radius: 1rem`), warm borders; adapts from 1 to 3 columns.
- **CTA Band:** Full-width brick primary band; white text. Primary nav and hero CTAs use lime accent with dark text `#1C1A18` for WCAG AA (white-on-lime fails).
- **Palette locked:** Unlike `fresh-modern`, this theme does not ship a colour scheme selector. Tokens are fixed in `:root` only.

## Widgets Available

Required library set (11; no ContactInfo):

1. Hero — top of the home page (soft warm gradient, approachable)
2. PageHero — interior page header
3. PainGrid — customer-problem cards
4. AudienceGrid — audience/persona cards
5. ServiceCards — service offerings (optional pricing)
6. Testimonial — quote with attribution
7. CTABand — full-width CTA banner
8. ValuesGrid — values/principles grid (`items[]` prop)
9. ProcessSteps — numbered process section
10. ContactForm — contact form (phone/address via props if needed)
11. Content — structured story/content block (About page)

Optional extensions: none shipped in this theme.

## Sections on Each Page

| Page | Sections (in order) |
|------|---------------------|
| Home | Hero → PainGrid → AudienceGrid → ServiceCards → Testimonial → CTABand |
| About | PageHero → Content → ValuesGrid → ProcessSteps → CTABand |
| Services | PageHero → ServiceCards → CTABand |
| Contact | PageHero → ContactForm |
| 404 | Minimal error section |

<!-- Contact is NOT PageHero → ContactInfo → ContactForm. About MUST include Content. -->

## Pricing Display

YES — ServiceCards accepts an optional `price` prop. Shown as a prominent heading (e.g. "From $12") above the feature list. Pricing display is optional per service; omit the prop to hide it.

## Mobile Nav

Hamburger icon at < 768px viewport width. Links expand into a vertical stack with a **solid warm off-white background** (`var(--bg)`) and bottom border — no transparent or see-through overlays. `aria-expanded` toggled on the button; collapse sets `display: "none"`; resize resets `aria-expanded`. Nav links are keyboard-accessible.

## Live Reference Site

No FreshSite live deploy at baseline. Visual and typography references:

- https://www.allbirds.com (mood reference — approachable product warmth; not client content)
- https://fonts.google.com/specimen/Nunito
- https://fonts.google.com/specimen/Lora
