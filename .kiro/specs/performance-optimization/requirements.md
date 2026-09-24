# Requirements Document

## Introduction

This feature rewrites "Tabungan Onlen" — an Indonesian savings and expense tracker — from a single-file React 18 + Babel standalone + Tailwind CDN bundle into a multi-file Vanilla HTML + CSS + JavaScript application. The rewrite targets measurable improvements in first-visit load time, runtime responsiveness, and localStorage efficiency by eliminating runtime compilation overhead, splitting assets for browser caching, caching derived values, debouncing user inputs, and batching DOM mutations.

## Glossary

- **App**: The Tabungan Onlen web application delivered as multiple static files.
- **Shell**: The HTML entry-point file (`index.html`) that bootstraps the App.
- **StyleSheet**: The external CSS file (`app.css`) containing all styles.
- **ScriptModule**: The external JavaScript file (`app.js`) containing all application logic.
- **CriticalCSS**: The minimal inline CSS placed inside `<style>` in the Shell required to render the above-the-fold loading state without a flash of unstyled content.
- **DerivedCache**: The in-memory object that stores pre-computed aggregates (total income, total expense, net balance, per-category totals, per-month bar data) derived from the current transaction list.
- **FilteredList**: The subset of transactions that matches the active type filter and search query, produced by the filtering pipeline.
- **SearchInput**: The text `<input>` element used to filter the transaction list by description or category name.
- **DebounceTimer**: A pending `setTimeout` handle used to delay execution of a search recomputation until the user pauses typing.
- **ThrottleGate**: A flag or timestamp used to prevent add/delete operations from firing more than once per 300 ms when triggered repeatedly.
- **StorageQueue**: A pending `requestIdleCallback` or `setTimeout(0)` handle used to batch localStorage writes.
- **DOMBatch**: A single call to a DOM-update function that applies all pending view changes in one reflow cycle.
- **LazyPanel**: A chart or visualisation panel whose rendering is deferred until the user navigates to its tab.
- **LocalStorage**: The browser's `window.localStorage` API used to persist transactions, budget, and theme under the keys `tabungan_onlen_transactions`, `tabungan_onlen_budget`, and `tabungan_onlen_theme`.

---

## Requirements

### Requirement 1: Multi-File Asset Structure

**User Story:** As a returning user, I want the browser to cache the stylesheet and script independently of the HTML, so that repeat visits load without re-downloading unchanged assets.

#### Acceptance Criteria

1. THE App SHALL be delivered as exactly three files: `index.html` (Shell), `app.css` (StyleSheet), and `app.js` (ScriptModule).
2. THE Shell SHALL load the StyleSheet via a `<link rel="stylesheet" href="app.css">` tag placed in `<head>`.
3. THE Shell SHALL load the ScriptModule via a `<script src="app.js" defer>` tag placed in `<head>`; the `<script>` tag SHALL NOT include a `type="module"` attribute.
4. THE Shell SHALL NOT contain any `<script type="text/babel">` tag, Babel CDN reference, React CDN reference, or Tailwind CDN reference.
5. THE App SHALL NOT depend on any build tool, bundler, Node.js process, or CDN-delivered resource at runtime, including static `import` statements inside the ScriptModule that reference a CDN URL.
6. THE Shell SHALL NOT contain any inline `<style>` block or inline `<script>` block outside of the CriticalCSS `<style>` block defined in Requirement 2.

---

### Requirement 2: Critical CSS Inlining

**User Story:** As a first-time visitor, I want to see a styled loading indicator immediately on page open, so that the page does not appear blank while the stylesheet downloads.

#### Acceptance Criteria

1. THE Shell SHALL contain an inline `<style>` block in `<head>` with CriticalCSS that, at minimum, sets `body` background colour, centres the loading indicator, and applies the loading spinner or placeholder font.
2. WHEN the Shell is rendered before `app.css` has finished downloading, THE App SHALL display a visually styled loading state using only CriticalCSS.
3. THE CriticalCSS block SHALL contain no more than 2 KB of CSS text (unminified).
4. THE StyleSheet SHALL be loaded with `<link rel="stylesheet" href="app.css">` without a `media` trick or `onload` swap that would cause a flash of unstyled content after load.

---

### Requirement 3: Deferred and Non-Blocking Script Loading

**User Story:** As a user on a slow connection, I want the page to become interactive as early as possible, so that I can read the UI before all scripts execute.

#### Acceptance Criteria

