# Implementation Plan: Performance Optimization — Vanilla JS/CSS/HTML Rewrite

## Overview

Rewrite "Tabungan Onlen" from a single-file React 18 + Babel + Tailwind CDN bundle into a three-file (`index.html`, `app.css`, `app.js`) zero-dependency Vanilla HTML5/CSS3/ES6+ application. Each task builds incrementally on the previous one, ending with all components wired together and all 11 correctness properties covered by property-based tests.

## Tasks

- [ ] 1. Create `index.html` shell with CriticalCSS and static containers
  - [ ] 1.1 Write the HTML skeleton with `<meta>`, `<title>`, font `<link rel="preconnect">`, CriticalCSS `<style>` block (≤ 2 KB), `<link rel="stylesheet" href="app.css">`, and `<script src="app.js" defer>`
    - CriticalCSS must style `#root .loading-shell`, `loading-icon`, `loading-title`, `loading-sub`, and the `_float` keyframe
    - No Babel, React, or Tailwind CDN references anywhere in the file
    - Static containers: `<header id="app-header" hidden>`, `<main id="app-main" hidden>`, `<footer id="app-footer" hidden>`, `<div id="toast-container">`, `<div id="dialog-container">`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 2.1, 2.2, 2.3, 2.4, 3.1, 3.4, 12.1, 12.3, 12.4_

  - [ ]* 1.2 Write property test for CriticalCSS size invariant (Property 1)
    - **Property 1: CriticalCSS size invariant**
    - Assert `new TextEncoder().encode(criticalCssText).length <= 2048` — static assertion, no generator needed
    - **Validates: Requirements 2.3**

- [ ] 2. Create `app.css` with all migrated styles
  - [ ] 2.1 Write `app.css` with `@import` for Plus Jakarta Sans font (with `font-display: swap`), CSS reset, base body styles, glass morphism classes (`.glass`, `.dark .glass`), scrollbar styles (`.thin-scroll`), and dark mode variable (`html.dark`)
    - Match the visual design from the original: same glass-morphism backgrounds, border colours, blur values
    - _Requirements: 1.1, 12.4_

  - [ ] 2.2 Add all animation keyframes and utility classes to `app.css`
    - Keyframes: `float`, `jelly`, `popIn`, `slideDown`, `slideUp`, `slideFromLeft`, `slideFromRight`, `exitRight`, `growWidth`, `barGrow`, `numPulse`, `rippleOut`, `fadeIn`
    - Utility classes: `.anim-float`, `.jelly-once`, `.pop-in`, `.slide-down`, `.slide-up`, `.slide-left`, `.slide-right`, `.exit-right`, `.grow-width`, `.bar-grow`, `.num-pulse`, `.card-lift`, `.ripple-host`, `.ripple-host .rpl`
    - _Requirements: 9.3_

  - [ ] 2.3 Add component-level CSS classes to `app.css`
    - Layout: `.right-sticky` (sticky positioning ≥ 1024 px), responsive grid helpers, mobile touch targets (`.tap-lg`), `.dialog-backdrop`
    - Donut chart: `.donut-arc`, `.flow-node`
    - Theme colour transitions: `body`, `.glass`, `.glass *`
    - _Requirements: 9.3_

- [ ] 3. Scaffold `app.js` IIFE with §1–§3 (constants, formatters, LS wrapper)
  - [ ] 3.1 Write the outer IIFE (`(function() { 'use strict'; ... })()`), §1 constants (`KEYS`, `CATS`, `CAT`, `INCOME_IDS`, `EXPENSE_IDS`), and §2 formatter utilities (`rp`, `rpShort`, `clamp`, `uid`, `todayISO`)
    - All values must match the original — same category IDs, colours, emojis, and IDR formatter locale
    - _Requirements: 1.5_

  - [ ] 3.2 Write §3 `LS` localStorage wrapper (`get`, `set`, `getJSON`, `setJSON`) — each method wrapped in `try/catch`
    - _Requirements: 11.3_

