# Design Document

## Feature: Performance Optimization — Vanilla JS/CSS/HTML Rewrite

### Introduction

This document describes the technical architecture for rewriting "Tabungan Onlen" from a single-file React 18 + Babel standalone + Tailwind CDN bundle into a three-file Vanilla HTML5 + CSS3 + ES6+ JavaScript application. Every design decision maps directly to a requirement in `requirements.md`. No frameworks, CDNs, build tools, or runtime compilers are used.

---

## Overview

"Tabungan Onlen" is rewritten as a zero-dependency, three-file application (`index.html`, `app.css`, `app.js`) that eliminates all runtime compilation overhead from React, Babel, and Tailwind CDN. State is held in a single `AppState` object with an incremental `DerivedCache` to avoid re-scanning the full transaction list on every render. DOM updates are batched via `DocumentFragment` + `replaceChildren`, and expensive sub-views (charts) are rendered lazily on first activation and skipped on re-activation when data has not changed.

---

## Architecture

```
CodingCamp-21September26-dafimudlir/
├── index.html   ← Shell: markup skeleton + CriticalCSS + one deferred <script>
├── app.css      ← StyleSheet: all styles migrated from inline Tailwind equivalents
└── app.js       ← ScriptModule: entire application logic as a single ES6+ IIFE
```

The application is a **plain document-oriented SPA**. There is no virtual DOM, no module bundler, and no runtime JSX transform. All state lives in a single `AppState` object. All DOM mutations go through a small set of focused render functions that are called explicitly when state changes.

### Layered Architecture

```
┌─────────────────────────────────────────┐
│             index.html (Shell)          │
│  CriticalCSS · <link app.css> · <script app.js defer>  │
└────────────────────┬────────────────────┘
                     │ DOMContentLoaded
                     ▼
┌─────────────────────────────────────────┐
│            app.js — Sections            │
│                                         │
│  §1  Constants & Category Definitions   │
│  §2  Formatter Utilities                │
│  §3  LocalStorage Wrapper (LS)          │
│  §4  AppState — single source of truth  │
│  §5  DerivedCache — incremental cache   │
│  §6  FilteredList — cached filter/sort  │
│  §7  StorageQueue — async LS writes     │
│  §8  DOM Helpers — fragment, textNode   │
│  §9  LazyPanel Registry                 │
│  §10 Render Functions                   │
│  §11 Event Wiring (ThrottleGate,        │
│       DebounceTimer, init)              │
└─────────────────────────────────────────┘
```

---

## File Responsibilities

### `index.html` — Shell

- Contains exactly one `<style>` block (CriticalCSS, ≤ 2 KB) that renders the loading state.
- Contains `<link rel="stylesheet" href="app.css">` in `<head>` (the only render-blocking resource).
- Contains `<script src="app.js" defer>` in `<head>` (no `type="module"`, no `type="text/babel"`).
- Includes `<link rel="preconnect" href="https://fonts.googleapis.com">` for the Plus Jakarta Sans font.
- HTML skeleton provides all structural containers that `app.js` populates; no JSX or template literals needed at runtime.
- No other inline `<style>` or inline `<script>` blocks are present.

```html
<!DOCTYPE html>
<html lang="id" class="">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Tabungan Onlen 💰</title>

  <!-- Font preconnect (does not block render) -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

  <!-- CriticalCSS: ≤ 2 KB, renders above-the-fold loading state -->
  <style>
    /* ... (see CriticalCSS section) ... */
  </style>

  <!-- StyleSheet: render-blocking, all non-critical styles -->
  <link rel="stylesheet" href="app.css" />

  <!-- ScriptModule: deferred, no type=module -->
  <script src="app.js" defer></script>
</head>
<body>
  <!-- Loading placeholder (replaced when app.js initialises) -->
  <div id="root">
    <div class="loading-shell">
      <div class="loading-icon">🏦</div>
      <div class="loading-title">Tabungan Onlen</div>
      <div class="loading-sub">Memuat aplikasi…</div>
    </div>
  </div>

  <!-- Static containers — app.js fills these in -->
  <header id="app-header" hidden></header>
  <main id="app-main" hidden></main>
  <footer id="app-footer" hidden></footer>
  <div id="toast-container" aria-live="polite"></div>
  <div id="dialog-container"></div>
</body>
</html>
```

### `app.css` — StyleSheet

