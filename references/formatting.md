# Professional formatting guide

How to make a TukeySheets sheet look like a polished financial model or
analyst deliverable using `edit_format_range`. All styling is layered: only
the fields you pass change, so build up in passes — base format first, then
headers, then totals, then accents. A later pass never resets what an earlier
pass set unless it names the same field.

There are no MCP tools for column widths, row heights, merged cells, or
conditional formatting — design within those limits (use `wrap_text: true`
for long headers instead of widening columns; lead with a title row instead
of merging cells).

## Color conventions (financial-model standard)

| Element | Treatment |
|---|---|
| Hardcoded inputs / assumptions | Blue font `#0000C8` |
| Formulas / calculations | Black font (default — leave alone) |
| Cross-sheet links | Green font `#007A33` |
| Notes / footnotes | Gray italic `#808080` |
| Header row | Bold white text on dark navy fill `#1F3864` (or bold black on light gray `#F2F2F2` for a lighter look) |
| Section sub-headers | Bold, light fill `#DDEBF7`, no border |
| Warnings / flags | Red font `#C00000` |

Stick to one accent family per sheet. Dark-fill headers read as "report";
light-gray headers read as "working model".

## Number formats

Multi-section codes (`positive;negative;zero`) and `[Red]`/`[Blue]` section
colors are supported. The formats that make a model read as professional:

| Use | Code |
|---|---|
| Currency, whole | `$#,##0;($#,##0)` |
| Currency, cents | `$#,##0.00;($#,##0.00)` |
| Plain thousands | `#,##0;(#,##0)` |
| Negative-in-red | `#,##0;[Red](#,##0)` |
| Percent | `0.0%` |
| Multiple (comps/valuation) | `0.0"x"` |
| Thousands-scaled (000s) | `#,##0,` |
| Millions-scaled | `$#,##0,,"M"` |
| Date | `yyyy-mm-dd` or `mmm-yy` for period headers |
| Zero shown as dash | `#,##0;(#,##0);"–"` |

Always use parentheses (not minus signs) for negatives in financial output,
and apply one consistent decimal precision per column — mixed precision in a
column is the most common "amateur" tell.

## Alignment and typography

- Headers: `align: "center"`, `bold: true`, `wrap_text: true` if long.
- Numbers: right-aligned (the default general alignment already does this for
  numerics — don't force `align: "right"` onto text columns).
- Labels / first column: `align: "left"`.
- Title row: `font_size: 14`, `bold: true`; subtitle `font_size: 10`,
  `font_color: "#808080"`.
- Body text stays at the default size — only titles and headers deviate.

## Borders

Less is more: never box every cell (`border: "all"` reads as a gridline mess).
The professional pattern:

- Header row: `border: "bottom"`, `border_style: "medium"`.
- Subtotal rows: `border: "top"`, `border_style: "thin"`.
- Grand-total row: `border: "top_bottom"` with `border_style: "thin"` top and
  a second pass `border: "thick_bottom"` — or simply
  `border: "top"` + `bold: true`, which is the cleaner modern look.
- Table boundary (optional, report-style): `border: "outer"`,
  `border_style: "thin"`, `border_color: "#BFBFBF"`.

## Worked example

A P&L-style sheet with header in row 1, data in rows 2–13, total in row 14:

```
edit_format_range range=A1:F1  bold=true font_color=#FFFFFF fill_color=#1F3864
                  align=center border=bottom border_style=medium
edit_format_range range=B2:F14 number_format=$#,##0;($#,##0)
edit_format_range range=A2:A14 align=left
edit_format_range range=B2:D13 font_color=#0000C8        # hardcoded inputs only
edit_format_range range=A14:F14 bold=true border=top border_style=thin
edit_format_range range=A16     italic=true font_size=9 font_color=#808080  # footnote
```

Order matters only where passes touch the same field — apply the broad
number-format pass before narrow font-color overlays so nothing is skipped,
and keep input-blue passes scoped to actual hardcoded cells (check with
`data_get_cell` if unsure whether a cell holds a formula).

## Checklist before calling a sheet "done"

1. One number format per column; negatives in parentheses; consistent decimals.
2. Inputs blue, formulas black — spot-check a few cells with `data_get_cell`.
3. Header row formatted and bottom-bordered; totals bolded with a top border.
4. No `border: "all"` anywhere; no more than two fill colors on the sheet.
5. Footnotes/sources present in gray italic below the table.
6. Formatting cannot be visually verified over MCP (`report_snapshot` captures
   data and plots, not styles) — state what was applied rather than claiming
   it "looks right".
