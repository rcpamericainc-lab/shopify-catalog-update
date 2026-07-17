# QA Validation

Use this prompt as a final checklist before handing off any Shopify import
CSV (from `generate_imports.md`) for actual import. Run every check
programmatically — don't eyeball a sample and call it done.

## 1. Header integrity (the check that matters most)

- Pull the header row from the file you're about to ship and from the
  current live Shopify export. **Compare them field-by-field, not just by
  count.**
- Confirm `Status` appears **exactly once**, as the last column. Two columns
  named `Status` is the specific bug that caused a prior import to fail with
  "Status isn't valid" — the importer read the first (blank) one. Any
  reference/working file used as a template should itself be checked against
  the live export before you trust its header.
- Confirm the column count matches the live export exactly.

## 2. Row shape

- Every data row must have the same number of fields as the header. Flag and
  investigate any row that doesn't (usually an unescaped quote/comma issue).
- No trailing blank rows.

## 3. Status values

- Every `Status` value must be exactly one of `active`, `draft`, `archived`
  — nothing blank, nothing else.

## 4. Handle/variant grouping

- For multi-row products, confirm Handle is populated on every row
  (Shopify's real export repeats Handle on every variant row — it's Title,
  Vendor, Body HTML, etc. that go blank on continuation rows, not Handle).
- Confirm each Handle's rows are contiguous (no product's rows split across
  two separate blocks).
- Check for accidental duplicate Handles across otherwise-unrelated product
  groups (would merge two different products into one on import).

## 5. Required variant fields

- Every variant row should have a `Variant SKU` and `Variant Price`, unless
  you've already confirmed that gap pre-exists in the live catalog (report
  pre-existing gaps, don't silently "fix" them by inventing values).
- `Option1 Name`/`Option1 Value` should be `Title`/`Default Title` for
  single-variant products, otherwise a real option name (e.g. `Size`) with a
  matching value — never blank on the first variant row.

## 6. Price sanity cross-check

- For any row where price was updated, cross-check the new value against an
  independent source (e.g. a pre-computed pricing diff report, or the
  original supplier catalog by SKU) for a sample — ideally all — of changed
  rows. Flag any mismatch rather than trusting a single derivation path.

## 7. Row-count reconciliation

- Compare the output row count against the expected count from the
  comparison-phase source file (e.g. `products_add.csv` group count,
  `products_update.csv` line count). Explain any difference — don't let
  counts silently drift.

## 8. Junk/edge-case rows

- Confirm no non-product rows (footer text, section headers, blank
  descriptions) made it into the final file. Cross-check against the
  junk-row exclusion list from the comparison phase.

## 9. Needs_Review items

- List every product still flagged `Needs_Review` (and its reason) in your
  final summary. These should never be silently resolved one way or the
  other — surface them for a human decision.

## 10. File-permission sanity (Cowork-specific)

- If writing into a locked/delivered folder fails with a permission error,
  don't assume it's fixed just because a delete-approval prompt was shown —
  confirm the actual file operation succeeds before reporting the fix as
  done. If the user declines the overwrite, keep the corrected version
  available and say so plainly instead of leaving the broken file in place
  unremarked.

## Sign-off

Only report a file as ready when all ten checks above pass. If any check
fails and you can't resolve it confidently, say so explicitly and describe
exactly what's wrong rather than shipping a "probably fine" file.