- All visual styles: glass morphism, animations, scrollbar, dark mode (`html.dark`), responsive breakpoints, typography.
- Includes `@import url('https://fonts.googleapis.com/...')` **at the top** with `font-display: swap`.
- No `@tailwind` directives; styles are hand-written CSS classes using the same naming conventions as the original Tailwind utility classes where practical.
- Animation keyframes are defined here; JavaScript only adds/removes class names.

### `app.js` — ScriptModule

- A single IIFE `(function() { 'use strict'; ... })()` so nothing leaks to `window`.
- All sections described below are sequential top-level code within the IIFE.
- Wired to `DOMContentLoaded` for initialisation.

---

## CriticalCSS Design

The inline `<style>` block in `<head>` must render the loading indicator before `app.css` arrives. It is capped at 2 KB unminified.

```css
/* CriticalCSS — max 2 KB */
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}
html{background:#fff7ed}
html.dark{background:#0f172a}
body{
  font-family:'Plus Jakarta Sans',ui-sans-serif,system-ui,sans-serif;
  min-height:100dvh;
}
#root{
  min-height:100dvh;display:flex;
  align-items:center;justify-content:center;
}
.loading-shell{
  text-align:center;color:#92400e;
  font-family:'Plus Jakarta Sans',sans-serif;
}
.loading-icon{
  font-size:3.5rem;
  animation:_float 2s ease-in-out infinite;
}
.loading-title{
  font-size:1.35rem;font-weight:800;
  margin-top:.75rem;color:#f97316;
}
.loading-sub{
  font-size:.85rem;margin-top:.4rem;color:#a16207;
}
@keyframes _float{
  0%,100%{transform:translateY(0)}
  50%{transform:translateY(-8px)}
}
```

This intentionally uses a private `_float` keyframe name so it does not clash with the `float` keyframe defined in `app.css`.

---

## Components and Interfaces

The following components are defined in `app.js` as plain objects and functions within a single IIFE. Full implementations appear in subsequent sections; this table is the authoritative interface summary.

### AppState

| Member | Signature | Description |
|---|---|---|
| `txs` | `Transaction[]` | Master transaction list; mutated only by add/delete/edit/import operations |
| `budget` | `number` | Savings target in IDR; persisted to `localStorage` |
| `dark` | `boolean` | Current dark-mode flag; toggled by theme button |
| `isLoaded` | `boolean` | Guards `scheduleStorageSave` from firing before init completes |
| `txFilter` | `'semua' \| 'pemasukan' \| 'pengeluaran'` | Active list filter; changing it invalidates `FilterState` |
| `searchQuery` | `string` | Active search string; changing it invalidates `FilterState` |
| `txVersion` | `number` | Monotonically incrementing counter; bumped on every `txs` mutation |

### DerivedCache

| Function | Signature | Description |
|---|---|---|
| `rebuildDerivedCache` | `(txs: Transaction[]) => void` | Full O(n) recompute; called at init and after bulk import |
| `cacheAdd` | `(tx: Transaction) => void` | Incremental add: O(1) update to all cache fields |
| `cacheDel` | `(tx: Transaction) => void` | Incremental remove: O(1) update to all cache fields |
| `cacheEdit` | `(oldTx: Transaction, newTx: Transaction) => void` | Incremental replace: equivalent to `cacheDel(oldTx); cacheAdd(newTx)` |

### FilteredList

| Function / Member | Signature | Description |
|---|---|---|
| `FilterState` | `{ lastTxVersion, lastFilter, lastQuery, cachedResult }` | Memoisation keys and cached array reference |
| `getFilteredList` | `() => Transaction[]` | Returns cached result on hit; recomputes (filter + sort) on miss |

### StorageQueue

| Function | Signature | Description |
|---|---|---|
| `scheduleStorageSave` | `() => void` | Debounces `localStorage.setItem` via `requestIdleCallback` / `setTimeout`; coalesces rapid mutations into a single write |

### ThrottleGate

| Function | Signature | Description |
|---|---|---|
| `withAddThrottle` | `(fn: () => void) => void` | Executes `fn` and blocks re-entry for 300 ms; disables the add button during the window |
| `withDelThrottle` | `(txId: string, fn: (id: string) => void) => void` | Executes `fn(txId)` and blocks all delete buttons for 300 ms |

### DebounceTimer

| Function | Signature | Description |
|---|---|---|
| `onSearchInput` | `(value: string) => void` | Clears immediately on empty string; otherwise debounces `renderTxList` by 300 ms |

