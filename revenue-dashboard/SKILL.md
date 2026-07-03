---
name: revenue-dashboard
description: "Maintain and update the Monnai Revenue Dashboard -- a single-file HTML web app (all CSS + JS inline) hosted on GitHub Pages at https://harsh-monnai.github.io/Revenue-Dashboard/. It pulls live data from a Google Sheet via gviz CSV endpoint and renders KPIs, charts (Chart.js), cross-filters, insight cards, and a Period Compare page. Use this skill whenever anyone asks to: update the revenue dashboard, add or remove a chart or KPI tile, change a filter, fix something on the revenue dashboard, add a new page/tab, modify how data is displayed, adjust the Period Compare view, push a new version live, or troubleshoot anything on the revenue dashboard. Also trigger when anyone at Monnai describes a feature request like 'can we add X to the revenue dashboard', 'the filter for Y isn't working', 'can you show Z on the dashboard', or any variation of 'update the revenue dashboard'. Even if someone just says 'fix the revenue dashboard' or 'I want to change something on the revenue page' or 'add a new view to the dashboard' -- use this skill."
---

# Monnai Revenue Dashboard -- Maintenance Skill

## What This App Is

A single-file HTML web app (~930 lines, all CSS + JS inline) that replaced Looker Studio for Monnai's internal revenue analytics. It has two pages in a tabbed layout:

| Page | Purpose |
|---|---|
| **Dashboard** | KPI tiles, donut charts (Region, Invoiced From, Revenue Type), monthly revenue bar, invoice buckets table, top customers treemap, insight cards, cross-filtering sidebar |
| **Period Compare** | Side-by-side month or date range comparison with KPIs, geography/type/invoiced-from/monthly charts, Top 10 customer changes table, bucket changes, and new/lost customers |

**Live URL**: https://harsh-monnai.github.io/Revenue-Dashboard/
**GitHub Repo**: https://github.com/harsh-monnai/Revenue-Dashboard
**Data Source**: Google Sheet `1Ai6EHhwwJTiXBwfLCyKanACg_h1t1VtNejY7Jjc1blk`, tab `Sales`

Data is fetched live from the Google Sheets gviz CSV endpoint on page load, with a JSONP fallback. No backend, no login required.

---

## File Location

The dashboard is a single file: `index.html` in the root of the `harsh-monnai/Revenue-Dashboard` GitHub repo.

When editing:
1. Always use the `Read` tool to read the current file before making any edits
2. Use targeted `Edit` calls rather than rewriting the whole file
3. The file is ~930 lines -- read specific sections as needed rather than the whole thing

---

## Deployment

The dashboard is hosted on GitHub Pages from the `main` branch. Any push to `main` automatically deploys within 1-2 minutes.

### Option A: CLI Deployment (preferred when git credentials are available)

```bash
# Clone if not already cloned
git clone https://github.com/harsh-monnai/Revenue-Dashboard.git
cd Revenue-Dashboard

# Edit index.html, then:
git add index.html
git commit -m "Brief description of change"
git push origin main
```

**If push fails with "Authentication failed"**, ask the user for a GitHub Personal Access Token (PAT):
1. Go to https://github.com/settings/tokens/new
2. Create a classic token with `public_repo` scope, 30-day expiry
3. Update the remote: `git remote set-url origin https://harsh-monnai:{PAT}@github.com/harsh-monnai/Revenue-Dashboard.git`

### Option B: Browser-Based Deployment (when using Claude in Chrome)

1. Navigate to `https://github.com/harsh-monnai/Revenue-Dashboard/edit/main/index.html`
2. Wait for the CodeMirror 6 editor to load
3. Access the CM6 editor view: `document.querySelector('.cm-content').cmTile.view`
   - **Important**: the property is `cmTile`, NOT `cmView`
4. Replace content:
   ```javascript
   const view = document.querySelector('.cm-content').cmTile.view;
   const docLen = view.state.doc.length;
   view.dispatch({changes: {from: 0, to: docLen, insert: newContent}});
   ```
5. Click the "Commit changes" button and confirm

### Encoding Gotcha

When deploying via browser, be careful with Unicode characters. The browser-based GitHub editor can double-encode UTF-8 characters (treating UTF-8 bytes as Latin-1 then re-encoding as UTF-8). To avoid this:
- Use HTML entities instead of Unicode characters for special symbols (e.g., `&#x25B2;` instead of `â²` for triangle up arrows)
- When fetching existing file content via the GitHub API, decode properly: base64 decode -> Latin-1 to Uint8Array -> TextDecoder('utf-8')