- [ ] 4. Implement §4 `AppState` and §5 `DerivedCache`
  - [ ] 4.1 Write §4 `AppState` object with all fields: `txs`, `budget`, `dark`, `isLoaded`, `txFilter`, `searchQuery`, `txVersion`
    - _Requirements: 4.1_

  - [ ] 4.2 Write §5 `DerivedCache` object and `rebuildDerivedCache`, `_applyTxToCache`, `cacheAdd`, `cacheDel`, `cacheEdit` functions
    - Incremental helpers must update only affected fields (type, cat, date buckets)
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

  - [ ]* 4.3 Write property test for DerivedCache full rebuild (Property 2)
    - **Property 2: DerivedCache correctness after full rebuild**
    - Use `arbTxArray` fast-check generator; assert `totalInc`, `totalExp`, `netBal`, `catTotals`, `monthTotals` against manual sums
    - **Validates: Requirements 4.1, 4.5**

  - [ ]* 4.4 Write property test for incremental cache equivalence (Property 3)
    - **Property 3: Incremental cache equivalence**
    - `cacheAdd`, `cacheDel`, `cacheEdit` each produce state identical to `rebuildDerivedCache` from scratch
    - **Validates: Requirements 4.2, 4.3, 4.4**

- [ ] 5. Implement §6 `FilteredList` cache and §7 `StorageQueue`
  - [ ] 5.1 Write §6 `FilterState` object and `getFilteredList()` function with memoisation keys (`lastTxVersion`, `lastFilter`, `lastQuery`, `cachedResult`)
    - Cache miss triggers sort (descending by date) + filter + search; cache hit returns same array reference
    - _Requirements: 5.1, 5.2, 5.3_

  - [ ]* 5.2 Write property test for FilteredList descending sort (Property 4)
    - **Property 4: FilteredList is sorted descending by date**
    - For any `arbTxArray`, every adjacent pair `[a, b]` satisfies `a.date >= b.date`
    - **Validates: Requirements 5.3**

  - [ ]* 5.3 Write property test for FilteredList cache identity (Property 5)
    - **Property 5: FilteredList cache identity on repeated call**
    - Two consecutive calls without state mutation return strict `===` reference
    - **Validates: Requirements 5.1, 5.2**

  - [ ] 5.4 Write §7 `StorageQueue` (`handle` field) and `scheduleStorageSave()` function
    - Use `requestIdleCallback` with 2 000 ms timeout fallback to `setTimeout(0)`; cancel previous handle on re-enqueue; catch `DOMException` and show error toast
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

  - [ ]* 5.5 Write property test for StorageQueue coalescing (Property 11)
    - **Property 11: StorageQueue coalesces rapid saves**
    - `n ≥ 2` mutations enqueued before idle callback fires result in exactly one `setItem` call with final state
    - **Validates: Requirements 8.2, 8.4**

- [ ] 6. Checkpoint — core data layer complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Implement §8 DOM helpers
  - [ ] 7.1 Write `buildTxCard(tx, idx)` and `buildEmptyState()` DOM-builder functions that return `Element` nodes without touching the live DOM
    - Cards must include inline-event wiring for edit, delete, and save buttons; use CSS class toggling only for animations (`.pop-in`, `.exit-right`)
    - _Requirements: 9.1, 9.3_

  - [ ] 7.2 Write `renderTxList()` using `DocumentFragment` + `replaceChildren`; write `updateTextNode(elementId, newText)` text-node mutator
    - `renderTxList` calls `getFilteredList()` — never iterates `AppState.txs` directly
    - `updateTextNode` mutates `firstChild.nodeValue` when the first child is a text node; falls back to `textContent`
    - _Requirements: 4.6, 9.1, 9.4_

  - [ ] 7.3 Write `renderSummaryCards()`, `renderHeaderBudgetBar()`, `renderTopSpenders()`, and `renderAll()` functions
    - All reads from `DerivedCache`; no layout reads in the same synchronous call stack as DOM writes
    - `renderAll` calls each render function once; used at init and after import
    - _Requirements: 4.6, 9.2_

