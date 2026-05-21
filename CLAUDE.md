# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Source of truth

**DESIGN_SPEC.md is the locked design specification** — referenced throughout the codebase and this document. It covers brand identity, color/typography/spacing scales, every page, components, buttons, breakpoints, animation, and an explicit "what not to do" list. The user has said: do not deviate from the spec. **Note:** DESIGN_SPEC.md may not be checked into the repo (it was authored externally). If it's absent, rely on the implemented values in `assets/isaboko.css` and the "Locked user overrides" section below as the effective spec.

This document covers the implementation plumbing that DESIGN_SPEC.md doesn't — how Dawn is configured, where brand overrides live, and how to add new sections without breaking the architecture.

## Repository

Custom Shopify theme for **Isaboko** — a Brooklyn-based brand making one-of-a-kind pieces from vintage kimonos. Built on **Shopify Dawn** with brand overrides layered on top. Liquid + CSS + vanilla JS; no build step, no package manager.

## Common commands

```bash
shopify theme dev      # Local preview with hot reload (requires Shopify CLI + store auth)
shopify theme push     # Upload to a connected store
shopify theme pull     # Download theme files from a connected store
shopify theme check    # Run theme-check linter
```

No JS bundler, no formal test suite. Formatting via `.prettierrc.json` (Dawn defaults). `.theme-check.yml` disables `MatchingTranslations` and `TemplateLength` checks.

## Architecture

### Brand-override entry point

[assets/isaboko.css](assets/isaboko.css) is the **single sanctioned place for brand overrides**. It's loaded after `base.css` from [layout/theme.liquid:259](layout/theme.liquid#L259) so its rules win on equal specificity. Do not edit `base.css` to apply brand changes — it would create merge pain if Dawn is ever updated. The file is organized by spec section: §2 colors, §3 typography, §5 buttons, §6 top bar, §7 header + mega menu, §24 flat, plus sections for each custom component (FOOTER, HERO, CATEGORY TILES, PATCH CLUB, PROCESS, LOOKBOOK, NEWSLETTER, PRODUCT TABS). Search for the `§` or all-caps section comment to jump to the right block.

### Two CSS-variable conventions (deliberate)

| Convention | Where | Shape | Consume as |
|---|---|---|---|
| **Brand tokens** | `--isaboko-*` in `isaboko.css` | hex (`#E8000F`) — per spec §2 | `color: var(--isaboko-red)` |
| **Dawn tokens** | `--color-*` etc. in `theme.liquid` | RGB triplet (`232, 0, 15`) | `color: rgb(var(--color-foreground))` or `rgba(..., 0.5)` |

Don't mix them — `rgb(var(--isaboko-red))` will produce `rgb(#E8000F)` which is invalid CSS. The brand tokens are direct hex because the spec writes them that way.

### Color schemes (Dawn's section-level theming)

Dawn lets each section pick a `color_scheme` setting that maps to one of five schemes in [config/settings_data.json](config/settings_data.json). The schemes have been rewritten for Isaboko's palette:

| Scheme  | Background | Text     | Typical use                                |
| ------- | ---------- | -------- | ------------------------------------------ |
| scheme-1| `#FFFFFF`  | `#000000`| Default body / product pages               |
| scheme-2| `#000000`  | `#FFFFFF`| Footer, dark sections                      |
| scheme-3| `#E8000F`  | `#FFFFFF`| **Header (per spec §7)**, red callouts     |
| scheme-4| `#CAFF00`  | `#000000`| Newsletter bar, lime accents               |
| scheme-5| `#000000`  | `#E8000F`| Logo-on-black wordmark motif as a scheme   |

Each section's chosen scheme drives Dawn's `--color-background`, `--color-foreground`, `--color-button`, etc. on the section wrapper.

### Top bar (custom section)

[sections/isaboko-topbar.liquid](sections/isaboko-topbar.liquid) is a **static centered banner** (black background, white 13px text, ~36px height). It replaces Dawn's `announcement-bar` in [sections/header-group.json](sections/header-group.json). Dawn's [sections/announcement-bar.liquid](sections/announcement-bar.liquid) is left on disk but no longer referenced. CSS lives in `isaboko.css` under "§6 TOP BAR". Note: DESIGN_SPEC.md §6 originally specified a scrolling ticker — the user overrode this to a single static centered line on 2026-05-18. Don't reintroduce marquee/animation without explicit instruction.

### Header

