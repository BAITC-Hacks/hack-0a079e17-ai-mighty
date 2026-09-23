# Data audit — Электрокомплект supplier replenishment case

Initial inventory of the two supplied ZIP archives. Source archives were read only; no source files were modified or extracted into the repository.

## Files and observed structure

| Source | Sheet dimensions | Observed fields / purpose |
| --- | --- | --- |
| IEK — MOQ | 1,939 rows × 5 columns | 1C code, supplier SKU, product name, minimum shipment multiple |
| IEK — detailed sales dynamics | 171,605 rows × 8 columns | Date, document number/type, 1C code, product, unit, warehouse, quantity |
| IEK — monthly inventory history | 2,857 rows × 37 columns | Product, unit, 1C code, monthly quantities and beginning balances, Jan 2024–Sep 2026 |
| IEK — monthly sales quantities | 2,466 rows × 36 columns | Product, 1C code, monthly sales quantities, Jan 2024–Sep 2026 |
| IEK — orders in transit, dated 2026-09-22 | 2,642 rows × 9 columns | 1C code, IEK SKU, product, quantities grouped by six purchase-order documents and expected receipt dates |
| IEK — seasonality | 41 rows × 15 columns | Dedicated seasonality reference; multi-row header, needs closer inspection before mapping |
| Systeme Electric — MOQ | 557 rows × 5 columns | Product, 1C code, SKU, minimum order multiple |
| Systeme Electric — detailed sales dynamics | 77,314 rows × 8 columns | Same apparent structure as IEK detailed sales |
| Systeme Electric — monthly inventory history | 704 rows × 37 columns | Product, unit, 1C code, monthly quantities/balances, Jan 2024–Sep 2026 |
| Systeme Electric — monthly sales quantities | 557 rows × 38 columns | Product and 1C code with monthly quantities, Jan 2024–Sep 2026; workbook has a second sheet that needs review |
| Systeme Electric — orders in transit, dated 2026-09-22 | 499 rows × 70 columns | Workbook has two sheets; first sheet has a wide layout (A:BR), so exact order/quantity mapping needs review |
| Systeme Electric — seasonality | 35 rows × 15 columns | Dedicated seasonality reference; needs closer inspection |

## Findings and risks

- Both brands have enough sales history and per-SKU monthly sales aggregates to build a baseline demand forecast and estimate seasonality by product. The separate “seasonality” workbook in each archive is a **brand-level monthly sales summary**, not a per-SKU seasonality table; it is useful as a cross-check, not as a per-item forecast input.
- Sales detail appears to include date, SKU/code, warehouse, and quantity, but the observed headers do **not** include anonymized customer ID or price. Therefore, identifying a large one-off sale to the same customer and calculating monetary sales value are not currently supported by the visible schema. A robust fallback can flag unusually large transactions by SKU/date, but it cannot claim customer-level detection.
- IEK inventory history has monthly buckets explicitly labelled “beginning balance”; it is not a current stock snapshot. The latest visible bucket is September 2026, so current-on-hand inventory needs another source or a clearly stated approximation.
- Systeme Electric’s workbook dated 2026-09-22 contains a richer consolidated sheet with monthly sales, category, growth/seasonality coefficients, stock by locations, reserved/free stock, a suggested order, and an in-transit column. It covers 497 product codes versus 554 in its monthly sales/MOQ sheets. Treat its precomputed recommendation/coefficient fields as partner-provided signals to validate, not as ground truth. Its second sheet is a brand-level seasonality table.
- IEK transit quantities are grouped by six purchase-order documents with expected receipt dates. The Systeme Electric transit workbook is also a consolidated planning sheet, not merely a list of open orders. In both cases, check whether transit is outstanding and avoid double-counting receipts already reflected in on-hand stock.
- MOQ files appear to provide minimum shipment multiples, useful for rounding recommendations. They do not necessarily include supplier lead times or supplier ownership.
- Explicit stockout periods and supplier lead-time tables have not yet been identified in these archives. The seasonality sheets may include some derived signals, but that needs validation.
- The Excel entries show Cyrillic filenames garbled in the ZIP directory listing, likely due to legacy filename encoding. Spreadsheet contents and Russian headers were readable. The importer should not depend on archive entry filenames for data semantics.

## Code overlap check

Counts below use trimmed 1C product codes from non-empty data rows; intersection counts are exact string matches. “Rows” in a transaction export count transaction rows, while “unique codes” count distinct products.

| Brand | Source | Data rows with code | Unique codes |
| --- | --- | ---: | ---: |
| IEK | MOQ | 1,938 | 1,937 |
| IEK | Detailed sales | 171,603 | 2,151 |
| IEK | Monthly inventory history | 2,853 | 2,853 |
| IEK | Monthly sales | 2,463 | 2,463 |
| IEK | In-transit orders | 2,623 | 2,616 |
| Systeme Electric | MOQ | 554 | 554 |
| Systeme Electric | Detailed sales | 77,312 | 565 |
| Systeme Electric | Monthly inventory history | 701 | 701 |
| Systeme Electric | Monthly sales | 554 | 554 |
| Systeme Electric | Consolidated planning / transit sheet | 497 | 497 |

| Brand | Pair | Matching unique codes |
| --- | --- | ---: |
| IEK | MOQ ↔ detailed sales | 1,738 |
| IEK | MOQ ↔ monthly inventory | 1,854 |
| IEK | MOQ ↔ monthly sales | 1,761 |
| IEK | MOQ ↔ transit | 1,906 |
| IEK | detailed ↔ monthly inventory | 2,151 |
| IEK | detailed ↔ monthly sales | 2,140 |
| IEK | detailed ↔ transit | 1,940 |
| IEK | monthly inventory ↔ monthly sales | 2,456 |
| IEK | monthly inventory ↔ transit | 2,295 |
| IEK | monthly sales ↔ transit | 2,084 |
| Systeme Electric | MOQ ↔ detailed sales | 545 |
| Systeme Electric | MOQ ↔ monthly inventory | 554 |
| Systeme Electric | MOQ ↔ monthly sales | 554 |
| Systeme Electric | MOQ ↔ consolidated planning | 469 |
| Systeme Electric | detailed ↔ monthly inventory | 565 |
| Systeme Electric | detailed ↔ monthly sales | 545 |
| Systeme Electric | detailed ↔ consolidated planning | 470 |
| Systeme Electric | monthly inventory ↔ monthly sales | 554 |
| Systeme Electric | monthly inventory ↔ consolidated planning | 474 |
| Systeme Electric | monthly sales ↔ consolidated planning | 469 |

These joins are promising, but not complete. Duplicate codes exist in IEK MOQ (1 duplicate), IEK transit (7 duplicate rows), and transaction histories naturally repeat codes across transactions. Several workbooks contain products absent from another source; the import must preserve those rows and flag missing inputs instead of silently dropping them.

## Next audit steps

1. Confirm detailed-sales date ranges and transaction semantics, especially negative/return quantities.
2. Inspect duplicate codes and unmatched product lists; determine whether rows represent different units, warehouses, or inactive SKUs.
3. Resolve current on-hand stock and open-transit semantics, especially the date basis and whether in-transit figures are already reduced by receipts.
4. Define a normalized import schema and document missing inputs/fallbacks before implementing recommendations.
