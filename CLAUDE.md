# US National Debt Forecasting Dashboard

## Overview

Single-file web application for exploring how federal spending and tax policy changes affect the US national debt over a 20-year projection (2025–2045). No build tools, no server — just `index.html` opened in a browser.

## Architecture

- **Single file**: `index.html` contains all HTML, CSS (~450 lines), and JavaScript (~520 lines)
- **External dependency**: Chart.js loaded from CDN (`https://cdn.jsdelivr.net/npm/chart.js`)
- **Layout**: CSS Grid with a 380px left controls panel and flexible right charts area. Responsive — collapses to single column under 1100px.
- **Theme**: Dark theme with CSS custom properties (vars defined in `:root`)

## Data-Driven Config Pattern

All UI controls are generated from config arrays — no hardcoded HTML for individual controls. To add new items, just add an entry to the appropriate array.

### `SPENDING_CATEGORIES`
Array of `{ key, label, defaultVal, color }`. Each entry generates a slider in the controls panel. Values are in **billions USD**.

### `REVENUE_SOURCES`
Array of `{ key, label, rateLabel, rateUnit, defaultRate, min, max, step, base, baseLabel, baseGrowth, color, calcRevenue, wikiUrl, description }`. Each entry generates a toggle + rate input + estimated revenue display + Wikipedia link. The `calcRevenue(rate, base)` function computes annual revenue from the rate and current tax base.

### `ASSUMPTION_CONTROLS`
Array of `{ key, label, defaultVal, min, max, step, group }`. Each generates a slider in the assumptions panel. The `group` field is either `'assumptions'` (for `state.assumptions.*`) or `'revGrowth'` (for `state.revGrowth.*`). Values are displayed as percentages but stored as decimals in state.

### `EXISTING_REV_SPLIT`
Fixed proportions for breaking down baseline federal revenue into Income Tax (52%), Payroll Tax (24%), Corporate Tax (10%), Other (14%). Used only for the revenue doughnut chart.

## State Management

```
DEFAULTS (const) → deep clone → state (mutable, the single source of truth)
```

The `state` object contains:
- `nationalDebt`: starting debt in billions (38,000)
- `gdp`: starting GDP in billions (30,600)
- `baseRevenue`: existing federal revenue in billions (5,000)
- `spending`: object keyed by category (`socialSecurity`, `medicare`, etc.) — values set by sliders
- `newRevenue`: object keyed by tax type — each has `{ enabled: boolean, rate: number }`
- `assumptions`: `{ gdpGrowthRate, inflationRate, baseRevenueGrowthRate, spendingGrowthRate, interestRate }` — all stored as decimals (e.g., 0.025 for 2.5%)
- `revGrowth`: `{ wealthTax, landTax, stockTax, carbonTax, ftt }` — annual growth rates for each tax base, stored as decimals

`resetDefaults()` deep-clones `DEFAULTS` back into `state` and resets all DOM inputs.

## Economic Model — `runProjection()`

20-year year-by-year loop. For each year:

1. **Interest**: `debt × interestRate` (compounds on growing debt — this is the debt spiral mechanism)
2. **Total Spending**: sum of all spending categories + interest
3. **New Revenue**: for each enabled tax, apply its `calcRevenue(rate, currentBase)` to the current-year tax base
4. **Total Revenue**: baseline revenue + new revenue
5. **Deficit**: spending − revenue
6. **Advance**: debt += deficit, grow GDP, grow baseline revenue, grow each spending category, grow each tax base

### Key Model Behaviors

- **Wealth tax base depletion**: For wealth-based taxes (wealthTax, landTax, stockTax), the revenue collected is subtracted from the tax base before growth is applied. A 50% wealth tax will rapidly shrink its own base.
- **Interest compounding**: Interest is calculated on current debt each year. Higher deficits → more debt → more interest → bigger deficits. This feedback loop can cause explosive debt growth.
- **Tax base growth**: Each tax base grows at its own rate (configurable via assumption sliders). Carbon emissions decline at -2%/yr by default.
- **VAT special case**: VAT base is `GDP × 0.60` (consumption share), so it grows implicitly with GDP rather than having a separate base.
- **GDP consumption share** (`GDP_CONSUMPTION_SHARE = 0.60`) is a constant, not adjustable via slider.

## JavaScript Organization

The `<script>` block is organized into labeled sections:

| Section | Contents |
|---|---|
| 1: Constants & Defaults | `SPENDING_CATEGORIES`, `REVENUE_SOURCES`, `ASSUMPTION_CONTROLS`, `DEFAULTS`, `EXISTING_REV_SPLIT` |
| 2: State | `state` variable (deep clone of DEFAULTS) |
| 3: Economic Model | `runProjection()` — the core 20-year simulation loop |
| 4: Formatting | `fmt(billions)` and `fmtShort(billions)` — format numbers as `$1.4T` or `$912B` |
| 5: Build UI Controls | `buildSpendingControls()`, `buildRevenueControls()`, `buildAssumptionControls()` |
| 6: Charts | Chart.js initialization for all 4 charts (debt trajectory, revenue doughnut, spending doughnut, deficit bar) |
| 7: UI Updates | `updateMetrics()`, `updateRevenueEstimates()` |
| 8: Recalculate | `scheduleRecalc()` (30ms debounce) and `recalculate()` (runs projection → updates everything) |
| 9: Reset | `resetDefaults()` |
| 10: Init | `init()` called on DOMContentLoaded |

## Charts

1. **20-Year Debt Trajectory** (line chart) — 4 lines: National Debt (red filled), GDP (blue dashed), Total Spending (orange), Total Revenue (green)
2. **Revenue Breakdown** (doughnut) — existing tax sources + any enabled new taxes. Dynamic labels.
3. **Spending Breakdown** (doughnut) — all 7 categories including auto-calculated interest
4. **Revenue vs Spending** (horizontal bar) — visual comparison showing the deficit gap

Charts update via data mutation + `.update('none')` for instant response during slider dragging.

## How to Add Things

### New spending category
Add an entry to `SPENDING_CATEGORIES`: `{ key: 'education', label: 'Education', defaultVal: 200, color: '#...' }`. That's it — the slider, state, projection, charts, and reset all pick it up automatically.

### New revenue/tax source
Add an entry to `REVENUE_SOURCES` with all required fields including `calcRevenue`, `wikiUrl`, `description`. If it's a wealth-based tax (base depletes when taxed), add its key to the depletion check in `runProjection()`. Add a growth rate entry to `DEFAULTS.revGrowth` and an `ASSUMPTION_CONTROLS` entry if the base grows over time.

### New assumption slider
Add an entry to `ASSUMPTION_CONTROLS` with `group: 'assumptions'` or `group: 'revGrowth'`. If it's a new assumption key, also add it to `DEFAULTS.assumptions` or `DEFAULTS.revGrowth` and reference it in `runProjection()`.

## Baseline Data Sources

All figures are FY2025 approximations based on CBO, Treasury, and BEA data:
- National debt: ~$38.0T (debt held by public)
- GDP: ~$30.6T
- Federal revenue: ~$5.0T (breakdown: income 52%, payroll 24%, corporate 10%, other 14%)
- Federal spending: ~$7.0T (Social Security $1.4T, Medicare $912B, Medicaid $626B, Defense $900B, Veterans $326B, Other $1.8T, Interest ~$1.2T)
- Billionaire wealth: ~$8.2T (935 billionaires)
- Stock market cap: ~$69T
- US land value: ~$25T (estimate)
- CO2 emissions: ~5B metric tons/year
- Annual trading volume: ~$100T
