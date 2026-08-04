# SCSS Styling Patterns

This document describes how to write SCSS stylesheets for Gazette print front pages.

## File Structure

### Asset Repository Layout

```
pfp-assets-{clientId}/
├── package.json
├── schemas/
│   └── editionItem/
│       ├── pfp-{client}-cover-story.json
│       ├── pfp-{client}-ref.json
│       └── pfp-{client}-story.json
├── static/
│   └── (logos, fonts, icons)
├── stylesheets/
│   ├── {theme}.scss              # Entry point (imports theme file)
│   ├── _mixins.scss              # Shared mixins across themes
│   └── themes/
│       └── {theme}/
│           ├── {theme}.scss      # Main theme styles
│           ├── _fonts.scss       # @font-face declarations
│           └── _graphik.scss     # (optional) font-specific partials
└── dist/                         # Built output (not committed)
```

The entry point `stylesheets/{theme}.scss` typically contains a single import:

```scss
@use './themes/{theme}/{theme}';
```

### Theme Scoping

Theme styles do not need to be scoped under a product class. The `.print-product--{NAME}` class (set on the `<body>` element, matching the DrEdition print product name) is available for product-specific overrides when multiple products share a theme — e.g., different brand colors:

```scss
// No scoping needed for shared styles
.block--logo {
  background-size: contain;
}

// Product-specific overrides
.print-product--DAILY {
  --color-brand: #005593;
}

.print-product--WEEKLY {
  --color-brand: #c4122f;
}
```

## Base Styles