### DOM Helpers

| Function | Signature | Description |
|---|---|---|
| `renderTxList` | `() => void` | Rebuilds `#tx-list-items` from a `DocumentFragment`; calls `getFilteredList()` |
| `updateTextNode` | `(elementId: string, newText: string) => void` | Mutates an existing text node in place; avoids layout recalc on summary card updates |
| `renderSummaryCards` | `() => void` | Updates income / expense / net text nodes from `DerivedCache` |
| `renderHeaderBudgetBar` | `() => void` | Updates budget bar width, percentage, and spent text from `DerivedCache` |
| `renderAll` | `() => void` | Calls all render functions once; used at init and after import |

### LazyPanel Registry

| Function / Member | Signature | Description |
|---|---|---|
| `LazyPanels` | `{ donut, bar, flow }: { node: Element \| null, renderedAtVersion: number }` | Per-chart cache of the constructed DOM node and the `txVersion` at last render |
| `activateChartTab` | `(tabId: 'donut' \| 'bar' \| 'flow') => void` | Creates the panel on first call; skips `renderChartData` when `txVersion` is unchanged |
| `buildChartPanel` | `(tabId: string) => Element` | Constructs the static SVG/HTML skeleton once per tab |
| `renderChartData` | `(tabId: string, node: Element) => void` | Updates data-bound attributes/text nodes without recreating the skeleton |
| `invalidateLazyPanels` | `() => void` | Sets all `renderedAtVersion` to `-1`; called after every `txs` mutation |

---

## AppState — Single Source of Truth

All mutable state lives in one plain object. No framework reactivity; renders are triggered imperatively.

```js
// §4  AppState
const AppState = {
  txs:        [],          // Transaction[]
  budget:     5_000_000,   // number (IDR)
  dark:       false,       // boolean
  isLoaded:   false,       // guards localStorage writes (see §11)

  // UI state
  txFilter:   'semua',     // 'semua' | 'pemasukan' | 'pengeluaran'
  searchQuery: '',         // string

  // Version counter — incremented on every txs mutation
  txVersion:  0,
};
```

**Transaction shape:**
```js
/**
 * @typedef {{ id: string, type: 'income'|'expense', cat: string,
 *             amt: number, date: string, desc: string }} Transaction
 */
```

---

## DerivedCache — Incremental Computed Value Cache

### Data Structure

```js
// §5  DerivedCache
const DerivedCache = {
  totalInc:    0,   // sum of all income amounts
  totalExp:    0,   // sum of all expense amounts
  netBal:      0,   // totalInc - totalExp
  catTotals:   {},  // { [catId: string]: number }  — expense totals per category
  monthTotals: {},  // { [YYYY-MM: string]: { inc: number, exp: number } }
};
```

### Full Rebuild

Called once at startup (after LocalStorage read) and once after a full import:

```js
function rebuildDerivedCache(txs) {
  DerivedCache.totalInc    = 0;
  DerivedCache.totalExp    = 0;
  DerivedCache.catTotals   = {};
  DerivedCache.monthTotals = {};

  for (const tx of txs) {
    _applyTxToCache(tx, +1);
  }
  DerivedCache.netBal = DerivedCache.totalInc - DerivedCache.totalExp;
}
```

### Incremental Helpers

```js
// Internal: apply a transaction delta (+1 = add, -1 = remove)
function _applyTxToCache(tx, sign) {
  const amt = tx.amt * sign;
  const mo  = tx.date.slice(0, 7);  // 'YYYY-MM'

  if (tx.type === 'income') {
    DerivedCache.totalInc += amt;
  } else {
    DerivedCache.totalExp += amt;
    DerivedCache.catTotals[tx.cat] =
      (DerivedCache.catTotals[tx.cat] || 0) + amt;
    if (DerivedCache.catTotals[tx.cat] <= 0)
      delete DerivedCache.catTotals[tx.cat];
  }

  if (!DerivedCache.monthTotals[mo])
    DerivedCache.monthTotals[mo] = { inc: 0, exp: 0 };
  if (tx.type === 'income')
    DerivedCache.monthTotals[mo].inc += amt;
  else
    DerivedCache.monthTotals[mo].exp += amt;

  DerivedCache.netBal = DerivedCache.totalInc - DerivedCache.totalExp;
}

function cacheAdd(tx)           { _applyTxToCache(tx, +1); }
function cacheDel(tx)           { _applyTxToCache(tx, -1); }
function cacheEdit(oldTx, newTx) {
  _applyTxToCache(oldTx, -1);
  _applyTxToCache(newTx, +1);
}
```

