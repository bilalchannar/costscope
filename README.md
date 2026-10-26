# CostScope

CostScope is a frontend-only project cost dashboard built with plain HTML, CSS, and JavaScript. It runs in a modern browser without a build step or backend.

## Run it

1. Keep `index.html`, `styles.css`, and `app.js` together in one folder.
2. Open `index.html` in a modern browser. If browser file restrictions cause issues, serve the folder with any simple static file server.
3. Add line items manually or import a CSV/Excel workbook into Project 1. Optionally import a second CSV/Excel workbook into Project 2 for a side-by-side comparison. Project 2 is replaced independently when another file is selected. The first worksheet is used for `.xlsx` and `.xls` files.

The Excel reader (SheetJS) is loaded from a public CDN, so Excel import requires an internet connection. CSV import and manual entry work without it. Google Fonts also use a CDN.

## Cost calculation

Each line must have category, optimistic, most likely, pessimistic, and actual costs. All three estimates are editable directly in the table, along with the actual cost. For a new zero-valued row, enter the pessimistic estimate first, then most likely, then optimistic, so each edit keeps the estimates in order. Cost values must be finite and from 0 to $1,000,000,000,000; nonzero values below $0.000001 are rejected to prevent unstable percentage calculations. Blank and malformed values are rejected. At most 5,000 rows can be loaded.

PERT expected cost is `(Optimistic + 4 × Most likely + Pessimistic) / 6`.

Variance is `Actual − PERT expected`. Percentage variance is `Variance / PERT expected × 100`. When expected cost is zero, percentage variance is shown as **N/A** if actual cost is positive; if both are zero it is 0%. The total percentage uses the same rule.

## Imports

CSV files should have a header row. Required fields are `Category`, `Actual` (or `Spent`), and either all three PERT estimate columns (`Optimistic`, `Most likely`, `Pessimistic`) or one estimate/budget column. A single estimate column is used for all three PERT points. `Description` is optional; common alternatives such as `Item`, `Budget`, and `Spent` are recognized. Invalid rows are skipped with a count and first-row explanation. Quoted CSV fields and embedded commas/newlines are supported.

For Excel, the first worksheet is read. Formulas are interpreted using values stored in the workbook; the browser does not recalculate formulas.

## AI and privacy

The local cost summary works without a key. For extra AI commentary, paste a Gemini API key into the analyst panel, click **Save key**, then click **Analyze with Gemini**. The app currently requests Google's `gemini-3.8-flash` model. It sends the entered cost rows and the API key directly from the browser to Google only when you request analysis. The key is held only in JavaScript memory for that page session; it is not saved to local storage or embedded in the source. Clear or close the page to discard it. Anyone with access to the browser session can still inspect the active page, and Google receives the submitted data. Use sample or non-sensitive data for demos.

This frontend-only API-key approach is not suitable for a public deployment or confidential information. A production app should call an authenticated server-side service that protects the key, applies access controls, and handles provider errors. Model output is advisory and may be inaccurate; check any conclusions against the displayed calculations.

## Reports and limitations

**Download report** creates a styled standalone HTML report with print-friendly project summary cards and detailed tables. When both datasets are present it also includes overall and category-level comparisons; categories match case-insensitively, and differences are Project 2 minus Project 1. Imported text is HTML-escaped in the report and table. No data is uploaded unless you explicitly request Gemini analysis.

Data exists only in the current page session and is not saved to a database. Currency display is USD. Charts are simple canvas visualizations and do not add uncertainty distributions or probability ranges beyond the PERT expected-cost calculation.

See [PART-B-Cybersecurity-Analysis.md](PART-B-Cybersecurity-Analysis.md) for the threat scenarios, mitigations, deployment checklist, and safe test demonstrations.

