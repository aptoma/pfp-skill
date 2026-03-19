# Processor Development Guide

Processors inject dynamic content (edition info, logos, dates) into rendered blocks. Each customer theme can have a processor; most do.

## Function Signature

```javascript
// src/lib/processors/{name}.js
exports.process = (context, block, content, attributes) => {
    // Modify content and/or attributes
    return [content, attributes];
};
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `context` | `ProcessorContext` | Edition and product data |
| `block` | `Block` | The normalized block being rendered |
| `content` | `string` | Rendered HTML of the block's elements |
| `attributes` | `BlockAttributes` | Class, style, name, lang for the block div |

**Return value:** Always `[content, attributes]`. Both must be returned even if only one was modified.

## Context Object

Assembled in `server/handlers.js:renderStatic()`:

```javascript
{
    theme: 'my-theme',                     // Set by getProcessor()
    printEdition: {
        data: {                           // Edition content schema values
            issueNumber: 12,
            price: '49',
            barcode: 'weekday',
            volume: 130,
            // ... any fields from edition content schema
        },
        publishDate: '2025-03-15'         // ISO date string
    },
    printProduct: {
        name: 'sa-print',                 // Product identifier
        data: {                           // Product content schema values
            startYear: 1868,
            homepage: 'www.sa.no',
            tagline: 'Your daily newspaper',
            // ... any fields from product content schema
        }
    }
}
```

The `data` objects are free-form — their shape depends on the edition and product content schemas configured in DrEdition.

## Block Matching Strategies

Processors typically check `block.name` or `block.contentType` to decide which blocks to modify.

### By block name (most common)

```javascript
if (block.name === 'Logo') {
    // inject logo and edition info
}
```

This is the most common matching strategy.

### By block name prefix

```javascript
if (block.name.startsWith('Issue')) {
    // inject issue info for any Issue-* block
}
```

Useful when multiple blocks share a prefix (e.g. `Issue`, `Issue Info`).

### By content type

```javascript
if (block.contentType === 'flag') {
    // inject edition info into flag blocks
}
```

Useful when the content type is more reliable than block names (e.g. flag is a group type).

### Multiple block names

```javascript
switch (block.name) {
    case 'Logo':
    case 'Logo Magasin':
    case 'Logo Magasin Hvit':
        // handle all logo variants
        break;
}
```

Useful when the same product has multiple logo variants with different styling.

## Common Operations

### Appending HTML to content

```javascript
content += `<div class="edition-info">
    <div class="date">${formattedDate}</div>
    <div class="issue">${printEdition.data.issueNumber}</div>
</div>`;
```

The content is the HTML of all elements inside the block. Appending adds markup after the existing items.

### Replacing content

```javascript
content = `<div class="edition-info">...</div>`;
```

Replaces all element content. Used when the block's items are just placeholders.

### Adding CSS classes

```javascript
attributes.class = attributes.class + ' block--logo';
attributes.class = `${attributes.class} block--issue-info`;
```

Always append with a space. The class string already contains `block block--{type} block--{modifiers}`.

### Lifting item modifiers to block

```javascript
block.elements.forEach(({modifiers}) => {
    modifiers.forEach((modifier) => {
        attributes.class = attributes.class + ' block--' + modifier;
    });
});
```

Promotes item modifiers to block classes, enabling logo color variants via CSS.

### Setting language

```javascript
attributes.lang = 'de';
```

Sets the `lang` attribute on the block div. Used when the content language differs from the document default.

## Edition/Product Data Fields

These fields come from DrEdition content schemas configured per customer. Common patterns:

### Edition data (`printEdition.data`)

| Field | Type | Example | Purpose |
|-------|------|---------|---------|
| `issueNumber` | `number` | `12` | Current issue number |
| `issue` | `number` | `45` | Alternative issue field |
| `price` | `string` or `number` | `'49'` or `49.00` | Cover price |
| `barcode` | `string` | `'weekday'` | Price lookup key |
| `volume` | `number` | `130` | Volume override |

### Product data (`printProduct.data`)

| Field | Type | Example | Purpose |
|-------|------|---------|---------|
| `startYear` | `number` | `1868` | Volume year calculation |
| `homepage` | `string` | `'www.example.com'` | Publication URL |
| `tagline` | `string` | `'Your daily news'` | Tagline / subtitle |

### Product name (`printProduct.name`)

Used for per-product logic:

```javascript
const startYear = {
    'product-a': 2014,
    'product-b': 1881,
    'product-c': 1860
}[printProduct.name] ?? 1900;
```

## Logo Handling

### SVG Embedding

A processor can embed the logo SVG directly in the HTML:

```javascript
if (block.name === 'Logo') {
    content += LOGO_SVG;
    attributes.class = (attributes.class || '') + ' block--logo';
}

const LOGO_SVG = `<svg xmlns="http://www.w3.org/2000/svg" viewBox="..." class="logo-svg">
  <g class="first-line" fill="currentColor">...</g>
  <g class="second-line" fill="currentColor">...</g>
</svg>`;
```

Advantages:
- Logo scales with CSS
- Color controlled via `currentColor` and CSS classes
- No external file dependency

Use this approach when: the logo needs CSS-controlled coloring, or when there are multiple color variants.

### CSS Background Image

Most customers use a CSS background image for the logo:

```scss
.block--logo {
    .item--skin-logo {
        background-image: url('logo.svg');
        background-size: contain;
        background-repeat: no-repeat;
    }
}
```

The logo file lives in the asset repo's `static/` directory and is referenced via the CSS.

Use this approach when: the logo is static and doesn't need dynamic color changes.

## Registration

Add the processor to `src/lib/processors/index.js`:

1. Import the module:
```javascript
const myCustomer = require('./my-customer');
```

2. Add a case in `mapProcessor()`:
```javascript
case 'my-customer':
    return (block, content, attributes) => myCustomer.process(context, block, content, attributes);