**Invariant**: After any sequence of `cacheAdd`/`cacheDel`/`cacheEdit` calls, `DerivedCache` must equal the result of calling `rebuildDerivedCache` over the full in-memory `AppState.txs`.

---

## FilteredList Cache

### Invalidation Keys

The FilteredList is valid as long as these three values have not changed since the last computation:

```js
// §6  FilteredList cache
const FilterState = {
  lastTxVersion:   -1,
  lastFilter:      null,
  lastQuery:       null,
  cachedResult:    [],   // Transaction[] — sorted descending by date
};
```

### Retrieval with Cache Check

```js
function getFilteredList() {
  const { txVersion, txFilter, searchQuery } = AppState;

  if (
    txVersion  === FilterState.lastTxVersion &&
    txFilter   === FilterState.lastFilter    &&
    searchQuery === FilterState.lastQuery
  ) {
    return FilterState.cachedResult;  // cache hit — same reference returned
  }

  // Cache miss — recompute
  let result = AppState.txs.slice().sort(
    (a, b) => b.date.localeCompare(a.date)
  );

  if (txFilter === 'pemasukan')   result = result.filter(t => t.type === 'income');
  if (txFilter === 'pengeluaran') result = result.filter(t => t.type === 'expense');

  if (searchQuery.trim()) {
    const q = searchQuery.toLowerCase();
    result = result.filter(t =>
      t.desc.toLowerCase().includes(q) ||
      (CAT[t.cat]?.label || '').toLowerCase().includes(q)
    );
  }

  FilterState.lastTxVersion = txVersion;
  FilterState.lastFilter    = txFilter;
  FilterState.lastQuery     = searchQuery;
  FilterState.cachedResult  = result;
  return result;
}
```

Sorting is done **once** at cache miss time. The `cachedResult` array is never mutated after creation.

---

## StorageQueue — Async LocalStorage Writes

### Design

```js
// §7  StorageQueue
const StorageQueue = { handle: null };

function scheduleStorageSave() {
  if (!AppState.isLoaded) return;

  // Cancel any pending write and replace with latest state
  if (StorageQueue.handle !== null) {
    if (typeof cancelIdleCallback !== 'undefined')
      cancelIdleCallback(StorageQueue.handle);
    else
      clearTimeout(StorageQueue.handle);
    StorageQueue.handle = null;
  }

  const saveNow = () => {
    try {
      LS.setJSON(KEYS.TXS,    AppState.txs);
      LS.set(KEYS.BUDGET,     String(AppState.budget));
    } catch (e) {
      if (e instanceof DOMException) {
        showToast('❌ Penyimpanan penuh. Data tidak dapat disimpan.', 'error');
      }
    }
    StorageQueue.handle = null;
  };

  if (typeof requestIdleCallback !== 'undefined') {
    StorageQueue.handle = requestIdleCallback(saveNow, { timeout: 2000 });
  } else {
    StorageQueue.handle = setTimeout(saveNow, 0);
  }
}
```

**Ordering guarantee**: All mutation functions (add, delete, edit) update `AppState.txs` and call `renderXxx()` synchronously, then call `scheduleStorageSave()`. The save callback executes after the current call stack clears — the DOM is always updated before `localStorage.setItem` is called.

---

## ThrottleGate — Add/Delete Rate Limiting

```js
// §11  ThrottleGate
const Throttle = {
  addActive: false,
  delActive: false,
};

const THROTTLE_MS = 300;

function withAddThrottle(fn) {
  if (Throttle.addActive) return;
  Throttle.addActive = true;

  const btn = document.getElementById('btn-add-tx');
  if (btn) btn.disabled = true;

  fn();  // execute the add action

  setTimeout(() => {
    Throttle.addActive = false;
    if (btn) btn.disabled = false;
  }, THROTTLE_MS);
}

function withDelThrottle(txId, fn) {
  if (Throttle.delActive) return;
  Throttle.delActive = true;

  // Disable all delete buttons during throttle window
  document.querySelectorAll('.btn-del-tx').forEach(b => b.disabled = true);

  fn(txId);

  setTimeout(() => {
    Throttle.delActive = false;
    document.querySelectorAll('.btn-del-tx').forEach(b => b.disabled = false);
  }, THROTTLE_MS);
}
```

---