---

## Architecture Overview

### Data Flow

```
Google Sheet (Sales tab)
  -> gviz CSV endpoint (primary) / JSONP (fallback)
  -> parseCSV() / parseJSONP()
  -> enrichRow() adds computed fields (_sales, _date, _yearMonth, _region, etc.)
  -> rawData[] (all rows)
  -> applyFilters() produces filteredData[]
  -> renderAll() updates every chart and KPI
```

### Page Structure

```
<body>
  #loading-overlay          -- spinner during data fetch
  #error-banner             -- retry bar on fetch failure
  .header-bar               -- logo, title, Standard/Forecast toggle, refresh button
  .tab-bar                  -- Dashboard | Period Compare tabs
  #page-dashboard           -- main dashboard (flex column, overflow hidden)
    .active-filters-bar     -- cross-filter chip bar
    .kpi-row                -- 8 KPI tiles
    .main-grid              -- 4-column CSS grid
      .filter-sidebar       -- region checkboxes, client/seller/date/month selects, insight cards
      .donut-top/mid/bottom -- 3 donut charts (Region, Invoiced From, Revenue Type)
      .monthly-col          -- monthly revenue bar chart (spans 3 rows)
      .right-top             -- invoice buckets table
      .right-bottom          -- top customers treemap
  #page-compare             -- period compare page (scrollable)
    .cmp-controls           -- period selectors, filters, Compare button
    .cmp-kpi-row            -- 4 KPI comparison tiles
    .cmp-charts             -- 4 comparison charts (2x2 grid)
    .cmp-tables             -- Top 10, Bucket Changes, New & Lost Customers
```

---

## Data Schema

### Row Object (after enrichRow)

Each row from the Google Sheet is enriched with computed fields:

| Field | Source | Type |
|---|---|---|
| `Client Name` | Sheet column | string |
| `Sales` | Sheet column | string (e.g., "$1,234") |
| `Date` | Sheet column | string (JS Date literal or M/D/Y) |
| `Region` | Sheet column | string (Americas, India, Europe, SEA, Indonesia) |
| `Invoice Type` | Sheet column | string (Recurring, Batch File, Proof of Concept Revenue, Voided/CN Issued) |
| `Invoiced from:` | Sheet column | string (US, India, Indonesia, Singapore) |
| `Seller` | Sheet column | string |
| `_sales` | Computed | number (parsed dollar amount) |
| `_date` | Computed | Date object |
| `_yearMonth` | Computed | string "YYYY-MM" |
| `_monthLabel` | Computed | string "Jan 24" |
| `_year` | Computed | number |
| `_monthNum` | Computed | number (1-12) |
| `_clientName` | Computed | string (alias for Client Name) |
| `_seller` | Computed | string |
| `_region` | Computed | string |
| `_invoiceType` | Computed | string |
| `_invoicedFrom` | Computed | string |

---

## Key JavaScript Functions

### Global State
- `rawData[]` -- all parsed rows from the Google Sheet
- `filteredData[]` -- rows after sidebar + chart cross-filters applied
- `currentMode` -- 'standard' or 'forecast' (persisted in localStorage)
- `chartFilters` -- object tracking cross-filter selections from clicking charts: `{region, invoiceType, month, client, invoicedFrom}`

### Data Loading
- `fetchData()` -- fetches CSV from gviz endpoint, falls back to JSONP, calls `populateFilters()` and `applyFilters()`
- `parseCSV(text)` -- hand-rolled CSV parser (handles quoted fields)
- `enrichRow(o)` -- adds computed `_sales`, `_date`, `_yearMonth`, etc.
- `fetchViaJSONP()` -- JSONP fallback when CSV endpoint fails

### Dashboard Filtering
- `applyFilters()` -- combines sidebar filters + chart cross-filters, produces `filteredData[]`, calls `renderAll()`
- `resetFilters()` -- clears all sidebar filters and chart cross-filters
- `setChartFilter(key, val)` -- toggles a cross-filter (clicking same value deselects)
- `clearChartFilters()` -- clears only chart-level cross-filters
- `renderFilterBar()` -- shows/hides the active filter chip bar

