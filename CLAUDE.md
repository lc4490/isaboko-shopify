# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository

Custom Shopify theme for **Isaboko** — a Brooklyn-based clothing brand making pieces from vintage kimonos. The theme is built on **Shopify Dawn** with brand-specific overrides layered on top. Liquid + CSS + vanilla JS; no build step, no package manager.

## Common commands

```bash
shopify theme dev      # Local preview with hot reload (requires Shopify CLI + store auth)
shopify theme push     # Upload to a connected store
shopify theme pull     # Download theme files from a connected store
shopify theme check    # Run theme-check linter
```

No JS bundler, no formal test suite. Formatting is Prettier via `.prettierrc.json` (Dawn defaults).

## Brand system (locked)

- **Colors** — Black `#000000`, Red `#E8000F`, Lime Green `#CAFF00`, White `#FFFFFF`. These are the only brand tokens. Greys come from opacity on black, not new colors.
- **Typography** — System sans stack (`-apple-system, BlinkMacSystemFont, "Segoe UI", "Inter", Roboto, ...`). Forced via [`--font-body-family` and `--font-heading-family` in assets/isaboko.css](assets/isaboko.css), which override Dawn's Shopify-font-picker output because `isaboko.css` is loaded after `base.css` in [layout/theme.liquid](layout/theme.liquid). Headings are weight 800.
- **Nav** — Fixed structure: `HOME · SHOP · CUSTOM · WORLD · PATCH CLUB`. Header is a single black bar with `HOME SHOP CUSTOM` left of a centered red **ISABOKO** wordmark, and `WORLD PATCH CLUB` to the right. The wordmark is `{{ shop.name }}` styled via `.header--middle-center .header__heading-link .h2` in `isaboko.css` — merchant must set the shop name to "ISABOKO" in Shopify admin for it to render correctly.
- **Pages in scope** — Homepage, Product Category, Product Detail, Product Variations, World/Archive, Fabric, Cart drawer. Don't scaffold pages outside this list without confirming.

## Architecture

### Color schemes (the primary theming knob)

Dawn drives all section/component theming through **color schemes** defined in [config/settings_data.json](config/settings_data.json). The five schemes have been rewritten for Isaboko's palette — do NOT change these without authorization:

| Scheme  | Background | Text     | Button   | Use                                      |
| ------- | ---------- | -------- | -------- | ---------------------------------------- |
| scheme-1| `#FFFFFF`  | `#000000`| `#000000`| Default body / product pages             |
| scheme-2| `#000000`  | `#FFFFFF`| `#FFFFFF`| Header, dark sections, footer            |
| scheme-3| `#E8000F`  | `#FFFFFF`| `#000000`| Red accents, sale badges, callouts       |
| scheme-4| `#CAFF00`  | `#000000`| `#000000`| Lime accents, secondary callouts         |
| scheme-5| `#000000`  | `#E8000F`| `#E8000F`| Black-with-red — wordmark / brand motif  |

Sections pick their scheme via the `color_scheme` setting (visible in the theme editor as a color swatch). The CSS variables (`--color-background`, `--color-foreground`, `--color-button`, etc.) are emitted per scheme by Liquid in [layout/theme.liquid:57-95](layout/theme.liquid) and applied via `.color-scheme-N` body/section classes.

When introducing new brand colors, add them to the existing five schemes — don't create a 6th. Dawn's settings_schema and many sections hard-reference these five.

### Brand layer

[assets/isaboko.css](assets/isaboko.css) is the **single sanctioned place for brand overrides**. Anything that should apply site-wide (font stack, heading defaults, wordmark color, header layout) goes here. It loads after `base.css` so its rules win on equal specificity.

Don't edit `base.css` to apply brand changes — it would create merge pain if Dawn is ever updated. Add to `isaboko.css` instead.

### Header (customized)

The header uses Dawn's `middle-center` logo position but renders a **split nav** — first half of `main-menu` links left of the logo, second half right of it. Implementation:

1. [sections/header.liquid](sections/header.liquid) sets `isaboko_split_nav = true` when `logo_position == 'middle-center'` and `menu_type_desktop == 'dropdown'`, then renders [snippets/header-dropdown-menu.liquid](snippets/header-dropdown-menu.liquid) twice with `menu_offset` / `menu_limit` / `menu_position` params (`left` and `right`).
2. [snippets/header-dropdown-menu.liquid](snippets/header-dropdown-menu.liquid) was extended to accept those params (with sane defaults so calling `{% render 'header-dropdown-menu' %}` still works). IDs use `link.handle` instead of `forloop.index` to avoid collisions across the two renders.
3. [assets/isaboko.css](assets/isaboko.css) overrides Dawn's `.header--middle-center` grid from `'navigation heading icons'` (3 columns) to `'left-menu logo right-menu icons'` (4 columns).

The nav structure is configured in Shopify admin under **Navigation → main-menu** — the theme references it by handle, it isn't defined in the theme files.

### Shopify-theme folder roles (Dawn standard)

- `layout/` — top-level page wrappers; `theme.liquid` is the default.
- `templates/` — JSON files composing sections into pages.
- `sections/` — full-width, schema-driven Liquid components.
- `snippets/` — reusable Liquid fragments invoked with `{% render %}`.
- `config/` — `settings_schema.json` (settings UI) + `settings_data.json` (current values, **including color schemes**).
- `assets/` — CSS, JS, SVGs served as theme assets.
- `locales/` — translation files. Storefront strings come from `*.json`; theme-editor strings from `*.schema.json` (referenced via `t:...` keys in section schemas).

## Conventions

- **Schema-driven styling** — Single-property settings render as inline CSS variables (`style="--gap: {{ block.settings.gap }}px"`); multi-property settings render as a CSS class on the element. Both consumed inside `{% stylesheet %}` blocks. (Dawn convention, also stated in README.md.)
- **CSS variables as RGB triplets** — Dawn stores colors as `R, G, B` (no `rgb()` wrapper) so they can be used with opacity via `rgba(var(--color-foreground), 0.5)`. New brand tokens in `isaboko.css` follow the same convention (`--isaboko-red: 232, 0, 15`).
- **Component CSS files** — Many Dawn sections load their own `component-*.css` from `assets/`. When customizing a component, prefer overriding in `isaboko.css` over editing the component CSS in place.
- **No new color tokens** — All theme work should derive from the four brand colors. If a section needs an off-shade, use `rgba(var(--isaboko-black), 0.6)` etc., don't introduce `--isaboko-grey-1`.
- **Don't edit Dawn files for brand changes** — Override in `isaboko.css`. The only Dawn files modified for Isaboko so far are: `config/settings_data.json` (palette), `sections/header-group.json` (header config), `sections/header.liquid` (split nav wiring), `snippets/header-dropdown-menu.liquid` (slice params), `layout/theme.liquid` (one `<link>` for isaboko.css).