## DebounceTimer — Search Input

```js
// §11  DebounceTimer
let _debounceHandle = null;
const DEBOUNCE_MS   = 300;

function onSearchInput(value) {
  // Immediate clear path: empty input bypasses debounce
  if (value === '') {
    if (_debounceHandle !== null) {
      clearTimeout(_debounceHandle);
      _debounceHandle = null;
    }
    AppState.searchQuery = '';
    renderTxList();
    return;
  }

  if (_debounceHandle !== null) clearTimeout(_debounceHandle);
  _debounceHandle = setTimeout(() => {
    _debounceHandle    = null;
    AppState.searchQuery = value;
    renderTxList();
  }, DEBOUNCE_MS);
}
```

---

## DOM Batching — DocumentFragment + replaceChildren

All list re-renders use a single `DocumentFragment` to avoid repeated reflow:

```js
// §8  DOM Helpers
function renderTxList() {
  const list    = getFilteredList();
  const container = document.getElementById('tx-list-items');
  if (!container) return;

  const frag = document.createDocumentFragment();
  if (list.length === 0) {
    frag.appendChild(buildEmptyState());
  } else {
    list.forEach((tx, i) => frag.appendChild(buildTxCard(tx, i)));
  }

  // Single reflow: all children replaced atomically
  container.replaceChildren(frag);
}
```

Summary card value updates go through a dedicated helper that mutates only the text node, never re-creates the card element:

```js
function updateTextNode(elementId, newText) {
  const el = document.getElementById(elementId);
  if (el && el.firstChild?.nodeType === Node.TEXT_NODE) {
    el.firstChild.nodeValue = newText;  // text node mutation — no reflow
  } else if (el) {
    el.textContent = newText;
  }
}
```

Animations are driven entirely by CSS class toggles:

```js
// Entry animation: add pop-in class, never write el.style.*
card.classList.add('pop-in');

// Exit animation: add exit-right class, remove card after transition ends
function deleteTxCard(id) {
  const card = document.getElementById(`tx-${id}`);
  if (!card) return;
  card.classList.add('exit-right');
  card.addEventListener('animationend', () => card.remove(), { once: true });
}
```

---

## LazyPanel Registry — Deferred Chart Rendering

Charts are not rendered at page load. Each panel is created the first time its tab is activated.

### Registry Data Structure

```js
// §9  LazyPanel Registry
const LazyPanels = {
  donut: { node: null, renderedAtVersion: -1 },
  bar:   { node: null, renderedAtVersion: -1 },
  flow:  { node: null, renderedAtVersion: -1 },
};
```

### Tab Activation Logic

```js
function activateChartTab(tabId) {
  const panel = LazyPanels[tabId];
  const container = document.getElementById('chart-panel-container');
  if (!container) return;

  const currentVersion = AppState.txVersion;

  if (panel.node === null) {
    // First activation: create DOM elements
    panel.node = buildChartPanel(tabId);
    container.replaceChildren(panel.node);
    panel.renderedAtVersion = currentVersion;
    renderChartData(tabId, panel.node);
    return;
  }

  // Reuse cached node
  if (container.firstChild !== panel.node) {
    container.replaceChildren(panel.node);
  }

  // Skip recalculation if data has not changed
  if (panel.renderedAtVersion === currentVersion) return;

  renderChartData(tabId, panel.node);
  panel.renderedAtVersion = currentVersion;
}
```

`buildChartPanel` creates the static SVG/HTML skeleton once. `renderChartData` updates data-bound text nodes, `d` attributes on SVG paths, and bar `height`/`y` attributes — it never recreates the skeleton.

---

## LocalStorage Initialisation — Single-Pass

```js
// §3  LS wrapper (unchanged from original — already try/catch safe)
const LS = {
  get(key, fallback = null) { ... },
  set(key, val) { ... },
  getJSON(key, fallback = null) { ... },
  setJSON(key, val) { ... },
};

// §11  init() — called once on DOMContentLoaded
function init() {
  // ── Single synchronous block reads all three keys ──
  let storedTxs    = null;
  let storedBudget = null;
  let storedTheme  = null;

  try {
    storedTxs    = LS.getJSON(KEYS.TXS,    null);
    storedBudget = parseInt(LS.get(KEYS.BUDGET, ''), 10);
    storedTheme  = LS.get(KEYS.THEME, 'light');
  } catch {
    // Storage unavailable — all remain null, defaults applied below
  }

  // Apply values or defaults
  AppState.txs    = Array.isArray(storedTxs) ? storedTxs : [];
  AppState.budget = (!isNaN(storedBudget) && storedBudget > 0)
                    ? storedBudget : 5_000_000;
  AppState.dark   = storedTheme === 'dark';

  // Populate DerivedCache BEFORE first render
  rebuildDerivedCache(AppState.txs);

  // Now safe to write
  AppState.isLoaded = true;

  // Apply dark mode class
  document.documentElement.classList.toggle('dark', AppState.dark);

  // First render
  renderAll();
}

document.addEventListener('DOMContentLoaded', init, { once: true });
```

