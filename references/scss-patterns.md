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

`frontend/styles.scss` provides the foundation that all themes build on. Theme stylesheets should not override these unless necessary.

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

Set by `utils.js:getCssRootProperties()`:

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
| `--image-resize` | `width` \| `height` \| `square` \| `disabled` | Image resize direction |
| `--text-position-options` | `top` `bottom` `left` `center` `right` | Available text positions |
| `--text-position-default` | e.g. `bottom-left` | Default text position |
| `--text-resizable` | `true` \| `false` | Enable title resize |
| `--text-resize-min` | e.g. `8pt` | Min font size |
| `--text-resize-max` | e.g. `128pt` | Max font size |
| `--text-font-size-field` | `titleFontSize` | Which adjustment field to update |
| `--text-line-height-field` | `titleLineHeight` | Line height adjustment field |
| `--text-affinity` | e.g. `bottom left` | Current text position (set by mixin) |
| `--movable` | e.g. `top left` | Movable axes |
| `--resizable` | e.g. `width height` | Resizable axes |
| `--rotatable` | `on` | Enable rotation |

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

Prefer approach 1 for clarity; use the mixin when you want automatic RGB conversion.

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