- [ ] 8. Implement §9 LazyPanel Registry
  - [ ] 8.1 Write `LazyPanels` registry object (`donut`, `bar`, `flow` entries with `node` and `renderedAtVersion`), `buildChartPanel(tabId)` skeleton builder, and `invalidateLazyPanels()` function
    - `buildChartPanel` creates the static SVG/HTML skeleton once; no data-bound values written here
    - `invalidateLazyPanels` sets all `renderedAtVersion` to `-1`
    - _Requirements: 10.1, 10.2_

  - [ ] 8.2 Write `renderChartData(tabId, node)` for all three chart types (donut, bar, flow)
    - Donut: update SVG `<path d>` attributes and legend text nodes from `DerivedCache.catTotals`
    - Bar: update SVG `<rect>` height/y attributes from `DerivedCache.monthTotals`
    - Flow: update flow-node amount text nodes from `DerivedCache` totals
    - _Requirements: 4.6, 10.3_

  - [ ] 8.3 Write `activateChartTab(tabId)` function with lazy-creation logic and version-skipping guard
    - First activation: create node, insert into `#chart-panel-container`, set `renderedAtVersion = txVersion`, call `renderChartData`
    - Subsequent activation with same `txVersion`: swap node into container, skip `renderChartData`
    - Subsequent activation with changed `txVersion`: call `renderChartData` and update `renderedAtVersion`
    - _Requirements: 10.1, 10.2, 10.3, 10.4_

  - [ ]* 8.4 Write property test for lazy panel deferred creation (Property 9)
    - **Property 9: Lazy panel DOM not created before first tab activation**
    - `LazyPanels[t].node === null` before any activation; non-null after first; same reference on second activation
    - **Validates: Requirements 10.1, 10.2**

  - [ ]* 8.5 Write property test for chart skip on same version (Property 10)
    - **Property 10: Chart recalculation skipped when txVersion unchanged**
    - `renderChartData` spy is not called on second activation when `txVersion` has not changed; called once when it has
    - **Validates: Requirements 10.3, 10.4**

- [ ] 9. Implement §11 event wiring — ThrottleGate, DebounceTimer, CRUD handlers, init
  - [ ] 9.1 Write `Throttle` object, `withAddThrottle(fn)`, and `withDelThrottle(txId, fn)` functions
    - Disable `#btn-add-tx` / all `.btn-del-tx` elements during the 300 ms window; silently discard re-entries
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5_

  - [ ]* 9.2 Write property test for ThrottleGate deduplication (Property 6)
    - **Property 6: ThrottleGate prevents duplicate mutations**
    - `n ≥ 2` add triggers within 300 ms window appends exactly 1 transaction; delete symmetric
    - **Validates: Requirements 7.1, 7.2, 7.5**

  - [ ] 9.3 Write `onSearchInput(value)` debounce handler (300 ms; immediate clear path for empty string)
    - _Requirements: 6.1, 6.2, 6.3_

  - [ ] 9.4 Write `addTransaction(tx)`, `deleteTransaction(id)`, `editTransaction(id, updates)`, `deleteAllTransactions()` mutation functions following the 5-step invariant ordering (mutate → increment `txVersion` → `cacheAdd`/`cacheDel`/`cacheEdit` → render → `scheduleStorageSave`)
    - Each function calls `invalidateLazyPanels()` after the cache update step
    - _Requirements: 4.2, 4.3, 4.4, 8.1, 8.2_

  - [ ] 9.5 Write form submit handler (validate `amt > 0`, call `withAddThrottle`, call `addTransaction`) and event listeners for filter pills, dark-mode toggle, export button, and import file input
    - Export handler serialises `AppState.txs` + `AppState.budget` into `{ version: 3, exportedAt, budget, transactions }` JSON blob and triggers download
    - Import handler validates `data.transactions` is an array before calling `rebuildDerivedCache` + `renderAll` + `scheduleStorageSave`
    - _Requirements: 3.3, 8.1_

  - [ ] 9.6 Write `init()` function: single synchronous `try/catch` block reads all three LS keys, applies defaults, sets `AppState`, calls `rebuildDerivedCache`, sets `AppState.isLoaded = true`, applies dark class, calls `renderAll`; wire to `document.addEventListener('DOMContentLoaded', init, { once: true })`
    - _Requirements: 3.3, 11.1, 11.2, 11.3, 11.4_

  - [ ]* 9.7 Write property test for LocalStorage single-read invariant (Property 7)
    - **Property 7: LocalStorage keys read exactly once per session**
    - Spy on `localStorage.getItem`; assert each of the three keys appears exactly once after `init()`
    - **Validates: Requirements 11.1, 11.2**

  - [ ]* 9.8 Write property test for DerivedCache populated before first render (Property 8)
    - **Property 8: DerivedCache populated before first render**
    - Wrap `renderAll` with a spy; assert `DerivedCache.totalInc` already equals stored income sum before spy's first invocation
    - **Validates: Requirements 11.4**

