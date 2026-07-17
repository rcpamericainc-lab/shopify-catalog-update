# Generate Import Files

Use this prompt to turn the comparison-phase reports (from `compare_catalogs.md`)
into Shopify-import-ready CSVs. This is Phase 2 — everything here should be a
file you could hand to Shopify's product importer without further editing.

## The single most important rule

**The column template authority is a real, live Shopify product export**
(e.g. `01 Shopify Export/products_<date>.csv`), **not** any hand-maintained
"master spreadsheet" working file, even if one exists and looks similar.

Before writing any output file, pull the header row from the live export and
verify:

- Exactly one column named `Status`, positioned last.
- Column count matches the live export exactly (79 columns as of this
  project; verify fresh each time since Shopify's schema can change,
  especially the metafield columns).

A working file called `master-spreadsheet.csv` in this project has a known
bug: it has **two** columns named `Status` (an extra blank one prepended to
the real header). Files built against that header will fail Shopify import
with "Status isn't valid" because the importer reads the first (blank)
Status column. If a reference file's header doesn't exactly match the live
export, treat the live export as correct and figure out the delta — don't
propagate the bug forward.

## Building `products_add.csv`

Input: the flat `products_add.csv` from the comparison phase (`Base_Product,
Category, Variant_Description, MPN, Cost, Price`).

1. Group rows by `Base_Product` (verify groups are contiguous first).
2. **Title**: use a verified proper-case mapping (see `compare_catalogs.md`
   notes) rather than blind `.title()`.
3. **Handle**: slugify the title (lowercase, non-alphanumeric → hyphen,
   collapse/trim hyphens). Dedupe within the batch by appending `-2`, `-3`,
   etc. if a slug repeats.
4. **Vendor**: confirm the convention from real export data before assuming
   — in this project all products use `RCP America` regardless of brand
   name printed on the product (e.g. IK-branded items are still Vendor =
   RCP America).
5. **Type**: map each source `Category` to the standardized bucket list in
   `04 Working Files/data-standardization.md` (Chemicals, Compounds, Waxes &
   Protectants, Air Fresheners, Dyes, Equipment, Equipment Parts &
   Accessories, Pads & Bonnets, Towels/Applicators & Brushes, Packaging &
   Containers, Shop Supplies, Apparel & Merch). Some source categories are
   mixed (e.g. a "MAXSHINE" or "IK SPRAYER" bucket containing both full
   machines and accessory parts) — build per-item overrides for those rather
   than bucketing the whole category one way.
6. **Option1 Name/Value**: if a product has only one variant and its
   description equals the base product name exactly, use `Title` /
   `Default Title`. Otherwise use `Size`, with the value being the
   description text after stripping the base-product prefix.
7. Leave unknown/unavailable fields blank rather than inventing content:
   `Body (HTML)`, `Tags`, `Product Category` (full taxonomy path), SEO
   fields, images, metafields, barcode. Flag these gaps to the user rather
   than guessing.
8. First row of each product group carries all product-level fields; every
   subsequent variant row for that product leaves Handle/Title/Vendor/etc.
   blank and only carries variant-level fields (Option1 Value, SKU, Price).

## Building `products_update.csv`

Input: the `products_update.csv` flags file, the live Shopify export (for
existing field values), and the current/updated supplier catalog (for new
values).

1. **Row count discipline**: this file is one row per *product*, not per
   variant — match the row count of the source flags file exactly
   (duplicates in the source, e.g. multiple review candidates for one
   handle, should be preserved as duplicate output rows, not deduped).
2. Match each flagged product to its existing block of rows in the live
   export using Handle first, Title as fallback.
3. Determine the new price using this priority:
   a. Direct SKU match against the current/updated supplier catalog.
   b. If the representative variant's SKU doesn't resolve, search *other*
      variants in that product's block for one that does resolve to an
      actual price change, and use that variant as the representative row
      instead (don't blindly default to the first variant if it has no
      applicable update).
   c. Fall back to any pre-computed pricing diff report (e.g.
      `pricing_update.csv`) for edge cases the direct catalog match misses
      (renamed SKUs, blank SKUs matched by title+size instead).
4. Carry every other field unchanged from the live export row.
5. Products flagged with variants added/removed can't be fully represented
   under the one-row-per-product cap — note this limitation explicitly
   rather than silently dropping the information, and offer to produce a
   separate uncapped variant-level file if the user wants full fidelity.
6. Products flagged `Needs_Review` should still get their price update if
   one resolves cleanly, but call out the review count/reasons in your
   summary rather than silently proceeding as if resolved.

## Building the archive/remove rows

Input: `products_remove.csv` (Handle, Title, Status=archived — status
already decided).

1. Match each handle to the live export (fallback to any broader reference
   file if the live export snapshot is missing a handle — verify the
   fallback source actually contains it before using it).
2. Copy the product's existing row(s) as-is. Status is a product-level
   field, so **one row per product is sufficient** — you don't need to
   touch every variant row to archive a product.
3. Only field that changes: the trailing `Status` column → `archived`.
   Everything else (price, options, metafields) stays untouched.
4. These rows get appended to `products_update.csv`, not written as a
   separate file, unless the user says otherwise.

## Before you finish

- Re-verify the output header against the live export header, field by
  field, not just column count.
- Confirm every data row has the same column count as the header.
- Confirm no row has a `Status` value outside `active`, `draft`, `archived`.
- Report exact row counts and any excluded/unresolved items in your summary.
