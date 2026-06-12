---
name: tukeysheets
description: >-
  Drive a running TukeySheets app through its MCP tools — read and edit sheets,
  write formulas, transform data, run statistics and SQL, build plots from the
  TOML plot DSL, author Typst "Paper" reports and slide decks, and query
  external warehouse connections. Use whenever the user asks to work with
  TukeySheets, a .tukey workbook, or wants data analysis, charts, reports, or
  presentations built inside the live spreadsheet app.
compatibility: >-
  Requires a running TukeySheets desktop app serving MCP at
  http://127.0.0.1:31337/mcp (tools appear as mcp__tukeysheets__*). Slide-deck
  authoring needs network access on first compile to download Typst packages.
---

# TukeySheets

TukeySheets is a desktop spreadsheet/data-analysis app. A running instance
serves an MCP server over HTTP at `http://127.0.0.1:31337/mcp`; its tools
appear as `mcp__tukeysheets__*`. The full tool catalog is in
[references/tools.md](references/tools.md); professional formatting
conventions (financial model styling, number formats, borders) are in
[references/formatting.md](references/formatting.md); building Typst slide
decks from sheet data and plots is in
[references/slides.md](references/slides.md).
Everything below assumes the app is running —
if the tools are missing, ask the user to launch TukeySheets first.

The project lives **in memory**. An app crash or restart drops every sheet,
plot, and paper, so re-import data after a relaunch and encourage saving.

## Golden rules

1. **Orient before acting.** Start with `workspace_get_active_context` (active
   sheet, cursor, selection — cheap) and `workspace_list_sheets`. For an
   unfamiliar sheet, call `data_profile_sheet` once instead of chaining
   `data_read_range` calls.
2. **Data starts at row 2.** Row 1 always holds the header (imported CSVs
   without a header get a synthesized `column0,column1,…` row 1). Use
   letter+row ranges that start at row 2 — `A2:A7044` for 7043 records.
   Ranges that include row 1 silently corrupt grouped/survival plots: numeric
   reads drop the header text but text reads keep it, so columns come back
   ragged by one row (phantom legend entries, scrambled curves).
3. **Prefer markdown reads.** `data_read_range` and the SQL tools default to
   markdown output — far cheaper in tokens than JSON. Reads are capped at
   2000 rows; page with `offset`, or use `data_preview` for a head+tail.
4. **Transforms are non-destructive** (except `transform_sort`, which sorts in
   place). Every other `transform_*` materializes a **new** sheet and
   preserves the source. Use `dry_run: true` to see the resulting row count
   before committing. Chain transforms by feeding each output sheet name into
   the next call.
5. **Validate, preview, then commit plots.** `plot_validate` → 
   `plot_render_preview` → `plot_create`. Never guess TOML field names: call
   `plot_help` once per plotting task and `plot_mark_info` for the specific
   mark.
6. **Suppress inline payloads when exporting.** `plot_create`, `plot_export`,
   and `paper_render` return inline SVG/base64 by default and responses can
   be huge. When writing to disk, pass `path` plus `inline: false`.
7. **Verify writes.** `edit_write_cells` echoes the **raw** stored value, not
   the evaluated result — formulas evaluate asynchronously after the batch
   commits. To confirm formula outputs, follow up with `data_read_range`
   passing `values: true` (or `data_get_cell` for one cell). Sanity-check
   unfamiliar or complex formulas with `edit_evaluate_formula` before
   committing them.

## Core workflows

### Explore a dataset

1. `data_profile_sheet` — dims, dtypes, missing values, quantiles, top
   categorical values.
2. `data_quick_info` with 2+ columns — per-column stats **plus** correlations,
   contingency tables, and grouped comparisons in one call. Prefer it over
   chaining `data_profile_column` when relationships matter.
3. `data_profile_column` for a deep dive on one flagged column (full
   quantiles, outlier count, sparkline).
4. `stat_correlate` across numeric columns to surface relationships, then
   plot the strongest findings.

### Edit cells and formulas

- `edit_write_cells` bulk-writes values; a leading `=` marks a formula.
- Formula language is TukeySheets's own. Before writing unfamiliar `SOLVE`,
  `SAMPLE.*`, `SEQ.*`, `PROB.*`, `COHEN.*`, regex, or text-slicing formulas,
  call `edit_lookup_formula` to confirm the signature.
- `edit_format_range` only changes the style fields you pass; everything else
  is preserved. Number formats use Excel-style codes (`'0.00%'`,
  `'$#,##0.00'`, `'yyyy-mm-dd'`).
- Dropdowns/constraints: `edit_set_validation` (list/whole/decimal/date/
  time/textLength). Enforcement applies to typed input only — formulas,
  paste, and fill bypass it. Check existing rules first with
  `data_list_validations`; overlapping rules are replaced wholesale.

### Format a sheet professionally

Whenever you build a deliverable — a model, summary table, or anything the
user will look at — finish with a formatting pass. Read
[references/formatting.md](references/formatting.md) for the conventions; the
short version:

1. Layer styles broad-to-narrow: number formats on whole columns first, then
   header/total/accent overlays (`edit_format_range` only touches the fields
   you pass).
2. Financial color code: blue font for hardcoded inputs, black for formulas,
   green for cross-sheet links, gray italic for notes.
