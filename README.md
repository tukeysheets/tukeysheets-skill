# TukeySheets Skill

A [Claude Code](https://claude.com/claude-code) skill for driving a running
[TukeySheets](https://tukeysheets.com) desktop app through its MCP tools —
read and edit sheets, write formulas, transform data, run statistics and SQL,
build plots, and author Typst reports and slide decks inside the live
spreadsheet app.

## Requirements

- A running TukeySheets desktop app serving MCP at
  `http://127.0.0.1:31337/mcp` (its tools appear as `mcp__tukeysheets__*`)
- Network access on first slide-deck compile (to download Typst packages)

## Installation

The skill directory name must match the skill name (`tukeysheets`), so clone
the repo to that path.

**Personal (available in all projects):**

```sh
git clone https://github.com/tukeysheets/tukeysheets-skill ~/.claude/skills/tukeysheets
```

**Per-project:**

```sh
git clone https://github.com/tukeysheets/tukeysheets-skill .claude/skills/tukeysheets
```

Then launch TukeySheets, start Claude Code, and ask it to work with your
sheets — the skill activates automatically whenever a task involves
TukeySheets or a `.tukey` workbook.

## Contents

- [`SKILL.md`](SKILL.md) — core workflow rules and conventions
- [`references/tools.md`](references/tools.md) — full MCP tool catalog
- [`references/formatting.md`](references/formatting.md) — professional formatting conventions
- [`references/slides.md`](references/slides.md) — building Typst slide decks
