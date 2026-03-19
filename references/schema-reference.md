# Schema Field Reference

This document describes every field the normalizer recognizes, its type, and its effect on the rendered HTML. Use it when defining content schemas for new customers.

See also: `types/gazette.d.ts` in the Gazette repository for TypeScript type definitions.

## Item Schema Fields

### Text Content Fields

These fields produce HTML divs within the rendered item. Empty/falsy values produce no output.

| Field | Type | CSS Class | Container | Notes |
|-------|------|-----------|-----------|-------|
| `header` | `string` | `.header` | Top-level (above `.text`) | Supports `headerHorizontalOffset` and `headerVerticalOffset` |
| `pretitle` | `string` | `.pretitle` | Inside `.text` | Plain text or HTML |
| `title` | `string` | `.title` | Inside `.text` | Supports rich text with `<strong>`, `<em>`. Gets `--font-size` and `--line-height` from adjustments |
| `subheadline` | `string` | `.subheadline` | Inside `.text` | Also known as subtitle in some schemas |
| `summary` | `string` | `.summary` | Inside `.text` | Supports rich text. Duplicated as `.summary--alt` outside `.text` for flexible placement |
| `byline` | `string` | `.byline` | Inside `.text` | |
| `section` | `string` | `.section` | Inside `.text-meta` | Also produces item modifier `item--section-{slugified}` |
| `printSection` | `string` | `.print-section` | Inside `.text-meta` | Section prefix from the print page plan |
| `pageRef` | `string` | `.page-ref` | Inside `.text-meta` | Page reference, e.g. "Side 4" |

### Control Fields

These fields modify rendering behavior without producing their own content div.

