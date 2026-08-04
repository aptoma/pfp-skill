# Schema Field Reference

This document describes every field the normalizer recognizes, its type, and its effect on the rendered HTML. Use it when defining content schemas for new customers.

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

Rules:

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

Declared on a front page item schema, naming the print item type it imports from. It does two things, both while the import list is built:

- **Gates the import list.** A print edition item is offered only if some front page item schema declares its type; unmatched print items are skipped entirely.
- **Selects the schema that maps the imported data.** `valueMap`, `valueTemplate`, `sectionPath`, `printSourceItemDataPath` and the HTML wrapping of `title` are all read off the **declaring** schema and applied once, at that moment. Nothing re-maps the data afterwards, so these annotations belong on the schema that declares `printSourceItemType` — putting them on the schema that ends up receiving the item has no effect.

**Each print front page product decides which schemas are considered, and the first match wins.** Content schemas are account-wide; a product names the item schemas it uses, and for a given type value DrEdition takes the first declaring schema among those, falling back to any `editionItem` schema in the account. A second schema declaring the same value is ignored *silently* — no error, no collision, just a mapping other than the one you intended.

**The receiving block decides the final item type, not this annotation.** An imported item arrives carrying the declaring schema. Dropping it into a block whose group `itemData.type` names a different schema swaps its `contentSchema` to that one and re-validates the data against it — unless the declaring schema sets [`preventGroupDefaults`](#preventgroupdefaults).

That re-validation **strips every field the receiving schema does not declare.** So a second page type — a table of contents, say — can be filled from print articles without declaring a `printSourceItemType` of its own, but only if it declares every field it needs to keep. A field imported and mapped through the generic schema vanishes on drop if the receiving schema has no property for it.

### printSourceItemDataPath

```json
{
  "x-dredition": {
    "printSourceItemDataPath": "item.data.frontPage"
  }
}
```

Imports a whole object of extra fields from the print item, beyond the native `title`, `pageRef`, `section`, `printSection` and `image` the import always handles. Use it when the print side already carries front-page-specific content — a kicker, an alternative summary — that the desk should not have to retype.

A dot-path resolved against `{item}`, so it starts with `item.`. Every key of the object found there becomes a top-level field on the imported front page item, and each value is HTML-wrapped exactly as `title` is: if the **declaring** schema types that property `"x-schema-form": {"type": "html"}` and the value is not already markup, it is wrapped in `<p>`. With the annotation above:

```json
{"data": {"frontPage": {"pretitle": "Kicker", "summary": "Front page summary"}}}
```

yields `pretitle: "Kicker"` and, for a `summary` typed `html`, `summary: "<p>Front page summary</p>"`.

Nothing is merged if the annotation is absent or the path resolves to nothing — no error either way, so a mistyped path fails silently.

**These fields merge last, overwriting the natives.** A key called `title`, `pageRef`, `section`, `printSection`, `image`, `id` or `type` in the imported object replaces the value the import just computed for it — including anything `valueMap` or `valueTemplate` produced. Name the keys to match the front page properties you intend to fill, and nothing else.

Like every imported field, these survive the drop into a block only if the receiving schema declares them — see [printSourceItemType](#printsourceitemtype).

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

Transforms imported values. The original print edition value is replaced with the mapped value as the import list is built. Read off the schema declaring `printSourceItemType` — see that section for why it has no effect on a receiving schema.

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

### itemLabel and itemLabelTemplate

Control how an item is labelled in the editor's import and edition lists, which
otherwise show only `type` and `id`. The two are alternatives, not composable — use
`itemLabel` for the default rendering, `itemLabelTemplate` when the text or styling
needs control.

```json
{
  "author": {
    "title": "Author",
    "type": "string",
    "x-dredition": {"itemLabel": true}
  }
}
```

`itemLabelTemplate` is an **object**, not a string, and every field is optional:

| Field | Default | Purpose |
|-------|---------|---------|
| `text` | `{{VALUE}}` | Label content |
| `title` | `{{TITLE}}` | Tooltip text |
| `color` | — | Text color (HTML color value) |
| `background` | — | Background color (HTML color value) |
| `sortPriority` | — | Numeric ordering of labels within an item |
| `hideInImportList` | `false` | Hide the label from the import list |
| `hideInEdition` | `false` | Hide the label from the edition list |
| `condition` | — | Expression; the label shows only when it evaluates true |

Placeholders: `{{VALUE}}` (the property's value), `{{TITLE}}` (the property's own
`title`), `{{ENUM_TITLE}}` (the title of the matching `oneOf` entry), `{{DATE}}` and
`{{CALENDAR_TIME}}` for date-valued properties.

```json
{
  "author": {
    "title": "Author",
    "type": "string",
    "x-dredition": {
      "itemLabelTemplate": {
        "title": "{{TITLE}}: {{VALUE}}",
        "text": "BY: {{VALUE}}",
        "color": "#6868ff",
        "background": "#000000"
      }
    }
  }
}
```

`condition` expressions read the item through `model`, with `$now` (epoch milliseconds)
and `$isoDate` (current time as an ISO string) available for time comparisons:

```json
{
  "factBoxes": {
    "title": "Fact Boxes",
    "type": "number",
    "x-dredition": {
      "itemLabelTemplate": {"condition": "model.factBoxes > 0"}
    }
  }
}
```

Full reference: [Content schemas — item labels](https://docs.aptoma.com/dredition/setup/print-automation/content-schemas#item-labels).

### preventGroupDefaults

```json
{
  "x-dredition": {
    "preventGroupDefaults": true
  }
}
```

When true, adding an item with this schema to a group skips the whole group-defaults step: the `itemData` defaults are not merged, the item's `contentSchema` is not swapped to the group's `itemData.type`, and the data is not re-validated against it. Used for special item types (e.g. freeplaced images) that should not inherit the block's default skin/type.

Because the swap and the re-validation are skipped together, this is also what preserves imported fields the group's schema does not declare — see [printSourceItemType](#printsourceitemtype).

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

**Convention:** add `"x-schema-form": {"type": "radios"}` to any enum with **4 or
fewer** options. Radios keep all choices visible at a glance, which suits the
small, fixed option sets typical of item schemas (skin, text position, image
position, alignment). Leave enums with 5+ options as the default dropdown to
avoid crowding the form.

## Schema Design Recommendations

1. **One generic schema with `printSourceItemType`** per print item type being imported. Include all fields you want to import, and put the `valueMap` / `valueTemplate` / `sectionPath` mappings here — this schema does the mapping for every item of that type.
2. **Specialized schemas without `printSourceItemType`** for specific skins/positions. Items reach them by being dropped into a block whose `itemData.type` names them, which strips any field they do not declare — so declare every imported field you need to survive the drop.
3. **Mark `adjustments` as `readonly`** — these are GUI-managed.
4. **Mark positional fields as `readonly`** in the group schema (x, y, width, height, layout).
5. **Use `modifiers` for styling hooks** that the user can toggle.
6. **Use `itemData` in group schema** to set default skin and type for items in that block.
7. **Keep `enum` values in lockstep with your SCSS.** Every value a modifier enum can emit must have a matching `.item--{key}-{value}` rule (or token) in the stylesheet, and each field's `default` must be one of its own `enum` values. Mismatches fail *silently*: the renderer still emits e.g. `item--gradient-size-60`, but with no rule to match, the property falls back to whatever the cascade provides instead of erroring — so a default outside the enum, or an enum value you never styled, looks fine in the schema yet renders wrong. Always use values present in the stylesheet.
8. **Use radio buttons for short enums** — add `"x-schema-form": {"type": "radios"}` to any enum with 4 or fewer options; leave longer enums as dropdowns.
9. **Every `adjustments` field the SCSS reads must exist in the content schema.** DrEdition only persists fields declared in the schema; anything else is held in the editor while the user works and dropped on save. A resize hook like `--text-font-size-field: titleFontSize` with no matching `adjustments.titleFontSize` property therefore looks fine in the editor and fails everywhere else: falls back to default in the page view, in the PDF, and on reopening the element. Debugging tell: *works in the editor, gone everywhere else → schema, not CSS.* Trap: a customer asset repo's `schemas/*.json` are examples only (that repo's `SCHEMAS.md` says changes there have no effect) and drift from the live schema configured in DrEdition — the reference copy can define the field while the live schema doesn't, so the property looks present in source when it isn't.