- [ ] 10. Checkpoint — full implementation wired together
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 11. Wire HTML skeleton to JavaScript render targets
  - [ ] 11.1 Update `index.html` with the full static HTML skeleton: header markup with `#budget-bar`, `#budget-pct`, `#budget-spent`; main with `#summary-income`, `#summary-expense`, `#summary-net`, `#tx-list-items`, `#chart-panel-container`, chart tab buttons; footer; confirm-dialog template inside `#dialog-container`
    - Verify all `id` attributes referenced in `app.js` render functions are present and match exactly
    - _Requirements: 1.1, 9.1, 9.4_

  - [ ] 11.2 Verify end-to-end wiring by opening `index.html` in a browser (no server required): loading indicator appears → app initialises → add a transaction → confirm summary cards update → toggle dark mode → export and re-import data
    - Fix any `getElementById` mismatches, missing event listeners, or CSS class name discrepancies found during this pass
    - _Requirements: 1.3, 3.3, 8.1, 9.1_

- [ ] 12. Final checkpoint — all requirements and tests verified
  - Ensure all 11 property-based tests and all unit tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP; they test correctness properties defined in the design document
- Each task references specific requirements for traceability
- Property tests use **fast-check** (`npm install --save-dev fast-check`) with a JSDOM environment; run with `npx jest --run` or `npx vitest --run`
- The 5-step mutation invariant (mutate → txVersion++ → cache → render → scheduleStorageSave) must be respected in every CRUD handler
- CriticalCSS uses `_float` (underscore prefix) to avoid colliding with the `float` keyframe defined in `app.css`
- No `type="module"` on the `<script>` tag — the IIFE pattern keeps everything encapsulated without ES module semantics

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "3.1"] },
    { "id": 1, "tasks": ["1.2", "3.2", "2.1"] },
    { "id": 2, "tasks": ["2.2", "4.1"] },
    { "id": 3, "tasks": ["2.3", "4.2"] },
    { "id": 4, "tasks": ["4.3", "4.4", "5.1"] },
    { "id": 5, "tasks": ["5.2", "5.3", "5.4"] },
    { "id": 6, "tasks": ["5.5", "7.1"] },
    { "id": 7, "tasks": ["7.2"] },
    { "id": 8, "tasks": ["7.3"] },
    { "id": 9, "tasks": ["8.1"] },
    { "id": 10, "tasks": ["8.2"] },
    { "id": 11, "tasks": ["8.3"] },
    { "id": 12, "tasks": ["8.4", "8.5", "9.1"] },
    { "id": 13, "tasks": ["9.2", "9.3", "9.4"] },
    { "id": 14, "tasks": ["9.5"] },
    { "id": 15, "tasks": ["9.6"] },
    { "id": 16, "tasks": ["9.7", "9.8", "11.1"] },
    { "id": 17, "tasks": ["11.2"] }
  ]
}
```