Uses **Dawn's stock middle-center layout** — nav links on the left of the logo, icons (search/account/cart/currency) on the right. Configured in `header-group.json`:

- `color_scheme: "scheme-3"` → red bar (#E8000F) with white text
- `logo_position: "middle-center"` → Dawn's built-in centered-logo grid
- `menu_type_desktop: "mega"` → activates [snippets/header-mega-menu.liquid](snippets/header-mega-menu.liquid) per spec §7 mega-dropdown requirement
- `enable_country_selector: true` → renders "UNITED STATES (USD)" on the right per spec
- `padding_top: 24`, `padding_bottom: 24` → taller bar matching the user's reference image

The "logo badge" (black rectangle with **lime border and lime `★ ISABOKO ★`** inside) is pure CSS — the Liquid still renders `{{ shop.name }}`, and `.header__heading-link .h2` styles it as a black-background span with lime text and a 2px lime border. The `★` characters come from `::before` / `::after` so the merchant can keep `shop.name = "ISABOKO"` in admin. (Note: DESIGN_SPEC.md §7 originally specified red text + bullets; lime stars + border is a locked user override from 2026-05-18.)

**Note:** earlier work split the nav into left/right halves around the logo. That was wrong per spec §7 (which puts all nav links left of the logo). Those customizations were reverted on 2026-05-18. `snippets/header-dropdown-menu.liquid` is pristine Dawn; **`sections/header.liquid` was re-edited later** (also 2026-05-18) to (a) render `isaboko-mega-menu` in the `menu_type_desktop == 'mega'` branch and (b) add a `mega_menu_image` block type — see "Mega menu" below.

### Mega menu (custom — replaces Dawn's text-only one)

Spec §7 requires editorial image slots in each top-level dropdown (SHOP / CUSTOM / WORLD). Dawn's `snippets/header-mega-menu.liquid` is pure text columns with no image support, so it's been **superseded** by [snippets/isaboko-mega-menu.liquid](snippets/isaboko-mega-menu.liquid) (Dawn's file is left on disk untouched as a fallback).

How images get into the dropdowns:
1. The header section in [sections/header.liquid](sections/header.liquid) exposes a `mega_menu_image` block type (limit 9). Each block has: `parent_link_handle` (select: shop / custom / world), `image`, `heading`, `link`.
2. In the theme editor, the merchant adds one block per image and picks which dropdown it shows in.
3. `isaboko-mega-menu.liquid` filters `section.blocks` by `block.settings.parent_link_handle == link.handle` and renders the matching blocks in a `.isaboko-mega-menu__promos` list to the right of the text columns.

The handle-match relies on Shopify's auto-generated handle for each top-level Navigation link (title `SHOP` → handle `shop`). If a merchant manually customizes a link's URL or alters the title, the handle changes and images stop appearing — the block schema's help text flags this.

### Top-level nav is hardcoded in the snippet

`linklists.main-menu` is **intentionally ignored** by `isaboko-mega-menu.liquid`. The five top-level items live in a parallel-array data block at the top of the snippet:

```
HOME       /                       (flat)
SHOP       /collections/all        (dropdown +)
CUSTOM     /pages/custom           (dropdown +)
WORLD     /pages/world             (dropdown +)
PATCH CLUB /products/patch-club    (flat)
```

To change the top-level nav, edit those arrays. To change the sub-menu URLs/items inside SHOP / CUSTOM / WORLD, edit the placeholder section in the same file (currently a `<p class="isaboko-mega-menu__placeholder">`).

The merchant still controls editorial images via **Customize theme → Header → Add block → Mega menu image** (configured per dropdown via `parent_link_handle` matching `shop` / `custom` / `world`).

### Shopify-theme folder roles (Dawn standard)

- `layout/` — top-level page wrappers; `theme.liquid` is the default.
- `templates/` — JSON files composing sections into pages.
- `sections/` — full-width, schema-driven Liquid components.
- `snippets/` — reusable Liquid fragments invoked with `{% render %}`.
- `config/` — `settings_schema.json` (settings UI) + `settings_data.json` (current values, **including color schemes**).
- `assets/` — CSS, JS, SVGs served as theme assets.
- `locales/` — translation files. Storefront strings come from `*.json`; theme-editor strings from `*.schema.json` (referenced via `t:...` keys in section schemas).

### Footer (custom section)

[sections/isaboko-footer.liquid](sections/isaboko-footer.liquid) replaces Dawn's stock footer. Wired in [sections/footer-group.json](sections/footer-group.json). White background, 2px black top border, "ISABOKO" wordmark, 6-column nav grid, social icons, red bottom bar with copyright + currency. CSS lives in `isaboko.css` under "FOOTER". Dawn's `sections/footer.liquid` is left on disk but no longer referenced.

### Homepage sections (custom)

`templates/index.json` composes these Isaboko sections in order:

| Section file | Purpose |
|---|---|
| `isaboko-hero.liquid` | Split-layout hero with image slideshow (left) + text/CTA (right). Vanilla JS for slide transitions. |
| `featured-collection` | Dawn's built-in featured collection (scheme-2, dark background). |
| `isaboko-category-tiles.liquid` | Category tile grid (NEW / BAGS / JACKETS / ACCESSORIES). Block-driven. |
| `isaboko-patch-club.liquid` | Patch Club promo section with heading, description, CTA. |
| `isaboko-process.liquid` | "What we do" mission section with description + CTA. |
| `isaboko-lookbook.liquid` | Photo grid of past collections. Block-driven (each block = one photo tile with label + link). |
| `isaboko-newsletter.liquid` | Lime full-width strip with Shopify customer email signup form. |

### Product tabs (custom snippet)

[snippets/isaboko-product-tabs.liquid](snippets/isaboko-product-tabs.liquid) replaces Dawn's collapsible accordions on the PDP. Four tabs: DETAILS / FABRIC / SIZING / CARE. Content pulled from product metafields with fallbacks. Expects `product` in scope.

## Build order (per spec §23)

CSS vars + reset → top bar → **header/nav** → footer → homepage → product category → product detail → cart drawer → variations → world → fabric → patch club. Phases 1–6 (vars, top bar, header, footer, homepage, and product tabs) are done. The next phases are product category grid, full product detail page, and cart drawer.

## Conventions

- **Schema-driven styling** — single-property settings render as inline CSS variables (`style="--gap: {{ block.settings.gap }}px"`); multi-property settings render as a CSS class on the element. Both consumed inside `{% stylesheet %}` blocks. (Dawn convention, also stated in `README.md`.)
- **Component CSS files** — Many Dawn sections load their own `component-*.css` from `assets/`. When customizing a component, prefer overriding in `isaboko.css` over editing the component CSS in place.
- **No new color tokens** — All theme work derives from the six brand colors in DESIGN_SPEC.md §2. If an off-shade is needed, use opacity on an existing token, not a new one.
- **No rounded corners, no shadows** — Spec §24. Dawn's settings already drive radius/shadow values to 0 — don't reintroduce them in component styles.
- **Dawn files modified for Isaboko**: `config/settings_data.json` (palette), `sections/header-group.json` (header config + topbar wiring), `sections/footer-group.json` (swapped Dawn footer for `isaboko-footer`), `layout/theme.liquid` (one `<link>` for isaboko.css), `sections/header.liquid` (renders `isaboko-mega-menu` snippet in the mega branch + adds `mega_menu_image` block type). `snippets/header-dropdown-menu.liquid` is pristine Dawn — don't customize it; override in CSS instead.

## Locked user overrides (deviations from DESIGN_SPEC.md)

These were locked by the user on 2026-05-18 based on a reference screenshot. They supersede the spec on the listed points. Don't revert them unless the user explicitly asks.

| What | Spec said | Locked override |
|---|---|---|
| **Top bar** (§6) | Scrolling ticker, text repeated 3×, uppercase | Static centered single line, merchant-typed casing preserved |
| **Logo badge** (§7) | Black bg, white-or-red text, bullet `•` flanking | Black bg, **lime text**, **2px lime border**, **★ stars** flanking |
| **Nav link typography** (§7) | 13px / 600 / 0.08em | **16px / 300 / 0.05em** (localization selector: 16px / 700 / 0.05em) |
| **Dropdown trigger** (§7) | Click to open | **Hover to open** (click is keyboard/touch fallback) |
| **Cart count bubble** | Scheme-3 default (black-on-white) | **Lime background, black text** |
| **Nav underlines** | Not specified | **Never underline nav links** in any state (idle, hover, active, open). Use weight, color, or `+`/`−` glyph instead. |
| **Header padding** | 20/20 | **24/24** (taller bar per reference image) |
| **Top bar text size** (§6) | 12px / 0.1em spacing | **13px / 0.04em spacing** |
