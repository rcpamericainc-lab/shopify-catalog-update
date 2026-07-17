# Compare Catalogs

Use this prompt to run the comparison phase of a catalog refresh: diffing the
live Shopify catalog against a new supplier catalog and classifying every
product into Add / Update / Remove / Review. This is Phase 1 — it produces
report files only, never Shopify-import-ready CSVs (that's `generate_imports.md`).

## Inputs

- **Current Shopify export** — the live catalog pulled from Shopify, e.g.
  `01 Shopify Export/products_<date>.csv`. Full Shopify product CSV format,
  one row per variant, Handle repeated on every row, other product-level
  fields (Title, Vendor, Body HTML, etc.) blank on continuation rows.
- **Supplier catalog** — the new/updated price list from the supplier, e.g.
  `02 Current Catalog/<name>.xlsx`. Often messy: category names embedded as
  section-header rows, inconsistent column layout (`Description, MPN,
  Column1, Price` is typical), and sometimes literal junk rows (payment
  terms, contact info) mixed into the data. Always inspect the raw sheet
  before parsing — don't assume a clean rectangular table.

## Matching priority

When deciding whether a supplier row is the same product as an existing
Shopify product, match in this order and stop at the first hit:

1. **SKU / MPN** (exact match)
2. **Barcode / UPC** (exact match)
3. **Handle** (exact match)
4. **Product Title** (exact or high-confidence fuzzy match)

Never guess. If a match is uncertain (e.g. a fuzzy title match below your
confidence threshold, or a product that could plausibly be a rename of an
existing one), classify it as **Review** with a `Review_Reason` explaining
why, rather than forcing it into Add/Update/Remove. It's fine — expected,
even — for the same handle to appear more than once in a review report if
there are multiple plausible candidate matches; don't collapse those into a
single guess.

## Junk-row filtering

Before classifying anything, drop rows that aren't real products: section
header rows, blank rows, and rows with no SKU/MPN **and** no price (these are
almost always footer text like "Ordering Terms & Conditions", payment method
notes, or email/contact lines that got swept into the data range). Report
what you excluded and why — don't silently drop rows without a record.

## Classification rules

- **Add** — supplier row has no match in the current Shopify export.
- **Update** — supplier row matches an existing product, and something
  differs: price change, variant(s) added, variant(s) removed.
- **Remove** — an existing Shopify product/handle has no corresponding row
  anywhere in the supplier catalog (discontinued).
- **Review** — anything that isn't a confident match at any priority tier
  above, including suspected renames. Include a similarity note/score if
  you're using fuzzy text matching.

## Outputs

Save everything under `05 Final Imports/<date>/`:

- `catalog_change_summary.md` — human-readable summary with `## Added`,
  `## Removed`, `## Price Changes`, and `## Variant Changes` sections (plain
  bullet lists of product titles/details, not full data).
- `products_add.csv` — simple flat format for new products:
  `Base_Product, Category, Variant_Description, MPN, Cost, Price`. One row
  per variant; `Base_Product` groups variants of the same product together
  and should appear on contiguous rows.
- `products_update.csv` — one row per existing product that changed:
  `Handle, Title, Status, Category, Has_Price_Change, Has_Variants_Added,
  Has_Variants_Removed, Needs_Review, Review_Reason`. This is a flags file,
  not a values file — it says *what* changed, not the new numbers.
- `products_remove.csv` — one row per discontinued product:
  `Handle, Title, Status, Vendor, Variant_SKUs`.
- `pricing_update.csv` — granular per-SKU price deltas:
  `Handle, Title, Variant, SKU, Old_Price, New_Price, Change_Amount,
  Change_Pct, Match_Method`. Record `Match_Method` (SKU vs Title/Size) so
  downstream steps know how confident each match is.
- `variants_update.csv` — per-variant additions/removals:
  `Handle, Title, Change_Type (Added/Removed), Description, SKU_or_MPN, Price`.

## Notes for next time

- Keep the simple/flag-file formats above for this phase — don't jump
  straight to Shopify's full column schema here. That conversion happens in
  `generate_imports.md`, using these reports as input.
- If the supplier catalog's Handle-equivalent names don't cleanly match
  Shopify Titles (e.g. all-caps supplier names vs mixed-case Shopify
  titles), build an explicit case-mapping first (this project used the
  already-standardized `## Added` list in `catalog_change_summary.md` as an
  answer key) rather than auto-title-casing, which mangles brand
  abbreviations and acronyms.