`frontend/styles.scss` provides the foundation that all themes build on. Overriding
these is routine — sizing an image, changing the gap between items — but the base
selectors are compound and outrank the single-class selectors a theme naturally
reaches for. See [Beating the Base Rules](#beating-the-base-rules) before writing
any rule that targets `.item` or `.image`.

### Block Positioning

```scss
.block {
  position: absolute;
  z-index: 1;
  box-sizing: border-box;
  height: var(--block-height);
  width: var(--block-width);
  top: var(--block-top);
  left: var(--block-left);
}
```

### Layout Types

| Layout | Behavior |
|--------|----------|
| `.block--row` | `display: flex; flex-direction: row`. Items share width equally. `item--text-only` and `item--fixed-width` get `flex-grow: 0`. |
| `.block--column` | `display: flex; flex-direction: column`. Items share height equally. `item--text-only` and `item--fixed-height` get `flex-grow: 0`. |
| `.block--box` | Items are `position: absolute` with individual dimensions from CSS variables. |
| `.block--single` | Single item inherits block dimensions. |

### Image Handling

```scss
.image {
  flex-grow: 1;
  position: relative;
}

.image-content {
  position: absolute;
  top: 0; bottom: 0; width: 100%;
  background-position: var(--object-position-x, 50%) var(--object-position-y, 50%);
  background-size: var(--object-background-size, cover);
  background-repeat: no-repeat;
}
```

### Hidden Elements

These elements exist in the HTML for flexible placement via CSS:

```scss
.summary--alt  { display: none; }  // Duplicate summary outside .text
.caption--alt  { display: none; }  // Caption when no image
.summary .text-meta { display: none; }  // Text-meta copy inside summary
```

Show them selectively in your theme when needed.

**Pitfall — re-exposing the hidden copies.** The shared base hides these with
`display: none`, but your theme stylesheet loads *after* the base, so a broad
rule at equal-or-higher specificity silently un-hides them. The classic case:
setting `.text-meta { display: block }` (or `flex`) to stack section and
page-ref also overrides `.summary .text-meta { display: none }`, re-showing the
duplicate copy inside the summary. Every item with a `section` or `pageRef` then
renders it twice — once inside the summary, once standalone after the byline.

**Debugging tell:** this presents as a renderer bug, not a CSS one. The markup
really does contain two `.text-meta` blocks, and the base rule hiding one of them
is still there in the cascade — it just lost.

Either scope the rule to the standalone block, which never matches the hidden copy:

```scss
.text > .text-meta { display: flex; }
```

or restate the hide with a longer chain, so it wins on specificity rather than on
source order:

```scss
.item {
  .text-meta { display: flex; gap: 1ch; align-items: baseline; }
  .summary .text-meta { display: none; }   // 0-3-0, beats the 0-2-0 above
}
```

### Image/Text Ordering

Don't hardcode flex `order` on `.image` / `.text` to stack them. The
`imagePosition` field already controls their source order and tags the item with
`item--image-first` (`beforeText`) or `item--image-last` (`afterText`).
Hardcoding `order` overrides the editor's choice. Key any position-dependent
styling — e.g. which edge a separating margin sits on — off those classes
instead:

```scss
.item--image-first .image { margin-bottom: var(--baseline); }
.item--image-last  .image { margin-top: var(--baseline); }
```

### Beating the Base Rules

Most base rules are compound selectors. A theme rule on a single skin class
(`.item--skin-foo`, 0-1-0) loses to them, and loses *silently* — the layout is simply
the base's, with nothing to indicate which rule won. Match or exceed the specificity
shown:

| Base selector | Specificity | Declares |
|---|---|---|
| `.image` | 0-1-0 | `flex-grow: 1`, `max-width/height: 100%`, `width: var(--image-width, auto)`, `height: var(--image-height, var(--item-height, auto))` |
| `.block--row .item` | 0-2-0 | `flex: 1 1 auto`, `margin-right: var(--column-gap)`, `display: flex`, `flex-direction: row` |
| `.block--column .item` | 0-2-0 | `flex: 1 1 auto`, `margin-bottom: var(--baseline)` |
| `.block--column .image` | 0-2-0 | `height: auto`, `width: var(--item-width, auto)`, `flex-basis: auto`, `max-width/height: none` |
| `.block--single .item` | 0-2-0 | `width: var(--block-width)`, `height: var(--block-height)` |
| `.block--box .item` | 0-2-0 | `position: absolute` plus `--item-*` dimensions |
| `.block--column .item.item--text-only`, `.block--column .item.item--fixed-height` | 0-3-0 | `display: block`, `flex-grow: 0`, `flex-shrink: 0` |
| `.block--row .item.item--flex-column` | 0-3-0 | `flex-direction: column` |
| `.block--column .item.item--fixed-height .image` | 0-4-0 | `height: var(--image-height, auto)` |

**Three base rules run the other way — beating them *is* the bug.** The hide-rules for
the duplicate elements are weak on purpose, so a theme rule that merely styles the
element wins by accident and reveals the copy Gazette intends to suppress:

| Base selector | Specificity | Revived by |
|---|---|---|
| `.summary--alt` | 0-1-0 | any rule on `.summary--alt` — **and any `.summary` rule**, since the element carries both classes |
| `.caption--alt` | 0-1-0 | any rule on `.caption--alt` only; it does not carry `.caption`, so `.caption` rules are safe |
| `.summary .text-meta` | 0-2-0 | any `.item .text-meta` rule — equal specificity, theme loaded later |

The `.text-meta` case is the one that actually happens, because laying section and
page-ref out as a row is a normal thing to want. See [Hidden
Elements](#hidden-elements) above for the two ways to write it safely; do not reach for
the specificity table here, since the goal is to *lose* to these rules, not to win.

Three failure modes are worth calling out, because none of them looks like a
specificity problem:

**Items stretch to fill the block.** `flex: 1 1 auto` on `.block--row .item` and
`.block--column .item` means items grow past their content height. A theme that expects
content-height items has to override `flex-grow` at 0-2-0 or higher — or let the schema
mark the item `item--text-only` / `item--fixed-height`, which the base already gives
`flex-grow: 0`.

**Inter-item gaps revert to the base value.** `.block--column .item { margin-bottom:
var(--baseline) }` means a theme setting a different gap on `.item--skin-foo` gets
`--baseline` instead. The error is one gap-difference per item, so a 1pt discrepancy
accumulates down the column and survives visual checking. Same for `margin-right:
var(--column-gap)` in row blocks.

**Image heights break on some items only.** `item--fixed-height` is added *only* to
items with a stored `adjustments.imageHeight`, so the 0-4-0 rule applies per item and
the breakage looks random rather than systematic. The same adjustment writes
`--image-height` **inline on the item div**, which no stylesheet can outrank. A theme
that needs deterministic image sizing must therefore win on the `height` declaration at
0-4-0 or above; setting `--image-height` from a rule cannot work.

#### Routing per-skin values through a custom property

Where the value differs per skin, do not repeat the high-specificity selector for every
skin. Declare a custom property the base does not set, consume it once at a specificity
that beats the base, and let each skin set its value at ordinary specificity — a custom
property only competes with other declarations of itself:

```scss
// Consume once, above the base's 0-2-0
.block--column .item { margin-bottom: var(--item-gap, var(--baseline)); }

// Each skin sets its value at 0-1-0 — no specificity contest
.item--skin-toc-entry { --item-gap: 13pt; }
.item--skin-toc-lead  { --item-gap: 20pt; }
```

### Margin Collapse Differs Between Items

`.block--row` and `.block--column` are flex containers, and flex items never collapse
margins: adjacent margins sum. But the base sets `display: block` on `item--text-only`
and `item--fixed-height` inside a column block, so **those items collapse their
children's margins and their siblings do not**. An image's `margin-top` merges with the
previous item's `margin-bottom` — `max()` of the two rather than the sum — and the gap
silently shrinks.

Because `item--fixed-height` appears the moment a user drags an image height, and
`item--text-only` whenever an item has no image, this changes per item at runtime and
looks like an intermittent layout bug rather than a CSS one.

Use `padding` on the item for gaps that must hold under both, since padding does not
collapse:

```scss
.block--column .item {
  margin-bottom: 0;
  &:not(:last-child) { padding-bottom: var(--item-gap); }
}
```

The `:not(:last-child)` is not optional: the base zeroes the last item's *margin* at
0-3-0, so a bare `padding-bottom` would add trailing space inside the block that the
margin version never had.

The same applies to any theme rule that makes a block or item `display: block` —
including the multi-column pattern below.

## Available Mixins

### Bootstrap Mixins (`frontend/_mixins.scss`)

These mixins ship with the bootstrap asset repo that new customer setups are based on. They are also included in Gazette's `frontend/_mixins.scss` for the dev server.

#### `text-affinity()`

Positions the `.text` container based on the `item--text-pos-*` modifier:

```scss
.item--skin-cover-story {
  @include text-affinity();
}
```

Generates rules for six text positions:

| Modifier | Text Position | Alignment |
|----------|--------------|-----------|
| `item--text-pos-top-left` | `top: var(--text-vertical-offset); left: var(--text-horizontal-offset)` | Left |
| `item--text-pos-bottom-left` | `bottom: var(--text-vertical-offset); left: var(--text-horizontal-offset)` | Left |
| `item--text-pos-top-center` | `top: var(--text-vertical-offset); left: 0; right: 0; margin: 0 auto` | Center |
| `item--text-pos-bottom-center` | `bottom: var(--text-vertical-offset); left: 0; right: 0; margin: 0 auto` | Center |
| `item--text-pos-top-right` | `top: var(--text-vertical-offset); right: var(--text-horizontal-offset)` | Right |
| `item--text-pos-bottom-right` | `bottom: var(--text-vertical-offset); right: var(--text-horizontal-offset)` | Right |

The mixin also sets `--text-affinity` CSS variable, readable by the editor for resize controls.

#### `image-gradient()`

Adds directional gradient overlays that adapt to text position:

```scss
.item--skin-cover-story {
  @include image-gradient();
}
```

Requires modifiers in the item schema:

```json
{
  "gradient": { "type": "string", "enum": ["vertical", "horizontal"] },
  "gradient-size": { "type": "string", "enum": ["25", "50", "75", "100"] }
}
```

The gradient uses `--gradient-start`, `--gradient-end`, and `--gradient-size` CSS variables. Direction automatically adapts: gradient flows from text toward image center.

#### `cmyk($name, $c, $m, $y, $k, $a)`

Declares a CSS custom property with screen RGB and print CMYK values:

```scss
@include cmyk(--color-brand, 100%, 50%, 0%, 25%);
// Screen: --color-brand: rgba(0, 128, 255, 1)
// Print:  --color-brand: cmyk(1, 0.5, 0, 0.25)
```

### Customer-Level Mixins

Customer repos may define additional shared mixins in `stylesheets/_mixins.scss`, e.g. `text-affinity()` and `image-gradient()` variants with `--gradient-direction` support.

## CSS Custom Properties Reference

### Defined by the Normalizer/Renderer

Set on the item div via inline styles:

| Property | Source | Description |
|----------|--------|-------------|
| `--image-width` | `adjustments.imageWidth` | Constrains image width |
| `--image-height` | `adjustments.imageHeight` | Constrains image height |
| `--title-font-size` | `adjustments.titleFontSize` | Title font size (pt) |
| `--font-size` | `adjustments.titleFontSize` | On `.title` div only |
| `--line-height` | `adjustments.titleLineHeight` | On `.title` div only |
| `--text-horizontal-offset` | `itemData.textHorizontalOffset` | On `.text` div |
| `--text-vertical-offset` | `itemData.textVerticalOffset` | On `.text` div |
| `--header-horizontal-offset` | `itemData.headerHorizontalOffset` | On `.header` div |
| `--header-vertical-offset` | `itemData.headerVerticalOffset` | On `.header` div |
| `--object-position-x` | AOI focus or adjustment | On `.image-content` |
| `--object-position-y` | AOI focus or adjustment | On `.image-content` |
| `--object-background-size` | `adjustments.imageBackgroundSize` | On `.image-content` |
| `--caption-vertical-offset` | `caption.verticalOffset` | On `.image-caption` |
| `--caption-horizontal-offset` | `caption.horizontalOffset` | On `.image-caption` |

### Defined by Base Styles (`styles.scss`)

Set on blocks/items via the renderer:

| Property | Description |
|----------|-------------|
| `--block-width`, `--block-height`, `--block-top`, `--block-left` | Block dimensions |
| `--item-width`, `--item-height`, `--item-top`, `--item-left`, `--item-rotate` | Item dimensions |
| `--color-black-screen`, `--color-black-cmyk` | Black color for screen/print |
| `--text-color` | Inherited text color (screen vs print) |

### Defined on Body / Root

Set on the `<body>` element:

| Property | Description |
|----------|-------------|
| `--page-width`, `--page-height` | Full page dimensions |
| `--margin-top`, `--margin-right`, `--margin-bottom`, `--margin-left` | Page margins |
| `--bleed` | Bleed width |
| `--grid-width`, `--grid-height`, `--grid-top`, `--grid-left` | Content grid |
| `--grid-column-count`, `--grid-column-gap`, `--grid-column-gaps` | Column grid |
| `--grid-baseline`, `--grid-baseline-offset` | Baseline grid |
| `--front-page-object-width`, `--front-page-object-height` | Front page object dimensions |
| `--front-page-object-top`, `--front-page-object-left` | Front page object position |

### Defined from Item Modifiers

Any modifier key starting with `--` becomes a CSS custom property on the item div. Common patterns:

| Property | Purpose |
|----------|---------|
| `--gradient-start` | Gradient start color |
| `--gradient-end` | Gradient end color |
| `--gradient-size` | Gradient coverage percentage |

### Editor CSS Custom Properties

Read by the DrEdition preview editor to configure interactive controls:

| Property | Values | Purpose |
|----------|--------|---------|
| `--image-swap-direction` | `horizontal` \| `vertical` | Image swap icon placement |
| `--image-resize` | `width` \| `height` | Image resize handle axis |
| `--image-resize-direction` | `left` \| `right` | Which edge the handle sits on; defaults to an edge derived from item order and `item--image-last` |
| `--text-position-options` | `top` `bottom` `left` `center` `right` | Available text positions |
| `--text-position-default` | e.g. `bottom-left` | Default text position |
| `--text-resizable` | `true` \| `false` | Enable title resize |
| `--text-resize-min` | e.g. `8pt` | Min font size |
| `--text-resize-max` | e.g. `128pt` | Max font size |
| `--text-font-size-field` | `titleFontSize` | Which adjustment field to update |
| `--text-line-height-field` | `titleLineHeight` | Line height adjustment field |
| `--text-affinity` | e.g. `bottom left` | Current text position (set by mixin) |
| `--movable` | e.g. `top left` | Movable axes |
| `--resizable` | space-separated `width` `height` `square` | Resizable axes for the whole item (`square` belongs here, not on `--image-resize`) |
| `--rotatable` | `on` | Enable rotation |

**`--image-resize` is inherited, so a skin cannot opt out by omission.** The editor
stylesheet sets `--image-resize: height` on `.block--column` and `--image-resize: width`
on `.block--row`, and the value inherits down to every item in the block. Only `width` and `height` enable a handle, so
suppressing it means declaring some other value on the item — `--image-resize: none`.
Removing the declaration from the skin leaves the block's value in force and the handle
still there.

## Skin Styling Pattern

Skins are the primary styling mechanism. Each skin value produces an `item--skin-{name}` class.

The preferred pattern is to define typography using CSS custom properties on `.item`, then set values per skin at the `.item--skin-*` level. This keeps specificity flat and makes skins composable with modifiers.

```scss
.print-product--MY-PRODUCT {
  // Base: bind elements to CSS custom properties
  .item {
    .title {
      font-family: var(--title-font-family);
      font-weight: var(--title-font-weight);
      font-size: var(--title-font-size);
      line-height: var(--title-line-height);
    }
    .summary {
      font-family: var(--summary-font-family);
      font-weight: var(--summary-font-weight);
      font-size: var(--summary-font-size);
      line-height: var(--summary-line-height);
    }
  }

  // Skin overrides: set variables, not direct properties
  .item--skin-cover-story {
    --title-font-family: MyHeadlineFont;
    --title-font-weight: 700;
    --title-font-size: 66pt;
    --title-line-height: calc(var(--title-font-size) + 2pt);
    --summary-font-family: MyTextFont;
    --summary-font-weight: 400;
    --summary-font-size: 16pt;
    --summary-line-height: calc(var(--summary-font-size) + 5pt);

    .image {
      order: 1;
      flex-grow: 1;
    }
    .text {
      order: 2;
    }
  }

  .item--skin-ref {
    --title-font-family: MyHeadlineFont;
    --title-font-weight: 400;
    --title-font-size: 18pt;
    --title-line-height: calc(var(--title-font-size) + 4pt);
  }
}
```

## Modifier Styling Pattern

Modifiers from the item schema produce `item--{key}-{value}` classes (string) or `item--{key}` (boolean).

```scss
// String modifier: text-color with values black/white
.item--text-color-white {
  --text-color: var(--color-white);
  --gradient-start: rgba(0, 0, 0, 0.8);
  --gradient-end: rgba(0, 0, 0, 0);
}

.item--text-color-black {
  --text-color: var(--color-black);
  --gradient-start: rgba(255, 255, 255, 0.8);
  --gradient-end: rgba(255, 255, 255, 0);
}

// Boolean modifier: text-shadow
.item--text-shadow {
  .title, .pretitle {
    text-shadow: 0 0 16pt var(--text-shadow-color);
  }
}

// Nested: combine skin + modifier style
.item--skin-cover-story {
  &.item--style-oppslag {
    .title { font-family: PublicoHeadline-Medium; }
  }
  &.item--style-oppslag-xl {
    .title { font-family: Graphik; font-weight: 700; text-transform: uppercase; }
  }
}
```

## Block Name Selectors

Each block div has a `name` attribute set from the block's title in the DrEdition template. Use `[name='...']` attribute selectors to target specific template blocks:

```scss
[name='Logo'] {
  background-size: contain;
}

[name='Cover 1'] .title {
  font-size: 72pt;
}
```

This tight coupling to template block names is intentional — block names are stable identifiers defined in the template editor, not user-editable content. They serve as the primary mechanism for applying block-specific styles that go beyond what type/modifier classes provide.

## Threaded and Multi-Column Layouts

A design where an entry starts in one column and finishes in the next cannot be
expressed as separate blocks — Gazette's blocks are independent and lay out in
isolation. Use one wide block as a CSS multi-column fragmentainer instead, and let the
items flow through it.

Four things are required, and each fails silently if missed:

```scss
[name='ToC'] {
  display: block;              // a flex container is monolithic — items will not split
  columns: 2;
  column-fill: auto;           // the default, `balance`, equalises the columns instead
  overflow: hidden;            // see below

  .item {
    display: block;            // same reason, per item
  }

  .image {
    // .image-content is absolutely positioned, so a fragmented image paints
    // only its first fragment and leaves the rest blank
    break-inside: avoid;
    break-after: avoid;        // keep the image with its text
  }
}
```

**Overflow must be clipped, not left visible.** Overflowing a fixed-height multicol
makes PDFreactor break to a new page and continue there — on a front page format, a
spurious second page in the PDF. `overflow: hidden` clips instead. The cost is that the
clipped content is also invisible to DrEdition's overflow warning, which only reports
*visible* overflow, so the editor gets no signal that an entry was cut.

`column-span: all` works as expected for a lead item spanning both columns.

Note that making the block and its items `display: block` re-enables margin collapse
between an item and its children — see [Margin Collapse Differs Between
Items](#margin-collapse-differs-between-items).

## Print-Specific Considerations

### @media print

Use `@media print` for print-only overrides:

```scss
.print-product--MY-PRODUCT {
  // Screen colors (RGB)
  --color-brand: #005593;

  @media print {
    // Print colors (CMYK)
    --color-brand: cmyk(100%, 50%, 0%, 25%);
    --divider-width: 0.25pt;  // Thinner lines for print
    --border-width: 0.25pt;
  }
}
```

### PDFreactor Properties

```scss
.image-content {
  -ro-image-resampling: 220dpi;          // Resample images for print
  -ro-image-recompression: jpeg(85%);     // Compress JPEGs
}

// Don't resample PNG images (logos, graphics)
.image-content[style*='.png'] {
  -ro-image-resampling: none;
  -ro-image-recompression: none;
}

.content-area {
  -ro-pdf-overprint: mode1;               // Enable overprint for CMYK
  -ro-pdf-overprint-content: mode1;
}
```

**Overprint:** The content area enables CMYK overprint by default (`mode1`). This is correct for black text over images, but non-black text printed over images will render incorrectly with overprint enabled (the underlying image bleeds through the text color). Disable overprint on elements with non-black text over images:

```scss
// Disable overprint for logo and other non-black-on-image elements
.block--logo {
  -ro-pdf-overprint: none;
  -ro-pdf-overprint-content: none;
}
```

### Unsupported CSS

**`writing-mode` is not supported by PDFreactor.** Rotate text with `transform` only. The failure is silent: correct in the browser preview, unrotated or wrongly rotated in the PDF.

This is more than a find-and-replace, because `writing-mode` participates in layout (the element reserves box space in the rotated flow) while `transform` does not (a rotated element keeps its unrotated box). A rotated caption or credit needs:

- a width-reserving parent sized for the *unrotated* text
- the rotated text absolutely positioned inside that parent
- `white-space: nowrap` (there's no rotated flow to wrap into)
- an explicit `transform-origin`

Also note that `align-items` centers on a different axis once `writing-mode` is removed — re-check any flex alignment on the parent.

```scss
.image-credit {
  position: absolute;
  white-space: nowrap;
  transform-origin: 0 0;
  transform: rotate(270deg);
}
```

Don't reach for `writing-mode: vertical-rl` and then patch it with an added `transform: rotate()` — the two don't compose, and PDFreactor ignores the `writing-mode` half regardless. Use `transform` alone as shown above.

### Baseline Snapping Does Not Survive the Default Block Layouts

PDFreactor's `-ro-line-grid` / `-ro-line-snap` snap text to a shared baseline grid, and
its manual recommends them for multi-column containers. They do not work in a Gazette
theme as shipped: the grid propagates only through **block containers**, and `.item` is
`display: flex` — so the chain from `.content-area` breaks at every item and each one
ends up snapped to a grid of its own, at whatever phase it happens to start. The symptom
is items that each look internally correct while drifting relative to each other.

Snapping works only where the whole chain from the grid-creating ancestor down to the
text is `display: block` — in practice, the multi-column blocks above.

Two further properties of these declarations:

- Creating a grid is inert until something snaps to it, so `-ro-line-grid: create` can be
  declared broadly and the effect scoped by where `-ro-line-snap` is applied.
- No browser implements either property, so the DrEdition preview will not match the PDF.
  Verify baseline alignment in the PDF only.

### CMYK Colors

Two approaches:

**1. CSS Custom Properties with @media print:**

```scss
.print-product--MY-PRODUCT {
  --color-brand: #005593;          // Screen (RGB)

  @media print {
    --color-brand: cmyk(100%, 50%, 0%, 25%);  // Print (CMYK)
  }
}
```

**2. The `cmyk()` mixin:**

```scss
@include cmyk(--color-brand, 100%, 50%, 0%, 25%);
// Generates RGB for screen, CMYK for print automatically
```

Prefer approach 1 when exact RGB and CMYK values are both available (e.g. from a design spec or brand guidelines) — it produces the most accurate screen and print colors. The mixin is a convenience shortcut when only CMYK values are known, as it approximates the RGB conversion automatically.

### Font Embedding

```scss
@font-face {
  font-family: MyFont;
  font-style: normal;
  font-weight: 700;
  src: url('https://smooth-storage.aptoma.no/users/{org}/files/assets/static/fonts/MyFont/MyFont-Bold.otf');
  -ro-font-embedding-type: all;   // PDFreactor: embed full font
}
```

Fonts are hosted on Smooth Storage. The `-ro-font-embedding-type: all` directive ensures PDFreactor embeds the complete font in the PDF.

### Page Size and Bleed

Page size is set in the inline `<style>` by the handler, not in theme CSS. The theme should not set `@page` rules.

For bleed handling, see the bleed section in the end-user docs (`var/print-front-page/examples-explanations.md`).

## Annotated Walkthrough: Cover Story Theme

This walkthrough covers a representative cover story skin with full-bleed image-with-text-overlay design.

### 1. Theme Variables

```scss
.print-product--MY-PRODUCT {
  --baseline: 11pt;
  --column-gap: 4mm;
  --color-section: #005593;
  --color-accent-orange: #f5851f;
  --color-accent-red: #e30513;
  --font-serif: 'PublicoHeadline-Medium', serif;
  --title-font: PublicoHeadline-Bold, serif;
  --logo: url('...logo.svg');

  @media print {
    --color-section: cmyk(100%, 50%, 0, 25%);
    --color-accent-orange: cmyk(0%, 58%, 100%, 0);
    --logo: url('...logo.pdf');  // PDF for print
  }
```

Note how logos use SVG for screen and PDF for print, ensuring optimal rendering in both contexts.

### 2. PDFreactor Image Settings

```scss
  .image-content {
    -ro-image-resampling: 220dpi;
    -ro-image-recompression: jpeg(85%);
  }

  .image-content[style*='.png'] {
    -ro-image-resampling: none;
    -ro-image-recompression: none;
  }
```

### 3. Modifier-Driven Color Variables

```scss
  .item--section-color-blue  { --section-color: var(--color-section); }
  .item--section-color-red   { --section-color: var(--color-accent-red); }

  .item--text-color-white {
    --text-color: var(--color-white);
    --gradient-start: var(--color-black-80-alpha);
    --gradient-end: var(--color-black-zero-alpha);
  }

  .item--text-color-black {
    --text-color: var(--color-black);
    --gradient-start: var(--color-white-80-alpha);
    --gradient-end: var(--color-white-zero-alpha);
  }
```

Modifiers set CSS variables rather than direct styles. This means the same gradient mixin works regardless of text color — it reads from the variables.

### 4. Cover Story Skin

```scss
  .item--skin-cover-story {
    --title-font-size: 66pt;
    --title-font-default: PublicoHeadline-Medium;

    // Style variant: standard
    &.item--style-standard {
      .title {
        font-family: var(--title-font, var(--title-font-default));
        font-size: var(--title-font-size);
      }
    }

    // Style variant: extra large
    &.item--style-xl {
      --title-font-size: 78pt;
      .title {
        font-family: Graphik;
        font-weight: 700;
        text-transform: uppercase;
      }
    }

    .header {
      background: var(--color-black);
      color: var(--color-white);
      font: 900 22pt/20pt Graphik;
      text-transform: uppercase;
      padding: 2mm 8mm 1mm;
      padding-inline: var(--header-horizontal-offset, 8mm);
    }

    .title {
      --text-resizable: true;
      --text-font-size-field: titleFontSize;
      color: var(--text-color);
    }

    .image {
      margin-bottom: var(--baseline-x2);
      order: 1;          // Image first visually
      flex-grow: 1;
    }

    .text {
      order: 2;          // Text second visually

      .summary {
        color: var(--summary-color);
        font: normal 16pt/21pt PublicoText-Roman;
        margin-top: calc(var(--baseline-x2));
      }

      .text-meta {
        .section { display: none; }  // Hide section in cover story
        .page-ref::before {
          content: 'Page ';          // Localized page prefix
        }
      }
    }
  }
```

Key patterns demonstrated:
- **CSS custom property cascading**: `--title-font-size` set at skin level, overridden per style variant
- **CSS var fallback chains**: `var(--title-font, var(--title-font-default))` for overridable defaults
- **Flex ordering**: `.image { order: 1 }` / `.text { order: 2 }` to control visual order independently of HTML source order
- **Editor integration**: `--text-resizable: true` and `--text-font-size-field` enable live title resizing in the preview editor
- **Pseudo-content for labels**: `.page-ref::before { content: 'Page '; }` avoids hardcoding text in the data

### 5. Text-on-Image Items

For items where text overlays the image, the theme uses:

```scss
  .item--skin-cover-story {
    &.item--text-on-image {
      .text {
        position: absolute;
        z-index: 2;
        width: var(--text-width, auto);
      }

      .image {
        position: absolute;
        top: 0; bottom: 0; width: 100%; height: 100%;
      }
    }

    @include text-affinity();     // Position .text based on text-pos modifier
    @include image-gradient();    // Add gradient overlay adapting to text position
  }
```

This converts the item from a flex layout to an overlay layout, positioning the text absolutely over the image.