```

Multiple themes can share a processor:
```javascript
case 'my-group-product-a':
case 'my-group-product-b':
case 'my-group':
    return (block, content, attributes) => myGroup.process(context, block, content, attributes);
```

The theme name is matched first. If no match, the part before the first `-` is tried as a fallback (e.g. `my-customer-special` falls back to `my-customer`).

## Testing

Processor tests follow this pattern:

```javascript
const {process} = require('./my-customer');

describe('my-customer processor', () => {
    const context = {
        printEdition: {
            data: { issueNumber: 12, price: '49' },
            publishDate: '2025-03-15'
        },
        printProduct: {
            name: 'my-customer-print',
            data: { startYear: 1990, homepage: 'www.example.com' }
        }
    };

    it('should inject edition info into Logo block', () => {
        const block = { name: 'Logo', elements: [] };
        const [content, attributes] = process(context, block, '', { class: 'block block--single' });

        expect(attributes.class).toContain('block--logo');
        expect(content).toContain('edition-info');
        expect(content).toContain('2025');
    });

    it('should pass through non-Logo blocks', () => {
        const block = { name: 'Other', elements: [] };
        const [content, attributes] = process(context, block, '<div>original</div>', { class: 'block' });

        expect(content).toBe('<div>original</div>');
    });
});
```

Run processor tests: `npm test -- --testPathPattern=processors`

## Annotated Example: Simple Processor

The simplest processor pattern. Good starting point for new customers.

```javascript
'use strict';

const dayjs = require('dayjs');
const advancedFormat = require('dayjs/plugin/advancedFormat');
const weekOfYear = require('dayjs/plugin/weekOfYear');
const isoWeek = require('dayjs/plugin/isoWeek');
require('dayjs/locale/nb');
dayjs.extend(advancedFormat);
dayjs.extend(weekOfYear);
dayjs.extend(isoWeek);

exports.process = ({printEdition, printProduct}, block, content, attributes) => {
    // Set locale for date formatting
    dayjs.locale('nb');

    // Only modify the Logo block
    if (block.name === 'Logo') {
        // Add logo class for CSS targeting
        attributes.class = attributes.class + ' block--logo';

        // Format date from edition publish date
        const d = dayjs(printEdition.publishDate);
        const day = d.format('dddd');  // Full weekday name

        // Replace content with edition info HTML
        // Each div has a semantic class for CSS styling
        content = `<div class="edition-info">
            <div class="date">${day[0].toUpperCase() + day.substr(1)} ${d.format('D. MMMM YYYY')}</div>
            <div class="week">${d.isoWeek()}</div>
            <div class="issue">${printEdition.data.issueNumber}</div>
            <div class="volume">${d.year() - printProduct.data.startYear}</div>
            <div class="price">${printEdition.data.price}</div>
            <div class="homepage">${printProduct.data.homepage}</div>
        </div>`;
    }

    return [content, attributes];
};
```

Key patterns:
- dayjs with locale for localized date formatting
- `block.name === 'Logo'` match
- `attributes.class += ' block--logo'` for CSS hook
- Edition data for dynamic values (issueNumber, price)
- Product data for stable values (startYear, homepage)
- Content as semantic HTML divs — CSS handles layout and decoration

## Annotated Example: SVG Logo and Modifier Lifting

A more complex processor demonstrating SVG embedding and modifier lifting.

```javascript
exports.process = ({printEdition, printProduct}, block, content, attributes) => {
    dayjs.locale('de');

    if (block.name === 'Logo') {
        // Embed SVG logo inline for CSS color control
        content += LOGO_SVG;

        // Add logo class
        attributes.class = (attributes.class || '') + ' block--logo';

        // Lift item modifiers to block level
        // This allows logo color variants (e.g. block--text-color-white)
        block.elements.forEach(({modifiers}) => {
            modifiers.forEach((modifier) => {
                attributes.class = attributes.class + ' block--' + modifier;
            });
        });

        const productData = printProduct.data || {};
        const editionData = printEdition.data || {};

        // Format issue/year
        const m = dayjs(printEdition.publishDate);
        const shortYear = m.format('YY');
        const issue = String(editionData.issueNumber || 0).padStart(2, '0');
        const tagline = productData.tagline || '';

        // Thin space (\u2009) between issue and year
        content += `<span class="issue-year">${issue}\u2009/\u2009${shortYear}</span>`;

        if (tagline) {
            content += `\n<span class="tagline">${tagline}</span>`;
        }
    }

    // Set German language on all named blocks
    if (block.name) {
        attributes.lang = 'de';
    }

    return [content, attributes];
};
```

Key patterns:
- SVG embedded as a constant string with `currentColor` for CSS theming
- Modifier lifting: item-level modifiers promoted to block classes
- Defensive defaults (`|| {}`, `|| 0`, `|| ''`)
- `lang` attribute for non-English content
- Conditional tagline rendering