**Key guarantee**: `rebuildDerivedCache` is called before `renderAll()`. No render function ever iterates `AppState.txs` directly — they always read from `DerivedCache` or `getFilteredList()`.

---

## State Mutation Sequence

Every user action follows this invariant ordering:

```
1. Update AppState.txs (and AppState.budget if applicable)
2. Increment AppState.txVersion
3. Update DerivedCache incrementally (cacheAdd / cacheDel / cacheEdit)
4. Call render functions (read from DerivedCache / getFilteredList)
5. Call scheduleStorageSave()  ← deferred, after call stack clears
```

Example for adding a transaction:

```js
function addTransaction(tx) {
  withAddThrottle(() => {
    AppState.txs.unshift(tx);       // step 1
    AppState.txVersion++;           // step 2
    cacheAdd(tx);                   // step 3
    renderSummaryCards();           // step 4a
    renderHeaderBudgetBar();        // step 4b
    renderTxList();                 // step 4c
    invalidateLazyPanels();         // step 4d — mark panels as stale
    scheduleStorageSave();          // step 5
  });
}
```

`invalidateLazyPanels` sets each `LazyPanels[id].renderedAtVersion = -1` if the panel's data domain is affected — charts always depend on `txVersion`.

---

## Data Models

### Transaction

```js
/**
 * @typedef {{
 *   id:   string,      // uid() — 9-char base-36
 *   type: 'income' | 'expense',
 *   cat:  string,      // key into CATS array
 *   amt:  number,      // integer IDR, > 0
 *   date: string,      // ISO date 'YYYY-MM-DD'
 *   desc: string,      // non-empty string
 * }} Transaction
 */
```

### DerivedCache (runtime only, never persisted)

```js
/**
 * @typedef {{
 *   totalInc:    number,
 *   totalExp:    number,
 *   netBal:      number,
 *   catTotals:   Record<string, number>,
 *   monthTotals: Record<string, { inc: number, exp: number }>,
 * }} DerivedCache
 */
```

### Export/Import payload (JSON, backward-compatible with v3)

```js
/**
 * @typedef {{
 *   version:      number,      // always 3
 *   exportedAt:   string,      // ISO datetime
 *   budget:       number,
 *   transactions: Transaction[],
 * }} ExportPayload
 */
```

---

## Error Handling

| Scenario | Handling |
|---|---|
| `localStorage` read throws on init | Catch silently; apply defaults; `isLoaded` still set to `true` |
| `localStorage.setItem` throws `DOMException` | Catch in StorageQueue callback; show error toast; keep in-memory state |
| Import file JSON parse error | Catch in `FileReader.onload`; show error toast; leave existing state unchanged |
| Import JSON missing `transactions` array | Validate before applying; show error toast |
| `getItem` returns `null` for transactions | Treated as empty array (no change to `AppState.txs`) |
| Negative or non-numeric budget from storage | Fallback to `5_000_000` |
| Transaction `amt` field ≤ 0 on form submit | Validate in submit handler; show inline error; prevent add |

---

## Render Function Map

| Function | Reads from | Updates |
|---|---|---|
| `renderSummaryCards()` | `DerivedCache` | `#summary-income`, `#summary-expense`, `#summary-net` text nodes |
| `renderHeaderBudgetBar()` | `DerivedCache`, `AppState.budget` | `#budget-bar` width style, `#budget-pct` text, `#budget-spent` text |
| `renderTxList()` | `getFilteredList()` | `#tx-list-items` via `replaceChildren(fragment)` |
| `renderTopSpenders()` | `DerivedCache.catTotals` | `#top-spenders` via `replaceChildren(fragment)` |
| `renderChartData('donut', node)` | `DerivedCache.catTotals` | SVG `<path d>` attributes, legend text nodes |
| `renderChartData('bar', node)` | `DerivedCache.monthTotals` | SVG `<rect>` height/y attributes |
| `renderChartData('flow', node)` | `DerivedCache` | Flow node amount text nodes |
| `renderAll()` | all of the above | Called once at init and after import |