### Dashboard Rendering
- `renderAll()` -- calls all render functions below
- `renderKPIs()` -- updates 8 KPI tiles (customers, total, recurring, POC, batch, avg, MoM, period comparison)
- `renderPeriodComparison(d)` -- calculates YoY same-month comparison for the Period Comparison KPI
- `renderRegionTable()` -- region checkbox sidebar with sales totals
- `renderRegionDonut()` -- Sales by Region doughnut chart (clickable cross-filter)
- `renderInvoicedFromDonut()` -- Invoiced From doughnut (clickable)
- `renderTypeDonut()` -- Revenue by Type doughnut (clickable)
- `renderMonthlyBar()` -- Monthly Revenue stacked bar chart (clickable)
- `renderInvoiceBuckets()` -- Invoice Buckets table (Credit Note, Above $50k, $10k-$50k, $1k-$10k, Below $1k)
- `renderTreemap()` -- Top Customers treemap grid (clickable)
- `renderInsights()` -- 4 insight cards (new customers, top revenue, lowest revenue, at-risk)

### Tab Navigation
- `switchTab(name)` -- switches between 'dashboard' and 'compare' pages, handles Chart.js resize

### Period Compare
- `cmpMode` -- 'month' (single month) or 'range' (date range)
- `setCmpMode(mode)` -- toggles between single month and date range inputs
- `setCmpDefaults()` -- sets default periods (current month vs same month prior year)
- `populateCmpFilters()` -- fills region/client/seller dropdowns from rawData
- `getCmpData(isBase)` -- filters rawData for the selected period A or B
- `cmpLabel(isBase)` -- returns formatted label like "May 25" for the selected period
- `runComparison()` -- main entry point, calls all renderCmp* functions
- `deltaHTML(base, comp, isCur)` -- generates colored delta HTML with arrow icons (uses HTML entities `&#x25B2;`/`&#x25BC;` for encoding safety)
- `renderCmpKPIs(bD, cD)` -- 4 comparison KPI tiles (Total Rev, Customers, Recurring, Avg/Customer)
- `renderCmpGeo(bD, cD, lA, lB)` -- Geography comparison bar chart
- `renderCmpType(bD, cD, lA, lB)` -- Invoice Type comparison bar chart
- `renderCmpInvoiced(bD, cD, lA, lB)` -- Invoiced From comparison bar chart
- `renderCmpMonthly(bD, cD, lA, lB)` -- Monthly Revenue overlay bar chart
- `renderCmpTop10(bD, cD)` -- Top 10 Customer Revenue Changes table (sorted by Latest Month B revenue descending)
- `renderCmpBuckets(bD, cD)` -- Invoice Bucket Changes table
- `renderCmpNewLost(bD, cD)` -- New & Lost Customers table

### Formatting Helpers
- `fmtCur(v)` -- compact currency: $1.2M, $45K, $123
- `fmtFull(v)` -- full currency with commas: $1,234,567
- `fmtPct(v)` -- percentage with sign: +12.3%

---

## CSS Design System

### Colors
```css
/* Primary */
--blue: #1a73e8     /* header accents, primary actions, KPI borders */
--green: #0d652d    /* positive deltas, recurring KPI border */
--red: #c5221f      /* negative deltas, error banner */
--orange: #e8710a   /* POC KPI border */
--purple: #9334e6   /* Batch File KPI border */
--teal: #00897b     /* Avg/Customer KPI border */
--pink: #d81b60     /* MoM Growth KPI border */

/* Neutrals */
--bg: #f0f4f8       /* body background */
--card: #fff        /* card backgrounds */
--border: #e0e7ef   /* borders, grid lines */
--text: #333        /* primary text */
--label: #777       /* KPI labels, section headers */
```

### Chart Color Palettes
- **Region donut**: 5-color palette defined in `renderRegionDonut()`
- **Revenue Type donut**: 4-color palette defined in `renderTypeDonut()`
- **Invoiced From donut**: 4-color palette defined in `renderInvoicedFromDonut()`
- **Treemap**: color array in `renderTreemap()` (greens, blues, oranges, purples, teals)

### Period Compare Chart Colors
| Chart | Historic (A) | Latest (B) |
|---|---|---|
| Geography | `#90caf9` (light blue) | `#1a73e8` (blue) |
| Invoice Type | `#ffcc80` (light orange) | `#ff9800` (orange) |
| Invoiced From | `#ce93d8` (light purple) | `#6a1b9a` (purple) |
| Monthly Overlay | `#90caf9` (light blue) | `#1a73e8` (blue) |

### Layout
- Dashboard uses a 4-column CSS grid: `200px 1fr 1.4fr 1fr` with 3 rows
- Period Compare page is a single scrollable column with 2-column sub-grids for charts and tables
- Responsive breakpoints at 1200px, 1024px, and 900px

