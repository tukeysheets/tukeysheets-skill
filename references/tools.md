# TukeySheets MCP tool catalog

Every tool is served by the running app at `http://127.0.0.1:31337/mcp` and
surfaces as `mcp__tukeysheets__<name>`. `sheet` parameters default to the
active sheet when omitted. The server is stateless — `tools/list` and
`tools/call` work over plain HTTP POST without an initialize handshake, so a
`curl` fallback works if the MCP client lost its registration mid-session.

## workspace — sheets and files

| Tool | Purpose |
|---|---|
| `workspace_get_active_context` | Active sheet, cursor cell, selected range. Cheap; call first. |
| `workspace_list_sheets` | Every open sheet with type (Table / DataView / Plot), dims, active flag. |
| `workspace_create_sheet` | New empty Table sheet. Returns the (possibly auto-uniqued) name. |
| `workspace_duplicate_sheet` | Clone a Table sheet — data only, no plots/pivots. |
| `workspace_rename_sheet` | Rename a tab; cross-sheet references keep resolving. |
| `workspace_delete_sheet` | Close and remove a sheet. Not undoable via the API; name stays reserved for the session. |
| `workspace_import_file` | Open `.csv` / `.parquet` / `.jsonl` / `.ndjson` / `.json` / `.xlsx` / `.tukey` as a new sheet. Absolute host path. |
| `workspace_export_sheet` | Write a sheet to CSV or Parquet. Overwrites. |

## data — reading and profiling

| Tool | Purpose |
|---|---|
| `data_preview` | Head + tail N rows as markdown. The cheapest first look. |
| `data_schema` | Per-column dtype, null/distinct counts, sample values (≤50 columns). |
| `data_profile_sheet` | One-call summary: dims, dtype histogram, missing, quantiles, top categoricals. |
| `data_profile_column` | Deep single-column profile: full quantiles, outlier count, sparkline. |
| `data_quick_info` | Multi-column stats **plus** cross-column views (correlation matrix, grouped boxplot, contingency) when 2+ columns are passed. |
| `data_read_range` | A1-range read as markdown/JSON/CSV. 2000-row cap; page with `offset`. `values: true` renders formula cells as evaluated results instead of raw `=` source. |
| `data_get_cell` | One cell: raw source (formula text) and evaluated value. |
| `data_list_validations` | Every validation rule on a sheet (range, kind, bounds, list items). |

## edit — writing cells

| Tool | Purpose |
|---|---|
| `edit_write_cells` | Bulk write values/formulas (leading `=`). Echoes raw stored values — formulas evaluate async; verify with `data_read_range values:true`. |
| `edit_evaluate_formula` | Evaluate a formula without writing it anywhere. |
| `edit_lookup_formula` | Docs for a formula by name (`SOLVE`, `SAMPLE.NORMAL`, `PROB.NORMAL.CDF`, …). |
| `edit_clear_range` | Wipe contents in a range (or `'all'`); structure preserved. |
| `edit_format_range` | Font/color/alignment/borders/number-format; only supplied fields change. |
| `edit_set_col_widths` | Pixel width (20–800) or `autofit: true` for a column run (`'B'`, `'A:G'`). Auto-fit measures displayed strings — run it last. |
| `edit_merge_cells` | Merge a range (`all`/`horizontal`/`vertical`), Merge & Center by default. Non-origin cells must be empty unless `force: true`. |
| `edit_unmerge_cells` | Remove every merge region overlapping a range. |
| `edit_set_validation` | Dropdowns and constraints (list/whole/decimal/date/time/textLength). Typed input only. |
| `edit_clear_validation` | Remove rules overlapping a range (removed wholesale). |
| `edit_insert_rows` / `edit_delete_rows` | 1-based row insert/delete. Delete indexing is quirky — see SKILL.md. |
| `edit_insert_col` / `edit_delete_cols` | Column-letter insert/delete. |
| `edit_rename_column` | Rename a header; literal-string formula references are NOT rewritten. |

## plot — TOML plot DSL