1. THE ScriptModule SHALL be referenced with the `defer` attribute so HTML parsing completes before script execution begins.
2. THE App SHALL NOT use `document.write` or synchronous `<script>` tags in `<body>` that block rendering.
3. WHEN the `DOMContentLoaded` event fires, THE ScriptModule SHALL begin initialising the application state and rendering the first view.
4. THE App SHALL NOT load any third-party JavaScript library (including polyfills) from a CDN at runtime.

---

### Requirement 4: DerivedCache: Computed Value Caching

**User Story:** As a user who adds or edits transactions, I want summary totals and chart data to update instantly without the browser recalculating everything from scratch, so that the UI remains responsive regardless of transaction list size.

#### Acceptance Criteria

1. THE App SHALL maintain a DerivedCache object in memory that stores: total income, total expense, net balance, per-category expense totals (object keyed by category id), and per-month income/expense totals (object keyed by `YYYY-MM`).
2. WHEN a transaction is added, THE App SHALL update only the DerivedCache entries affected by that transaction's `type`, `cat`, and `date` fields, without recomputing unaffected entries.
3. WHEN a transaction is deleted, THE App SHALL update only the DerivedCache entries that included the deleted transaction, without recomputing unaffected entries.
4. WHEN a transaction's `amt` or `desc` field is edited, THE App SHALL recompute only the DerivedCache entries for that transaction's `type`, `cat`, and `date`.
5. WHEN the full transaction list is replaced via import, THE App SHALL recompute all DerivedCache entries once from the new list.
6. THE App SHALL read summary values exclusively from DerivedCache when rendering the summary cards, header budget bar, top-spenders list, donut chart data, and bar chart data.

---

### Requirement 5: FilteredList Caching

**User Story:** As a user scrolling through the transaction history, I want filter and search results to remain stable until I explicitly change the filter or search query, so that the list does not re-render unnecessarily.

#### Acceptance Criteria

1. THE App SHALL cache the FilteredList result in memory and invalidate it only when the transaction list changes, the active filter tab changes, or the SearchInput value changes after the DebounceTimer fires.
2. WHEN none of those three conditions have changed since the last render, THE App SHALL reuse the cached FilteredList without re-sorting or re-filtering.
3. THE App SHALL sort the FilteredList by transaction date descending exactly once per invalidation, not on every render cycle.

---

### Requirement 6: Search Input Debouncing

**User Story:** As a user typing in the search box, I want the list to update after I pause typing rather than on every keystroke, so that the UI does not stutter during fast input.

#### Acceptance Criteria

1. WHEN the user types into SearchInput, THE App SHALL start or reset a DebounceTimer with a 300 ms delay before triggering a FilteredList recomputation.
2. WHEN the DebounceTimer fires, THE App SHALL recompute FilteredList using the current SearchInput value and update the displayed transaction list in a single DOMBatch.
3. WHEN the user clears SearchInput completely, THE App SHALL cancel any pending DebounceTimer and immediately display the unfiltered list without waiting 300 ms.

---

### Requirement 7: Add/Delete Throttling

**User Story:** As a user who taps the add or delete button rapidly, I want the App to process each operation reliably without duplicating or skipping entries, so that my data stays accurate.

#### Acceptance Criteria

1. WHEN the user triggers an add transaction action, THE App SHALL apply a ThrottleGate that prevents a second add from being processed within 300 ms of the first.
2. WHEN the user triggers a delete transaction action, THE App SHALL apply a ThrottleGate that prevents a second delete from being processed within 300 ms of the first.
3. WHILE a ThrottleGate is active, THE App SHALL set the `disabled` attribute on the corresponding submit or delete button element.
4. WHEN 300 ms have elapsed since the last add or delete action, THE App SHALL remove the `disabled` attribute from the button and clear the ThrottleGate.
5. WHEN a ThrottleGate is active and the user triggers the same action again, THE App SHALL silently discard the duplicate trigger without queuing it for later execution.

---

### Requirement 8: Optimistic UI for localStorage Writes

**User Story:** As a user adding or deleting a transaction, I want the list to update immediately on screen, so that I do not perceive any lag from the localStorage write.

#### Acceptance Criteria

