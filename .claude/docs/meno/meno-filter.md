## MenoFilter

Client-side filtering, sorting, search, and pagination for CMS collections. Works with SSR-rendered DOM elements using data attributes - no custom JavaScript required.

### Quick Start

\`\`\`html
<div data-meno-filter="posts" data-meno-per-page="6">
  <!-- Filter buttons -->
  <button data-meno-clear>All</button>
  <button data-meno-filter-field="category" data-meno-filter-value="tech">Tech</button>

  <!-- List container -->
  <div data-meno-list>
    <div data-category="tech">...</div>
    <div data-category="design">...</div>
  </div>

  <!-- Pagination -->
  <button data-meno-page="prev">Prev</button>
  <span data-meno-page-current></span> / <span data-meno-page-total></span>
  <button data-meno-page="next">Next</button>
</div>
\`\`\`

The script auto-initializes on page load. Items are filtered by matching \`data-{field}\` attributes.

### Container Attributes

| Attribute | Description | Example |
|-----------|-------------|---------|
| \`data-meno-filter="collection"\` | Wrapper element, scopes all controls | \`data-meno-filter="posts"\` |
| \`data-meno-list\` | Container for filterable items | \`<div data-meno-list>\` |
| \`data-meno-per-page="n"\` | Items per page (0 = no pagination) | \`data-meno-per-page="6"\` |
| \`data-meno-filter-match="and|or"\` | Logic between filters (default: and) | \`data-meno-filter-match="or"\` |
| \`data-meno-debounce="ms"\` | Search input debounce (default: 300) | \`data-meno-debounce="500"\` |
| \`data-meno-active-class="class"\` | Active filter button class (default: active) | \`data-meno-active-class="selected"\` |
| \`data-meno-types\` | JSON object mapping fields to types | \`data-meno-types='{"price":"number"}'\` |
| \`data-meno-url-sync="true"\` | Enable URL query parameter sync | \`data-meno-url-sync="true"\` |

### Type Coercion

Enable proper comparison operators (\$gt, \$lt, etc.) on dates and numbers:

\`\`\`html
<div data-meno-filter="products"
     data-meno-types='{"price":"number","date":"date","stock":"number","featured":"boolean"}'>
\`\`\`

| Type | Coercion | Example Values |
|------|----------|----------------|
| \`string\` | None (default) | \`"hello"\`, \`"123"\` |
| \`number\` | \`parseFloat(value)\` | \`199\`, \`19.99\` |
| \`date\` | \`new Date(value).getTime()\` | \`"2024-01-15"\` |
| \`boolean\` | \`value === 'true' || value === '1'\` | \`true\`, \`"true"\` |

### Filter Controls

\`\`\`html
<!-- Button (toggles on click) -->
<button data-meno-filter-field="category" data-meno-filter-value="tech">Tech</button>

<!-- Checkbox (multiple = OR logic) -->
<input type="checkbox" data-meno-filter-field="tags" data-meno-filter-value="featured">

<!-- Select dropdown -->
<select data-meno-filter-field="category">
  <option value="">All</option>
  <option value="tech">Tech</option>
</select>

<!-- Radio buttons -->
<input type="radio" name="status" data-meno-filter-field="status" data-meno-filter-value="published">

<!-- Clear filters -->
<button data-meno-clear>Clear Filters</button>

<!-- Reset all (filters + search + sort + page) -->
<button data-meno-reset>Reset</button>
\`\`\`

### Filter Mode

Control single vs multiple selections:

\`\`\`html
<!-- Single-choice buttons (radio-like, no toggle-off) -->
<div data-meno-filter-mode="single">
  <button data-meno-filter-field="category" data-meno-filter-value="tech">Tech</button>
  <button data-meno-filter-field="category" data-meno-filter-value="design">Design</button>
</div>

<!-- Multi-choice buttons (uses \$in operator) -->
<div data-meno-filter-mode="multi">
  <button data-meno-filter-field="tags" data-meno-filter-value="featured">Featured</button>
  <button data-meno-filter-field="tags" data-meno-filter-value="new">New</button>
</div>
\`\`\`

| Element | Default | mode="single" | mode="multi" |
|---------|---------|---------------|--------------|
| Button | Toggle | Radio-like | Multiple active |
| Checkbox | Multi-select | Unchecks siblings | Multi-select |

### Range Inputs

\`\`\`html
<!-- Number range -->
<input type="number" data-meno-range="price" data-meno-range-bound="min" placeholder="Min">
<input type="number" data-meno-range="price" data-meno-range-bound="max" placeholder="Max">

<!-- Date range -->
<input type="date" data-meno-range="publishDate" data-meno-range-bound="min">
<input type="date" data-meno-range="publishDate" data-meno-range-bound="max">
\`\`\`

### Search

\`\`\`html
<input type="text" data-meno-search data-meno-search-fields="title,description">

<!-- With fuzzy matching (typo-tolerant) -->
<input type="text" data-meno-search
       data-meno-search-fuzzy="true"
       data-meno-search-threshold="0.3">
\`\`\`

### Sort

\`\`\`html
<!-- Button -->
<button data-meno-sort="title" data-meno-sort-order="asc">Sort A-Z</button>

<!-- Select -->
<select data-meno-sort>
  <option value="">Default</option>
  <option value="title:asc">Title A-Z</option>
  <option value="views:desc">Most Viewed</option>
</select>
\`\`\`

### Pagination

\`\`\`html
<button data-meno-page="prev">Previous</button>
<button data-meno-page="next">Next</button>
<button data-meno-page="first">First</button>
<button data-meno-page="last">Last</button>

<span data-meno-page-current></span> / <span data-meno-page-total></span>

<!-- Auto-generated page buttons -->
<div data-meno-page-buttons></div>

<!-- Per-page selector -->
<select data-meno-per-page-select>
  <option value="6">6</option>
  <option value="12">12</option>
</select>
\`\`\`

### Load More (Alternative to Pagination)

\`\`\`html
<!-- Basic load more button -->
<button data-meno-load-more>Load More</button>

<!-- Custom increment -->
<button data-meno-load-more="6">Load More</button>

<!-- With remaining count -->
<button data-meno-load-more>
  Load More (<span data-meno-remaining></span> remaining)
</button>
\`\`\`

Button auto-hides when all items loaded. Mutually exclusive with pagination.

### Count Displays

\`\`\`html
<span data-meno-count="results"></span>  <!-- Filtered count -->
<span data-meno-count="total"></span>    <!-- Total items -->
<span data-meno-count="visible"></span>  <!-- Visible on current page -->
\`\`\`

### Facets

\`\`\`html
<span data-meno-facet="category:tech">0</span>  <!-- Count of items with category=tech -->
\`\`\`

### Empty State

\`\`\`html
<div data-meno-empty>No results found</div>
\`\`\`

Automatically shown/hidden based on filtered results.

### CMS Integration in Meno

Use a \`list\` node with \`sourceType: "collection"\` and add \`data-{field}\` attributes to each item so the filter can match them:

\`\`\`json
{
  "type": "list",
  "sourceType": "collection",
  "source": "posts",
  "itemAs": "item",
  "children": [
    {
      "type": "node",
      "tag": "div",
      "attributes": {
        "data-category": "{{item.category}}",
        "data-featured": "{{item.featured}}"
      },
      "children": [
        { "type": "node", "tag": "h3", "children": "{{item.title}}" }
      ]
    }
  ]
}
\`\`\`

Wrap with a filter container in the page:

\`\`\`json
{
  "type": "node",
  "tag": "div",
  "attributes": {
    "data-meno-filter": "posts",
    "data-meno-per-page": "6"
  },
  "children": [
    { "type": "node", "tag": "button", "attributes": { "data-meno-clear": "" }, "children": "All" },
    {
      "type": "list",
      "sourceType": "collection",
      "source": "posts",
      "itemAs": "item",
      "children": [ ... ]
    },
    { "type": "node", "tag": "button", "attributes": { "data-meno-page": "prev" }, "children": "Prev" },
    { "type": "node", "tag": "button", "attributes": { "data-meno-page": "next" }, "children": "Next" }
  ]
}
\`\`\`