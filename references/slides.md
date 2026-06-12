# Slide decks from workbook data

A slide deck in TukeySheets is just a Paper whose Typst source uses a
presentation package. The compiler resolves `@preview` packages on demand
(downloaded once, cached to disk — the first compile needs network access and
is noticeably slower). `typslides` is the proven choice in this app.

All live-data embeds work inside slides exactly as in report papers:
`#plot("Sheet!N")` (vector, re-renders on every compile), `#sheet("Sheet!A1:D8")`
(range as table), `#cell("Sheet!B14")` (one formatted value inline). That means
a deck rebuilt after the data changes is automatically current — prefer embeds
over typing numbers into the source.

## Workflow

1. **Gather the story first.** `workspace_list_sheets`, then profile/stat
   tools to find the headline numbers. Decide the 3–6 findings worth a slide
   each before writing any Typst.
2. **Build the plots as sheet plots** (`plot_create`), not as Typst figures.
   Note each plot's `number` from `plot_list` — that's the `N` in
   `#plot("Sheet!N")`. Consider deck-dedicated copies of existing plots with
   larger fonts and short titles: a plot sized for a sheet panel is usually
   too dense for a projector.
3. **`paper_create`** with the deck skeleton (below), in a `folder` like
   `Decks` if the workbook has many tabs.
4. **Iterate with `paper_render`** (format `png`, one `page` at a time). It
   compiles fresh and returns `diagnostics[]` with byte offsets into your
   source — this is the verification loop. If the pinned package version
   doesn't exist, the diagnostic says so; adjust the version rather than
   abandoning the package.
5. **Check page count == slide count** (`paper_list`). typslides does not
   clip an overfull slide — overflow flows onto an extra page, so a surplus
   page means some slide needs trimming.
6. **`paper_export`** to PDF. Remember recompiles after `paper_set_source`
   are async (~10–15 s) — `paper_export` called immediately serves the
   previous revision. A `paper_render` round-trip first is the reliable way
   to know the latest source compiled.

## Skeleton

```typ
#import "@preview/typslides:1.3.3": *

#show: typslides.with(ratio: "16-9", theme: "bluey")

#front-slide(
  title: "Q2 Revenue Review",
  subtitle: [Data refreshed from the live workbook],
  authors: "Analytics",
)

#slide(title: [Revenue finished at #cell("Summary!B14"), up 12% QoQ])[
  #plot("Revenue!1")
]

#slide(title: "Where the growth came from")[
  #sheet("Summary!A1:D8")
  #place(bottom + left, text(size: 9pt, fill: gray)[Source: CRM export, Jun 2026])
]

#focus-slide[Recommendation: expand the EU pilot]
```

## Slide-craft rules

- **Assertion–evidence titles.** The slide title states the finding
  ("Churn is concentrated in month 2"), the body shows one plot or one small
  table that proves it. One idea per slide.
- **`#cell` for every headline number** so re-renders stay truthful.
- **Keep `#sheet` ranges small** — roughly 8 rows fits a slide; more
  overflows onto a phantom page. For bigger tables, aggregate first with
  `stat_run_sql`/`transform_*` into a small summary sheet and embed that.
- **One plot per slide**, full width. Two plots compete; if a comparison is
  the point, build a single faceted plot instead.
- **Footnotes/sources**: `#place(bottom + left, ...)` is the reliable way to
  pin text to the slide bottom.
- **Embeds need content, not strings.** `#cell(...)`/`#plot(...)` only
  evaluate inside markup content (`[...]`) — a `#` inside a string literal is
  literal text. Pass slide titles as content blocks when they carry a live
  number.
- **`$` in strings**: `\$` is only an escape inside markup content
  (`[...]`); inside string arguments (`"..."`) it renders a literal
  backslash — write plain `$` there.
- Match the deck theme to the plot theme (e.g. dark deck theme with the plot
  DSL's `theme = "dark"`) so embedded charts don't look pasted-in.
- Final visual pass: `paper_render` each page as PNG and actually look at
  it — text overflow and squashed plots only show up visually.