---

## Testing Strategy

### Approach

The eleven correctness properties below are verified through **property-based testing (PBT)** using [fast-check](https://fast-check.dev/). Each property is expressed as a universal statement ("for any valid input, the following invariant holds"), which maps directly to a `fc.property` assertion.

### Transaction Generator

```js
const arbTransaction = fc.record({
  id:   fc.string({ minLength: 1 }),
  type: fc.constantFrom('income', 'expense'),
  cat:  fc.constantFrom(...Object.keys(CATS)),
  amt:  fc.integer({ min: 1, max: 1_000_000_000 }),
  date: fc.date({ min: new Date('2020-01-01'), max: new Date('2030-12-31') })
        .map(d => d.toISOString().slice(0, 10)),
  desc: fc.string({ minLength: 1 }),
});

const arbTxArray = fc.array(arbTransaction, { minLength: 0, maxLength: 500 });
```

Unique IDs are enforced by post-filtering with `fc.uniqueArray(arbTransaction, { selector: t => t.id })` where identity is required.

### Property-to-Test Mapping

| Property | Test description | fast-check strategy |
|---|---|---|
| **P1** CriticalCSS size | Assert `new TextEncoder().encode(criticalCssText).length <= 2048` | Static assertion (no generator needed) |
| **P2** DerivedCache full rebuild | For any `txs`, `rebuildDerivedCache(txs)` produces correct sums | `arbTxArray` |
| **P3** Incremental cache equivalence | `cacheAdd`, `cacheDel`, `cacheEdit` each equal the full rebuild result | `arbTxArray` + single `arbTransaction` |
| **P4** FilteredList descending sort | Every adjacent pair in `getFilteredList()` satisfies `a.date >= b.date` | `arbTxArray` |
| **P5** FilteredList cache identity | Two calls without state mutation return `===` reference | `arbTxArray` |
| **P6** ThrottleGate deduplication | Firing `n` adds within 300 ms appends exactly 1 transaction | `fc.integer({ min: 2, max: 20 })` for repeat count |
| **P7** LocalStorage single-read | `getItem` call count per key equals 1 after `init()` | `arbTxArray` with `localStorage` spy |
| **P8** Cache before first render | `DerivedCache.totalInc` after `init()` equals sum of income in stored txs before any render | `arbTxArray` |
| **P9** Lazy panel deferred creation | `LazyPanels[t].node === null` before first tab activation | `fc.constantFrom('donut','bar','flow')` |
| **P10** Chart skip on same version | `renderChartData` not called when `txVersion` is unchanged | `fc.constantFrom('donut','bar','flow')` |
| **P11** StorageQueue coalescing | `setItem` called exactly once after `n` rapid mutations | `fc.integer({ min: 2, max: 50 })` |

### Unit Testing

Non-property tests cover error paths that are deterministic rather than universal:

- `scheduleStorageSave` — `DOMException` triggers error toast and keeps in-memory state intact
- Import handler — malformed JSON shows error toast, leaves `AppState.txs` unchanged
- Import handler — missing `transactions` key shows error toast
- `LS.getJSON` returning `null` — `AppState.txs` defaults to `[]`
- Budget read returning `NaN` or `≤ 0` — fallback to `5_000_000`

### Integration Testing

End-to-end scenarios (JSDOM or real browser):

1. **Full session lifecycle** — `init()` → add 3 transactions → verify `DerivedCache`, `FilteredList`, and `localStorage` snapshot
2. **Dark mode persistence** — toggle theme, reload page, verify `html.dark` class present
3. **Export → import round-trip** — export JSON, clear state, re-import, verify `AppState.txs` identity
4. **Chart lazy render** — activate donut tab, mutate a transaction, activate donut tab again, verify `renderChartData` called exactly twice total

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: CriticalCSS size invariant

*For any* version of `index.html`, the text content of the single inline `<style>` block in `<head>` SHALL have a byte length ≤ 2048 (measured as UTF-8 bytes of the unminified CSS text).

**Validates: Requirements 2.3**

---

### Property 2: DerivedCache correctness after full rebuild

*For any* array of transactions `txs`, calling `rebuildDerivedCache(txs)` SHALL produce a `DerivedCache` state where `totalInc` equals the sum of `amt` for all income transactions in `txs`, `totalExp` equals the sum of `amt` for all expense transactions, `netBal` equals `totalInc − totalExp`, `catTotals[c]` equals the sum of `amt` for all expense transactions with `cat === c`, and `monthTotals[m]` equals the combined income and expense sums for transactions whose `date.slice(0,7) === m`.

**Validates: Requirements 4.1, 4.5**

---

### Property 3: Incremental cache equivalence

*For any* non-empty transaction list `txs` and any transaction `tx`, the result of calling `cacheAdd(tx)` on the cache built from `txs` SHALL be identical to calling `rebuildDerivedCache([...txs, tx])` from scratch. Symmetrically, calling `cacheDel(tx)` on the cache built from `txs` (where `tx` is in `txs`) SHALL equal `rebuildDerivedCache(txs.filter(t => t.id !== tx.id))`. And calling `cacheEdit(oldTx, newTx)` SHALL equal `rebuildDerivedCache(txs.map(t => t.id === oldTx.id ? newTx : t))`.

**Validates: Requirements 4.2, 4.3, 4.4**

---

### Property 4: FilteredList is sorted descending by date

*For any* application state with a non-empty transaction list, the array returned by `getFilteredList()` SHALL be sorted in descending order by the `date` field (i.e., for every adjacent pair `[a, b]`, `a.date >= b.date`).

**Validates: Requirements 5.3**

---

### Property 5: FilteredList cache identity on repeated call

*For any* application state, calling `getFilteredList()` twice without mutating `AppState.txs`, `AppState.txFilter`, or `AppState.searchQuery` between calls SHALL return the same array reference (strict `===` identity).

**Validates: Requirements 5.1, 5.2**

---

### Property 6: ThrottleGate prevents duplicate mutations

*For any* sequence of `n ≥ 2` add-transaction triggers fired within a 300 ms window, exactly one transaction SHALL be appended to `AppState.txs`; all subsequent triggers in that window SHALL be silently discarded and `AppState.txs.length` SHALL increase by exactly 1. The symmetric property holds for delete: any sequence of delete triggers within 300 ms against the same or different transaction IDs SHALL result in at most one deletion.

**Validates: Requirements 7.1, 7.2, 7.5**

---

### Property 7: LocalStorage keys read exactly once per session

*For any* user session (from `DOMContentLoaded` to page unload), `window.localStorage.getItem` SHALL be called with each of the three keys (`tabungan_onlen_transactions`, `tabungan_onlen_budget`, `tabungan_onlen_theme`) exactly once. All subsequent reads within the session SHALL use `AppState` fields exclusively.

**Validates: Requirements 11.1, 11.2**

---

### Property 8: DerivedCache populated before first render

*For any* stored transaction list, after `init()` completes the synchronous LocalStorage read block and calls `rebuildDerivedCache`, the first call to any render function SHALL observe a `DerivedCache` whose `totalInc` and `totalExp` already reflect the stored transactions. No render function call SHALL precede `rebuildDerivedCache` within the `init` execution.

**Validates: Requirements 11.4**

---

### Property 9: Lazy panel DOM not created before first tab activation

*For any* chart tab `t ∈ {donut, bar, flow}`, if the user has never activated tab `t` in the current session, then `LazyPanels[t].node` SHALL be `null` and the chart panel container SHALL contain no child elements associated with tab `t`. After the first activation, `LazyPanels[t].node` SHALL be a non-null DOM element, and activating the same tab a second time SHALL return the identical node reference without creating new DOM elements.

**Validates: Requirements 10.1, 10.2**

---

### Property 10: Chart recalculation skipped when txVersion unchanged

*For any* sequence of chart tab activations where `AppState.txVersion` does not change between two consecutive activations of the same tab, the chart data computation function for that tab SHALL NOT be invoked on the second activation. Conversely, if `AppState.txVersion` has incremented between activations, the computation function SHALL be invoked exactly once.

**Validates: Requirements 10.3, 10.4**

---

### Property 11: StorageQueue coalesces rapid saves

*For any* sequence of `n ≥ 2` state mutations enqueued within the same event loop tick (or before the idle callback fires), `localStorage.setItem` for the transactions key SHALL be called exactly once after the queue drains, with the value reflecting the final state after all `n` mutations. No intermediate states SHALL be written to `localStorage`.

**Validates: Requirements 8.2, 8.4**