| Field | Type | Effect |
|-------|------|--------|
| `skin` | `string` | Adds `item--skin-{value}` class. Central styling hook. |
| `skinVariants` | `object` | If `skinVariants[skin]` exists, adds `item--skin-{skin}-{variant}`. |
| `textPosition` | `string` enum | Adds `item--text-pos-{value}`. Values: `auto`, `top-left`, `top-right`, `bottom-left`, `bottom-right`, `top-center`, `bottom-center`. |
| `imagePosition` | `string` enum | `beforeText`: image div before text div + `item--image-first`. `afterText` (default): image after text + `item--image-last`. |
| `summaryPosition` | `string` enum | `beforeTitle`: reorders content items so summary/byline/textMeta precede pretitle/title/subheadline. `afterTitle` (default): standard order. |
| `textMetaPosition` | `string` enum | Controls where `.text-meta` block renders. See [html-structure.md](html-structure.md#textmetaposition-variants) for all variants. |
| `textMetaInline` | `boolean` | Legacy: when true, appends text-meta inline at end of summary. Superseded by `textMetaPosition`. |
| `showCaption` | `boolean` | When `false`, suppresses caption rendering. Default: `true`. |

### Offset Fields

These produce CSS custom properties on their respective containers.

| Field | Type | CSS Property | Container |
|-------|------|-------------|-----------|
| `textHorizontalOffset` | `string` | `--text-horizontal-offset` | `.text` |
| `textVerticalOffset` | `string` | `--text-vertical-offset` | `.text` |
| `headerHorizontalOffset` | `string` | `--header-horizontal-offset` | `.header` |
| `headerVerticalOffset` | `string` | `--header-vertical-offset` | `.header` |

Values should be valid CSS lengths, e.g. `'4mm'`, `'2%'`.

### Modifiers Object

```json
{
  "modifiers": {
    "type": "object",
    "properties": {
      "text-color": { "type": "string", "enum": ["white", "black"] },
      "text-on-image": { "type": "boolean" },
      "--gradient-start": { "type": "string" }
    }
  }
}
```

Rules (from `normalizer.js:getModifiers()`):

| Modifier Type | Schema | Value Example | Result |
|--------------|--------|---------------|--------|
| Boolean (true) | `"text-on-image": { "type": "boolean" }` | `true` | CSS class `item--text-on-image` |
| Boolean (false) | same | `false` | Omitted (no class) |
| String | `"text-color": { "type": "string" }` | `"white"` | CSS class `item--text-color-white` |
| CSS variable | `"--gradient-start": { "type": "string" }` | `"rgba(0,0,0,0.8)"` | Inline style `--gradient-start: rgba(0,0,0,0.8)` on item div |

Keys starting with `--` are **not** converted to classes. They are extracted by `getCssProperties()` and set as inline CSS custom properties.

### Field-Level Property Classes

Any top-level schema property matching the pattern `{field}--{qualifier}` produces additional classes on that field's div.

Example:
```json
{
  "title--weight": { "type": "string", "enum": ["bold", "light"] }
}
```

With value `"bold"` → the `.title` div gets class `title title--weight-bold`.

If the field content contains `<strong>` tags, the class `{field}--has-strong` is automatically added.

### Image Object

```json
{
  "image": {
    "type": "object",
    "format": "image",
    "properties": {
      "url":          { "type": "string", "format": "imageUrl" },
      "thumbnailUrl": { "type": "string", "format": "imageThumbnailUrl" },
      "width":        { "type": "number", "format": "imageWidth" },
      "height":       { "type": "number", "format": "imageHeight" },
      "id":           { "type": "string", "format": "imageId" },
      "aoi":          { "type": "object", "format": "areaOfInterest" }
    }
  }
}
```

The `format` values are DrEdition conventions that enable the image picker UI and automatic image handling. The `aoi` (area of interest) object contains:

| Field | Type | Purpose |
|-------|------|---------|
| `aoi.x` | `number` | AOI rectangle x position (pixels) |
| `aoi.y` | `number` | AOI rectangle y position |
| `aoi.width` | `number` | AOI rectangle width |
| `aoi.height` | `number` | AOI rectangle height |
| `aoi.focus.x` | `number` | Focus point x (pixels). Used to derive `--object-position-x` |
| `aoi.focus.y` | `number` | Focus point y. Used to derive `--object-position-y` |
| `aoi.origin` | `string` | Origin of the AOI data (`'user'`, `'auto'`) |

The focus point is converted to a percentage of image dimensions for CSS `background-position`. Adjustments (`imagePositionX`, `imagePositionY`) override the focus-derived values.

### Caption Object

```json
{
  "caption": {
    "type": "object",
    "properties": {
      "text":             { "type": "string" },
      "credit":           { "type": "string" },
      "position":         { "type": "string", "enum": ["top-left", "top-right", "bottom-left", "bottom-right"] },
      "skin":             { "type": "string" },
      "verticalOffset":   { "type": "string" },
      "horizontalOffset": { "type": "string" }
    }
  }
}
```

Renders as:
```html
<div class="image-caption image-caption--{position} image-caption--skin-{skin}"
     style="--caption-vertical-offset: {verticalOffset}; --caption-horizontal-offset: {horizontalOffset}">
  {text}
  <div class="image-credit">{credit}</div>
</div>
```

Defaults: position = `bottom-left`, skin = `default`.

Caption is suppressed when `showCaption` is `false`, or when both `text` and `credit` are empty.

### Adjustments Object

```json
{
  "adjustments": {
    "type": "object",
    "readonly": true,
    "properties": {
      "titleFontSize":       { "type": "number" },
      "titleLineHeight":     { "type": "number" },
      "imageWidth":          { "type": "string" },
      "imageHeight":         { "type": "string" },
      "imagePositionX":      { "type": "string" },
      "imagePositionY":      { "type": "string" },
      "imageBackgroundSize": { "type": "string", "default": "cover" }
    }
  }
}
```

These values are set through the preview GUI (drag to resize title, drag to reposition image). Mark as `readonly` to prevent manual editing.

| Field | CSS Property | Element | Side Effect |
|-------|-------------|---------|-------------|
| `titleFontSize` | `--font-size: {n}pt` | `.title` style | Also sets `--title-font-size` on item |
| `titleLineHeight` | `--line-height: {n}` | `.title` style | |
| `imageWidth` | `--image-width` | Item style | Adds `item--fixed-width` class |
| `imageHeight` | `--image-height` | Item style | Adds `item--fixed-height` class |
| `imagePositionX` | `--object-position-x` | `.image-content` style | Overrides AOI focus |
| `imagePositionY` | `--object-position-y` | `.image-content` style | Overrides AOI focus |
| `imageBackgroundSize` | `--object-background-size` | `.image-content` style | Default: `cover` |

### Refs Array

Alternative to the standard text-meta for items that reference multiple articles:

```json
{
  "refs": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "title":   { "type": "string" },
        "pageRef": { "type": "string" }
      }
    }
  }
}
```

When `refs` is present and non-empty, the `.text-meta` block is replaced with:

```html
<div class="text-refs">
  <div class="text-ref">
    <div class="text-ref-title">{title}</div>
    <div class="text-ref-page">{pageRef}</div>
  </div>
  ...
</div>
```

### Positional Fields (Freeplaced Items)

For movable/freeplaced items (e.g. freeplaced images):

| Field | Type | CSS Property |
|-------|------|-------------|
| `x` | `number` | `--item-left: {x}mm` |
| `y` | `number` | `--item-top: {y}mm` |
| `width` | `number` | `--item-width: {width}mm` |
| `height` | `number` | `--item-height: {height}mm` |
| `rotate` | `number` | `--item-rotate: {rotate}deg` |

### Editor Capability Flags

| Field | Type | Effect |
|-------|------|--------|
| `movable` | `boolean` | Sets `data-movable` attribute on item div |
| `resizable` | `boolean` | Sets `data-resizable` attribute |
| `rotatable` | `boolean` | Sets `data-rotatable` attribute |

### Direct Content Fields

| Field | Type | Effect |
|-------|------|--------|
| `content` | `string` | Raw HTML. Bypasses normal text/image rendering entirely. |
| `assetUrl` | `string` | Renders as `<img src="{assetUrl}" alt=""/>`. Bypasses normal rendering. |

## Group Schema Fields

```json
{
  "type": "object",
  "properties": {
    "title":     { "type": "string", "default": "block" },
    "layout":    { "type": "string", "enum": ["single", "row", "column", "box"] },
    "type":      { "type": "string" },
    "x":         { "type": "number" },
    "y":         { "type": "number" },
    "width":     { "type": "number" },
    "height":    { "type": "number" },
    "modifiers": { "type": "object" },
    "itemData":  { "type": "object" }
  }
}
```

| Field | Effect |
|-------|--------|
| `title` | Block name. Used for processor matching (`block.name`). Must be unique per template. |
| `layout` | Determines block layout type and CSS: single, row, column, box. |
| `type` | Content type. Becomes first modifier and sets `block.contentType`. Used for processor matching. |
| `x`, `y` | Position in mm from content area origin. |
| `width`, `height` | Dimensions in mm. |
| `modifiers` | Same modifier rules as items. Produces `block--{modifier}` classes. Auto-includes `items-{count}`. |
| `itemData` | Default values applied to items when added to this group. Common: `type`, `skin`, `modifiers`. |

Most group fields should be `readonly` since they are managed by the template editor. The `modifiers` object is the exception — it can include user-editable fields.

## x-dredition Extensions

Schema-level annotations for DrEdition integration.

### printSourceItemType

```json
{
  "x-dredition": {
    "printSourceItemType": "print-article-item"
  }
}
```

Maps print edition items to front page items. When a print edition item has a schema matching this value, it becomes available for import into the front page. Only define on the generic/default item schema; omit from specialized schemas.

### valueMap

```json
{
  "section": {
    "type": "string",
    "x-dredition": {
      "valueMap": {
        "Nyheter": "Nyhet",
        "Kultur og underholdning": "Kultur"
      }
    }
  }
}
```

Transforms imported values. The original print edition value is replaced with the mapped value when the item is added to the front page.

### valueTemplate

```json
{
  "pageRef": {
    "type": "string",
    "x-dredition": {
      "valueTemplate": "Side {{VALUE}}"
    }
  }
}
```

Wraps the imported value in a template. `{{VALUE}}` is replaced with the actual value (e.g. page number `4` becomes `Side 4`).

### sectionPath

```json
{
  "x-dredition": {
    "sectionPath": "page.properties.section"
  }
}
```

Overrides the default section source (`item.data.section`). Dot-path notation can access `page` and `item` objects from the print edition.

### preventGroupDefaults

```json
{
  "x-dredition": {
    "preventGroupDefaults": true
  }
}
```

When true, the group's `itemData` defaults are not applied to items with this schema. Used for special item types (e.g. freeplaced images) that should not inherit the block's default skin/type.

## x-schema-form UI Hints

### HTML Editor

```json
{
  "title": {
    "type": "string",
    "x-schema-form": {
      "type": "html",
      "quillOptions": {
        "modules": {
          "toolbar": [["bold", "italic", "clean"]]
        },
        "formats": ["bold", "italic"]
      }
    }
  }
}
```

Renders a Quill rich text editor. Set `toolbar: false` and `formats: []` for plain HTML (no formatting toolbar).

### Radio Buttons

```json
{
  "imagePosition": {
    "type": "string",
    "enum": ["beforeText", "afterText"],
    "x-schema-form": {
      "type": "radios"
    }
  }
}
```

Renders enum values as radio buttons instead of a dropdown.

## Schema Design Recommendations

1. **One generic schema with `printSourceItemType`** for importing articles. Include all fields you want to import.
2. **Specialized schemas without `printSourceItemType`** for specific skins/positions. These inherit the import mapping from the generic schema.
3. **Mark `adjustments` as `readonly`** — these are GUI-managed.
4. **Mark positional fields as `readonly`** in the group schema (x, y, width, height, layout).
5. **Use `modifiers` for styling hooks** that the user can toggle.
6. **Use `itemData` in group schema** to set default skin and type for items in that block.
7. **Limit `enum` values** to what you actually style in your SCSS.