---

## How to Make Common Changes

### Add a new KPI tile
1. Add a `<div class="kpi-tile">` to `.kpi-row` in the HTML (follow existing pattern with `.kpi-label` + `.kpi-value`)
2. Give the value element a unique `id` (e.g., `id="kpi-newmetric"`)
3. Add CSS for the border-top color: `.kpi-tile.newmetric{border-top-color:#hexcolor}`
4. Calculate and set the value in `renderKPIs()`

### Add a new chart
1. Add a `<div class="chart-card">` in the grid with a `<canvas>` element
2. Assign it a grid position via CSS class or inline style
3. Add a global variable for the Chart.js instance (e.g., `let chartNew=null`)
4. Write a `renderNewChart()` function following the Chart.js 4.x pattern used in existing charts
5. Add `renderNewChart()` to the `renderAll()` function
6. If the chart should be clickable for cross-filtering, add an `onClick` handler that calls `setChartFilter()`

### Add a new filter
1. Add a `<div class="filter-control">` with a `<select>` to `.filter-sidebar`
2. Populate options in `populateFilters()` from rawData
3. Read the filter value in `applyFilters()` and add the filter condition
4. Reset it in `resetFilters()`

### Add a new tab/page
1. Add a `<button class="tab-btn" onclick="switchTab('newpage')">` to `.tab-bar`
2. Add a `<div id="page-newpage">` sibling to `#page-dashboard` and `#page-compare`
3. Style it: `#page-newpage{flex:1;overflow-y:auto;padding:12px 16px;display:none}`
4. Update `switchTab()` to handle the new page name -- set display to 'flex'/'block' and hide the others

### Modify the Period Compare Top 10 table
- Sort order is at line ~892: `ch.sort((a,b)=>b.comp-a.comp)` -- currently sorts by Latest Month (B) revenue descending
- To sort by absolute change instead: `ch.sort((a,b)=>Math.abs(b.diff)-Math.abs(a.diff))`
- Table columns are: #, Customer, Historic (A), Latest (B), Change ($), Change (%), Trend

### Change the data source
The Google Sheet ID is in `CONFIG.sheetId` at the top of the `<script>` block (~line 393):
```javascript
const CONFIG={sheetId:'1Ai6EHhwwJTiXBwfLCyKanACg_h1t1VtNejY7Jjc1blk',actualsTab:'Sales',forecastTab:null};
```
The sheet must be published to the web (File > Share > Publish to web) for the gviz endpoint to work.

### Add a new column from the sheet
1. The column must exist in the Google Sheet's `Sales` tab
2. It will automatically be parsed as a property of each row object (keyed by the column header)
3. Add a computed alias in `enrichRow()` if you want a cleaner property name (e.g., `o._newField = o['New Column'] || ''`)
4. Use it in render functions as needed

---

## Things to Watch Out For

- **Always read the file before editing** -- use the `Read` tool, then `Edit` for targeted replacements. Never guess line numbers.
- **Single-file constraint** -- everything stays in one HTML file. Do not split into separate CSS/JS files -- GitHub Pages serves just `index.html`.
- **Chart.js 4.4.1** -- loaded from CDN. All charts use the Chart.js 4.x API. Each chart instance must be destroyed before re-creating (`if(chart) chart.destroy()`).
- **Cross-filter state** -- clicking charts sets `chartFilters` which feeds into `applyFilters()`. If adding new clickable charts, follow the `setChartFilter(key, val)` pattern.
- **Encoding** -- use HTML entities (`&#x25B2;`, `&#x25BC;`, `&#x2014;`) for special characters in dynamically generated HTML, not raw Unicode. This prevents double-encoding issues during browser-based deployments.
- **Standard vs Forecast mode** -- the Forecast toggle is scaffolded but not yet active (`forecastTab:null`). The mode toggle switches a flag but only affects KPI rendering currently.
- **Period Compare labels** -- Period A is labeled "Historic Month" and Period B is labeled "Latest Month" throughout the UI (headers, table columns, chart legends).
- **gviz endpoint** -- if the Google Sheet is not published to web, the CSV endpoint will fail and it falls back to JSONP. Both require the sheet to be at least "Anyone with the link can view."

---

## Context File

The repo also contains `monnai_context.md` which provides business context about Monnai's entities, customers, products, data partners, and financial operations. This context is useful when interpreting dashboard data or explaining metrics to stakeholders. Read it when you need business context for the revenue data.
