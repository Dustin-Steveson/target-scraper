# Product Requirements Document
## Target & Amazon Ratings Scraper

**Version:** 1.1  
**Date:** May 29, 2026  
**Status:** Draft

---

## 1. Overview

Target & Amazon Ratings Scraper is a client-side single-page application (SPA) that allows users to bulk-lookup star ratings and review counts for either Target.com or Amazon.com products. Users switch between platforms via brand icon buttons at the top of the page. Target lookups use TCINs; Amazon lookups use ASINs. Results are displayed in a table and can be exported to CSV.

---

## 2. Problem Statement

Analysts, buyers, and merchandisers who work with Target or Amazon products often need to audit ratings and review volume across large product sets. Manually visiting each product page is slow. Neither retailer exposes a public bulk-ratings API. This tool automates the lookup process directly from a browser — no backend infrastructure required.

---

## 3. Goals

- Allow bulk retrieval of star ratings and review counts for Target and Amazon products.
- Operate entirely client-side (no server, no login, no API key).
- Be fast enough for practical use via concurrent fetching.
- Export results to CSV for downstream analysis.

---

## 4. Non-Goals

- Price, availability, or inventory data are out of scope.
- No user authentication or saved history.
- No backend persistence or database.
- Not a browser extension; runs as a standalone HTML file.

---

## 5. Users

| User | Description |
|---|---|
| **Merchandiser / Buyer** | Needs to audit ratings for a portfolio of products before a review or reorder decision. |
| **Market Researcher** | Pulls competitive or category data for reporting, including cross-retailer comparisons. |
| **Developer / Analyst** | Tests the tool with known TCINs or ASINs and exports results to feed into a pipeline. |

---

## 6. Core Features

### 6.1 Mode Switcher
- Two brand icon buttons at the top of the page: Target (red bullseye) and Amazon (orange logo).
- Switching modes updates the page title, subtitle, input label, placeholder, column header, and color scheme.
- Target mode uses a red (`#cc0000`) accent; Amazon mode uses orange (`#e47911`).
- Switching modes clears any in-progress results.

### 6.2 Product ID Input
- Large textarea accepting one or more product IDs (TCINs for Target, ASINs for Amazon).
- Supports newline, comma, and space delimiters.
- Deduplicates entries automatically.
- Validates format per mode: TCINs are numeric 5–12 digits; ASINs are 10 alphanumeric characters.
- Hint text and example ID displayed to guide users.

### 6.3 Fetch Ratings
- Triggers on clicking **Fetch Ratings**.
- Fetches each product page via a CORS proxy (`api.codetabs.com`).
- Concurrency limit of 3 simultaneous requests to avoid overloading the proxy.
- **Target extraction strategy** (three layers):
  1. DOM attribute selectors (`data-test="totalGuestRating"`, `data-test="ratingCountText"`).
  2. Embedded `<script>` JSON blocks (multiple key name variants).
  3. Raw HTML regex fallback for SSR-embedded structured data.
- **Amazon extraction strategy** (five layers, using the mobile product page `/gp/aw/d/{ASIN}`):
  1. DOM: first `<span>` inside `[data-hook="acr-average-stars-rating-text"]` (desktop fallback).
  2. DOM: `#acrPopover` `title` attribute (e.g. `"4.2 out of 5 stars"`) — primary mobile source for rating.
  3. DOM: `#acrCustomerReviewText` `aria-label` attribute (e.g. `"6,697 Reviews"`) for review count.
  4. JSON-LD `<script>` blocks (`aggregateRating.ratingValue` / `reviewCount`).
  5. Raw HTML regex patterns for embedded JSON (`averageStarRating`, `totalRatingCount`, etc.).
- CAPTCHA / bot-check detection: if Amazon returns a challenge page, the row is marked with a clear error.

### 6.4 Results Table
- Columns: **ID** (TCIN or ASIN), **Product Page** (link), **Rating**, **# of Ratings**, **Status**.
- Rows appear immediately as pending (spinner) and update in real time as each fetch completes.
- Color coding: green for found data, red for errors, grey for not-found.
- Direct link to the product page on target.com or amazon.com for each row.

### 6.5 Progress Indicator
- Live counter showing `X / Y complete` while fetching.
- Final message on completion.
- Fetch button is disabled and relabeled "Fetching…" during operation.

### 6.6 CSV Export
- Appears only after a successful fetch run.
- Columns: `TCIN`/`ASIN`, `URL`, `Rating`, `Number of Ratings`, `Error`.
- Filename includes the retailer and current date (e.g., `target-ratings-2026-05-29.csv`).
- All fields are properly quoted and escaped.

### 6.7 Clear
- Resets textarea, table, progress, and internal state.

---

## 7. Technical Architecture

| Concern | Decision |
|---|---|
| **Runtime** | Browser only — no build step, no framework |
| **Language** | Vanilla HTML/CSS/JavaScript (ES2020+) |
| **CORS proxy** | `https://api.codetabs.com/v1/proxy` (both modes) |
| **Target URL** | `https://www.target.com/p/-/A-{TCIN}` |
| **Amazon URL** | `https://www.amazon.com/gp/aw/d/{ASIN}` (mobile endpoint — avoids bot detection) |
| **Concurrency** | `Promise.all` over batches of 3 |
| **Parsing** | `DOMParser` + regex fallbacks |
| **Mode state** | `MODES` config object; `setMode()` toggles CSS class and updates all DOM labels |
| **Export** | `Blob` + object URL download |

---

## 8. Constraints & Known Limitations

| Limitation | Detail |
|---|---|
| **CORS proxy dependency** | All requests route through a third-party public proxy. If codetabs.com is unavailable or rate-limits, fetches will fail. |
| **Amazon bot detection** | Amazon aggressively blocks proxy IPs on desktop product URLs (`/dp/`). The mobile endpoint (`/gp/aw/d/`) is used to reduce detection. Occasional CAPTCHA responses are still possible; affected rows will show a clear error. |
| **Dynamic content** | Some product pages load ratings via client-side JavaScript after initial HTML delivery. The proxy returns only the SSR HTML, so dynamically injected ratings may not be captured. |
| **ID validity** | Invalid or discontinued TCINs/ASINs will return "Not found" rather than an error. |
| **No caching** | Every run re-fetches all IDs; there is no local caching between sessions. |
| **Scale** | Not optimized for thousands of IDs in a single run; large batches will be slow due to concurrency limit and proxy rate limits. |

---

## 9. Success Metrics

- User can input 50 TCINs or ASINs and receive ratings for ≥80% of them within a reasonable time.
- CSV export contains correct data for all resolved rows.
- No crashes or JS errors for valid inputs.
- Mode switch is seamless with no residual state from the previous mode.

---

## 10. Future Considerations

- **Alternative / fallback proxy**: automatically retry via a secondary proxy on failure.
- **Official APIs**: if Target or Amazon expose official ratings APIs, migrate off the scraping approach.
- **Caching**: persist results in `localStorage` keyed by TCIN/ASIN to avoid redundant fetches.
- **Product name column**: extract product title from the fetched HTML.
- **Retry logic**: automatically retry failed rows with exponential backoff.
- **Rate limiting UX**: expose a configurable delay between batches.
- **Additional retailers**: extend the mode system to support Walmart, Best Buy, or other retailers.
- **Browser extension**: package as a Chrome extension to avoid CORS entirely.