3. Negatives in parentheses (`$#,##0;($#,##0)`), one number format and
   decimal precision per column; multi-section codes and `[Red]` work.
4. Borders sparingly: medium bottom border on the header, thin top border +
   bold on totals — never `border: "all"`.
5. Merge the title and section banners across the table width with
   `edit_merge_cells` (Merge & Center by default).
6. Finish with `edit_set_col_widths` + `autofit: true` over the table's
   columns — **always last**, since number formats change the rendered
   strings auto-fit measures. A model with default 80 px label columns reads
   as unfinished.

### Clean and transform

`data_profile_sheet` to find dirty columns, then: `transform_dedupe` →
`transform_fillna` (value is a string; DuckDB casts per column) →
`transform_derive` with `CAST()` for type coercion. Quote column names with
double quotes and string literals with single quotes in all SQL arguments:
`"Revenue" >= 1000 AND "Region" = 'EU'`.

### SQL across sheets

`stat_run_sql` queries any sheet by its sheet name as a DuckDB table
(read-only; pass extra sheet names in `sheets` to join across sheets). The
full result also opens as a temporary accept/discard tab in the app. For
joins/unions you intend to keep, prefer `transform_join` / `transform_union`.

### Plot

1. `plot_help` once — TOML structure, canvas options, mark catalog, faceting.
2. `plot_suggest` for schema-appropriate starter specs, or `plot_mark_info`
   for the mark you intend to use.
3. Author TOML with **sheet-prefixed, row-2-anchored A1 ranges** for aes
   columns (bare column names like `body_mass_g` silently render empty).
4. `plot_validate` (authoritative — trust it over `plot_mark_info`'s mark
   list), `plot_render_preview` to iterate, `plot_create` to commit.
5. Manage with `plot_list` (IDs + specs), `plot_update`, `plot_export`,
   `plot_delete`.

Mark aliases that appear in older examples: `filled_contour` → `contour` +
`filled=true`; `errorbar` → `linerange`; `rect` renders via the **tile**
builder (consumes `x,y,fill`, not xmin/xmax); `spoke` via the **segment**
builder (consumes `x,y,xend,yend`); `polygon` has no equivalent.

### Statistics

- `stat_summary` — describe() for numeric columns.
- `stat_test` — Welch's t (two groups); `stat_regress` — OLS with dummy-coded
  predictors for 3+ groups.
- `stat_outliers` — IQR or z-score; pair with `transform_filter` to drop.
- `stat_survival` — Kaplan-Meier + log-rank when `group_column` is given.
- `stat_power_analysis` — exact power/sample-size/effect-size/alpha solving;
  pure computation, no sheet needed.

### Papers (Typst reports)

1. `paper_help` **before** writing any paper source.
2. `paper_create` / `paper_set_source`. Embed live data with
   `#sheet("Sheet 0!A1:D20")` and plots with `#plot("Sheet 0!1")`.
3. **Recompiles are async:** `paper_set_source` returns `ok` immediately but
   the compile takes several seconds — `paper_export`/`paper_list` called
   right after serve the *previous* revision. Wait ~10–15 s before exporting
   to verify.
4. Typst gotchas: `\$` is only an escape in markup content (`[...]`); inside
   string args use plain `$`. Pin footnotes with `place(bottom + left, ...)`.

### Slide decks

A deck is a Paper using a presentation package — `@preview` imports (e.g.
typslides) download and cache on first compile. Read
[references/slides.md](references/slides.md)
for the skeleton and slide-craft rules; the short version: decide the
findings first, build deck-sized plots with `plot_create`, embed live data
with `#plot`/`#sheet`/`#cell` instead of typing numbers, iterate with
`paper_render` page-by-page, and check page count equals slide count —
typslides overflow flows onto extra pages instead of clipping.

### Warehouse connections

`connections_list` → `connections_list_tables` (cached schema only — a stale
or empty cache needs a UI refresh by the user) → `connections_run_sql` for
ad-hoc queries (read-only unless `allow_writes: true`) or
`connections_materialize` to land results as a refreshable DataView sheet.

### Import / export / report

- `workspace_import_file` accepts `.csv`, `.parquet`, `.jsonl`, `.ndjson`,
  `.json`, `.xlsx`, `.tukey` (absolute path on the host).
- `workspace_export_sheet` writes CSV/Parquet; `paper_export` writes
  `.typ`/PDF; both overwrite existing files.
- `report_snapshot` — one-call markdown dump of a sheet's profile, schema,
  and every plot; the "export everything you know" call after exploration.

## Gotchas

- **Deleted sheet names stay reserved** for the session: re-creating one
  auto-uniques to `name (2)`, and renaming back silently no-ops. Pick fresh
  names or accept the suffix. All name-taking tools may auto-unique — always
  use the name **returned** by the call, not the one you asked for.
- **`edit_delete_rows` row indexing is non-obvious** (deleting `row_start: 1`
  has removed an unrelated row). Prefer row-2-anchored ranges over deleting
  the header row.
- `edit_rename_column` does **not** rewrite formulas that referenced the old
  name as a literal string.
- `workspace_duplicate_sheet` clones data only — plots and pivot configs are
  not copied.
- Renders are single-threaded; very large plot/paper renders briefly block
  the app.