1. WHEN the user confirms an add, edit, or delete action, THE App SHALL update the in-memory transaction array and re-render the affected DOM nodes before writing to LocalStorage.
2. WHEN the user confirms an add, edit, or delete action, THE App SHALL enqueue the LocalStorage write using `setTimeout(fn, 0)` or `requestIdleCallback(fn)` so that the write callback executes after the DOM update has been applied.
3. WHEN a StorageQueue callback fires, THE App SHALL write the full serialised transaction array to `tabungan_onlen_transactions` and the budget value to `tabungan_onlen_budget` in a single synchronous block.
4. IF a new StorageQueue entry is enqueued before the previous one has fired, THE App SHALL cancel the previous entry and replace it with the new one, ensuring only the latest state is written.
5. IF a StorageQueue write throws a `DOMException` (e.g., storage quota exceeded), THE App SHALL catch the exception, display a toast notification informing the user that data could not be saved, and retain the in-memory state unchanged.

---

### Requirement 9: DOM Batching and Reflow Minimisation

**User Story:** As a user on a mid-range mobile device, I want list and chart updates to render smoothly without visible jank, so that interactions feel fluid.

#### Acceptance Criteria

1. WHEN re-rendering the transaction list after any state change, THE App SHALL build a `DocumentFragment` containing all new transaction card elements and replace the existing list container's children with a single `replaceChildren(fragment)` call.
2. THE App SHALL NOT read a layout property (e.g., `offsetWidth`, `getBoundingClientRect`, `clientHeight`) and then write a DOM property or attribute within the same synchronous call stack.
3. WHEN animating a transaction card entry or exit, THE App SHALL use CSS class toggling rather than JavaScript-driven `style` property writes to trigger transitions.
4. WHEN updating summary card values, THE App SHALL update only the text node of the changed value, not re-render the entire summary card markup.

---

### Requirement 10: Lazy Chart Panel Rendering

**User Story:** As a user who only views the transaction list, I want the chart visualisations to not consume CPU time until I navigate to a chart tab, so that the default view loads faster.

#### Acceptance Criteria

1. WHEN the user first activates a chart tab (Donut, Bar, or Cash Flow), THE App SHALL create and insert the chart panel's DOM elements at that point; THE App SHALL NOT create those elements during initial page load.
2. WHEN a LazyPanel has been rendered once, THE App SHALL cache its DOM node and reuse it on subsequent tab activations without re-creating elements from scratch.
3. WHEN the active chart tab changes, THE App SHALL update only the chart data within the existing LazyPanel DOM node if the panel was previously rendered.
4. WHEN the user activates a previously rendered chart tab and the transaction list version counter has not incremented since that panel was last rendered, THE App SHALL NOT recalculate chart segment or bar data.

---

### Requirement 11: LocalStorage Read Efficiency

**User Story:** As a user reopening the app, I want all persisted data to be loaded in a single initialisation pass, so that the app starts up quickly and does not make redundant storage reads.

#### Acceptance Criteria

1. WHEN the App initialises, THE App SHALL read all three LocalStorage keys (`tabungan_onlen_transactions`, `tabungan_onlen_budget`, `tabungan_onlen_theme`) in a single synchronous block before rendering any dynamic content.
2. THE App SHALL read each LocalStorage key exactly once per session startup; subsequent reads during the session SHALL use in-memory state only.
3. IF a LocalStorage read throws an exception, THE App SHALL catch the exception, apply the following defaults — empty array for `tabungan_onlen_transactions`, `5000000` for `tabungan_onlen_budget`, `'light'` for `tabungan_onlen_theme` — and continue initialisation without displaying an error to the user.
4. WHEN the App initialises from LocalStorage data, THE App SHALL populate DerivedCache from the loaded transaction list before the first render, so that no post-render recomputation is needed.

---

### Requirement 12: First-Visit Load Time

**User Story:** As a first-time visitor on a standard mobile connection, I want the App to display meaningful content quickly, so that the page feels fast and does not frustrate me with a blank screen.

#### Acceptance Criteria

1. THE Shell SHALL NOT reference any external resource (font, icon, library) that is required for the above-the-fold loading state; all such resources SHALL either be inlined in CriticalCSS or deferred.
2. WHEN measured using browser developer tools Network throttling set to 'Fast 4G' (approx. 20 Mbps download, 20 ms RTT), THE App SHALL display the CriticalCSS loading indicator element (as defined in Requirement 2) within 500 ms of navigation start.
3. THE StyleSheet SHALL be the only render-blocking external resource referenced in `<head>`; all other external resources (scripts, fonts) SHALL use `defer`, `async`, `rel=preload`, or `rel=preconnect` attributes.
4. WHERE web fonts are used, THE Shell SHALL include a `<link rel="preconnect">` hint for the font origin and a `font-display: swap` declaration in the StyleSheet to prevent invisible text during font load.