| Tool | Purpose |
|---|---|
| `plot_help` | DSL reference: TOML structure, canvas, mark catalog, faceting, multi-Y. Call once per plotting task. |
| `plot_mark_info` | Full docs + example for one mark. Its "known marks" list is incomplete — `plot_validate` is authoritative. |
| `plot_validate` | Parse/validate TOML without rendering. Cheap; run on all authored TOML. |
| `plot_render_preview` | Render TOML without persisting. Iterate here. |
| `plot_suggest` | 1–5 schema-appropriate starter specs with rationale. |
| `plot_create` | Persist a plot; returns plot ID + inline SVG. |
| `plot_update` | Replace an existing plot's spec in place. |
| `plot_list` | Plot IDs + current specs for a sheet. |
| `plot_export` | Export SVG (cached, byte-identical to the UI export). `path` + `inline: false` to avoid token waste. |
| `plot_delete` | Remove a plot by ID. |

## paper — Typst reports

| Tool | Purpose |
|---|---|
| `paper_help` | Authoring reference incl. `#sheet("Sheet!A1:D20")` / `#plot("Sheet!N")` embeds. Read first. |
| `paper_create` | New Paper, optionally seeded with `source` and placed in a `folder` (which forms its Typst import path). |
| `paper_list` | All papers with page counts and last-compile error flags. |
| `paper_get_source` / `paper_set_source` | Read/replace Typst source. Recompile is **async** (~10–15 s). |
| `paper_render` | Compile and return PNG/SVG/PDF + diagnostics. |
| `paper_export` | Write self-contained `.typ` or PDF to an absolute path. |

## stat — analysis

| Tool | Purpose |
|---|---|
| `stat_run_sql` | Read-only DuckDB SQL over any sheets (by sheet name). Result also opens as a temp tab. |
| `stat_summary` | describe(): count/mean/sd/quartiles/skew/kurtosis per column. |
| `stat_correlate` | Pearson/Spearman matrix (lower-triangular). |
| `stat_regress` | OLS: coefficients, SEs, t-stats, R². |
| `stat_test` | Welch's two-sample t-test. |
| `stat_outliers` | IQR or z-score flags with row indices and fences. |
| `stat_survival` | Kaplan-Meier; + log-rank p-value when `group_column` given. |
| `stat_power_analysis` | Exact power/sample-size/effect-size/α solving for t/anova/F²/χ²/proportions/correlation. No sheet data needed. |

## transform — materialize derived sheets

All except `transform_sort` create a **new** sheet and preserve the source;
all support `dry_run: true`. SQL arguments use DuckDB syntax — double-quote
columns, single-quote strings.

| Tool | Purpose |
|---|---|
| `transform_sort` | Sort by one column — **in place**, the only mutating transform. |
| `transform_filter` | WHERE-clause body filter. |
| `transform_select` | Project / rename / reorder columns (`{column, as}` items). |
| `transform_dedupe` | Drop duplicates on `keys` or full row; first occurrence kept. |
| `transform_fillna` | Fill NULL/empty cells with a scalar (string-typed; DuckDB casts). |
| `transform_derive` | Add a computed column from a SQL expression. |
| `transform_bin` | Bin a numeric column: equal_width / quantile / explicit edges. |
| `transform_union` | UNION ALL of compatible sheets (dedupe afterwards if needed). |
| `transform_join` | inner/left/right/outer join; exactly one of `on` or `condition` (`l.`/`r.` aliases). |

## connections — external warehouses

SQLite, PostgreSQL, ClickHouse, BigQuery, Snowflake. Schema listings read the
cached introspection only — an empty/stale cache needs a UI refresh.

| Tool | Purpose |
|---|---|
| `connections_list` | All connections: id, name, kind, endpoint summary (never credentials). |
| `connections_list_schemas` | Cached database/schema/table/column tree. |
| `connections_list_tables` | Flat table list with optional glob over `db.schema.table`. |
| `connections_run_sql` | Run SQL, rows inline (5000 cap) + Parquet on disk + temp tab. Read-only unless `allow_writes: true`. |
| `connections_materialize` | Land a query as a refreshable DataView sheet with lineage. |

## report — capture

| Tool | Purpose |
|---|---|
| `report_snapshot` | Markdown dump of a sheet: profile, schema, every plot (inline SVG optional), optionally written to a file. |
| `report_save_macro` | Persist a JS function as a named macro on the workbook (same name overwrites). |
