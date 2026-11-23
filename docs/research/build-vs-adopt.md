# Build or adopt: can existing tools already do this?

Research for issue #2 (part of #1). Researched 2026-11-22 against each tool's own
repository source and README at its default branch, and its npm registry entry.

## The bar

From `HANDOFF.md`, the target output is:

```
Opus | ctx 100.0k/1.0M (10%) | $1.50 | org/repo:branch
```

The parts that are hard for other tools:

1. **Context used** = `current_usage.input_tokens + cache_creation_input_tokens + cache_read_input_tokens`,
   falling back to `total_input_tokens`. It is rendered as `used/limit (pct%)` with one
   decimal place (`100.0k`, `1.0M`).
2. **Only the used number** is colored, by **absolute token count**: green under 250k,
   yellow over 250k, red over 400k. The limit and percentage stay uncolored.
3. **Worktree** = the last two path components of `git rev-parse --show-toplevel`, cyan.
4. **Detached HEAD** shows the short sha (`@abc1234`).
5. The git segment and its separator disappear outside a repo; cost shows `n/a` when missing.

The second goal is Starship-style customization: a format string of `$module`
variables, plus per-module `format`, `style`, `symbol` and `disabled`.

## Summary

| Tool | Reproduces the exact layout? | Format string | Conditional colors | Runtime | License / health |
|---|---|---|---|---|---|
| ccstatusline | No. Close with widget merging, but no threshold color, root dir is one component, detached shows `(sha)` | No. Ordered widget list per line | No (hide rules only) | Node, 3.1 MB bundle with an Ink/React TUI | MIT, 13.7k stars, v2.2.34 (2026-11-06), 140 open issues |
| @owloops/claude-powerline | No. Fixed segment text, different context format, `detached` text | No. Segment toggles with options | Fixed: whole segment by % left (40/20) | Node >= 18 (`npx`, or `bunx --bun`) | MIT, 1.2k stars, v1.33.0 (2026-11-22), 2 open issues |
| ccusage `statusline` | No. Fixed line, no git segment | No (templates listed as "coming soon") | Context by % (50/80, configurable) | Node wrapper that spawns a native Rust binary; also scans usage logs | MIT, 19.5k stars, v20.0.51 (2026-11-14), statusline is "Beta" |
| CCometixLine (`ccline`) v2 | No, but closest. Has `@sha`, `{tokens}`/`{window}` fields, `separator = " \| "`. Misses one-decimal format, absolute thresholds, per-field color, git-toplevel path | Yes, per-item `{field}` templates with optional `[ ... ]` groups | Yes, `levels` by % of the window, but only for bar/gauge fill | Native Rust binary via npm | MIT, 3.6k stars, v2.0.1 (2026-10-24), 29 open issues |
| cc-statusline (chongdashu) | No. Generates a bash script from a few questions | No | No | Node to generate, then bash + jq at runtime | MIT, 646 stars, last push 2026-02-16 |
| claude-hud | No. A different product (HUD panel, tools/agents/todos) | No | n/a | Node >= 22.18, Claude Code plugin | 29.3k stars, v0.15.0 (2026-11-02) |

**Recommendation: build.** Use `reference/prototype.ts` as the starting point. Borrow two
design ideas: CCometixLine's template syntax (`[ ... ]` optional groups) and threshold
`levels`, and ccstatusline's careful `context_window` parsing. Details are at the end.

---

## ccstatusline (the user's previous status line)

Sources: <https://github.com/sirmalloc/ccstatusline>, <https://www.npmjs.com/package/ccstatusline>

- **Health:** MIT, about 13.7k stars, latest release v2.2.34 on 2026-11-06, about 140 open
  issues, pushed 2026-11-16 (`gh repo view sirmalloc/ccstatusline`;
  [releases](https://github.com/sirmalloc/ccstatusline/releases)). Very active.
- **Config model:** a JSON file at `~/.config/ccstatusline/settings.json`
  ([README](https://github.com/sirmalloc/ccstatusline/blob/main/README.md)), normally
  edited through an interactive TUI. The schema is `lines: WidgetItem[][]` (up to 3 lines)
  plus global options: `defaultSeparator`, `defaultPadding`, `colorLevel`, `powerline`,
  `globalBold`, `numberFormat`, `gitCacheTtlSeconds`
  ([src/types/Settings.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/types/Settings.ts)).
- **Per-widget options:** `color`, `backgroundColor`, `bold`, `dim`, `numberFormat`,
  `rawValue` (drops the label), `customText`, `customSymbol`, `commandPath`, `maxWidth`,
  `merge` (`true` or `'no-padding'`), and free-form `metadata`
  ([src/types/Widget.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/types/Widget.ts)).
  There is **no format string** and **no conditional coloring**. Colors are static per
  widget. "Conditional" support means *hiding* a widget (no git, zero value), not
  recoloring it
  ([README, "Unified conditional hiding"](https://github.com/sirmalloc/ccstatusline/blob/main/README.md)).
- **Widget breadth:** about 100 widgets (model, context length/window/percentage/bar,
  session cost, many git and jj widgets, custom text, custom command, and more)
  ([src/widgets/](https://github.com/sirmalloc/ccstatusline/tree/main/src/widgets)).
- **Can it reproduce the layout?** Partly, by composing merged widgets:
  `Model | ContextLength(raw) "/" ContextWindow(raw) " (" ContextPercentage ")" | SessionCost | GitRootDir ":" GitBranch`.
  - Context used: **matches**. It sums the `current_usage` fields and falls back
    sensibly ([src/utils/context-window.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/utils/context-window.ts)).
    Token formatting gives `512.0k` and `1.0M` by default
    ([src/utils/format-tokens.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/utils/format-tokens.ts)).
  - Threshold color on the used number: **not possible**. `ContextLength` has one static
    color ([src/widgets/ContextLength.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/widgets/ContextLength.ts)).
  - Two-component worktree: **not possible**. `GitRootDir` keeps only the last path part
    ([src/widgets/GitRootDir.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/widgets/GitRootDir.ts)).
    `GitWorktree` shows only the linked worktree's name
    ([src/widgets/GitWorktree.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/widgets/GitWorktree.ts)).
  - Detached HEAD: renders `(sha)`, not `@sha`
    ([src/widgets/GitBranch.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/widgets/GitBranch.ts)).
  - Hiding the git segment with its separator: supported. Manual separators collapse
    around hidden widgets ([README changelog](https://github.com/sirmalloc/ccstatusline/blob/main/README.md)).
  - Workaround for the gaps: a `Custom Command` widget that runs our own script. That
    spawns an extra process on every refresh, which defeats the purpose.
- **Runtime:** Node (`engines.node >= 14`). One bundled `dist/ccstatusline.js`, 3.1 MB
  unpacked on npm. The Ink TUI is lazy-loaded only when stdin is a TTY
  ([src/ccstatusline.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/ccstatusline.ts)).
  Git runs through `execFileSync` with `--no-optional-locks` and a persistent per-repo
  cache in `~/.cache/ccstatusline`
  ([src/utils/git.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/utils/git.ts)).
- **Install story:** `npx -y ccstatusline@latest` / `bunx`, or a "pinned global install"
  that the TUI performs and writes into `settings.json`
  ([README](https://github.com/sirmalloc/ccstatusline/blob/main/README.md)).

## @owloops/claude-powerline

Sources: <https://github.com/Owloops/claude-powerline>, <https://www.npmjs.com/package/@owloops/claude-powerline>

- **Health:** MIT, about 1.2k stars, v1.33.0 released 2026-11-22, 2 open issues
  ([releases](https://github.com/Owloops/claude-powerline/releases)). Active.
- **Config model:** JSON at `~/.claude/claude-powerline.json`, plus CLI flags (`--style`,
  `--theme`, `--charset`) and env vars. There is a web configurator at
  powerline.owloops.com. Layout is `display.lines[].segments.{name: {enabled, ...options}}`
  ([README](https://github.com/Owloops/claude-powerline/blob/main/README.md)).
  Colors come from themes, or from `colors.custom.<segment> = {bg, fg, bold}`. The `tui`
  style has a CSS-grid layout engine with `{segment.part}` placeholders. **No format string**
  for the normal styles.
- **Can it reproduce the layout?** No.
  - Context text is `◔ 34,040 (79%)`: grouped digits, no limit shown, and percent
    *remaining* by default. It uses an `autocompactBuffer` of 33k to compute "usable"
    percent ([README, Context](https://github.com/Owloops/claude-powerline/blob/main/README.md)).
    It does sum `current_usage` the same way we do
    ([src/segments/context.ts](https://github.com/Owloops/claude-powerline/blob/main/src/segments/context.ts)).
  - Thresholds are hardcoded: warning at <= 40% left, critical at <= 20% left. They
    recolor the **whole segment**
    ([src/segments/renderer.ts](https://github.com/Owloops/claude-powerline/blob/main/src/segments/renderer.ts)).
  - Directory style is `full | fish | basename`, with no "last two"
    ([README](https://github.com/Owloops/claude-powerline/blob/main/README.md)).
  - Detached HEAD shows the literal `detached`
    ([src/segments/git.ts](https://github.com/Owloops/claude-powerline/blob/main/src/segments/git.ts)).
- **Runtime:** Node >= 18, `bin` is a `#!/usr/bin/env node` shim into `dist/index.mjs`,
  636 KB unpacked, no runtime dependencies
  ([package.json](https://github.com/Owloops/claude-powerline/blob/main/package.json)).
  The README recommends `npx -y @owloops/claude-powerline@latest` in `statusLine`, which
  is exactly the per-refresh registry hit HANDOFF.md warns against. `bunx --bun` is
  offered as faster. The git segment can run several git commands (about 14 call sites in
  `src/segments/git.ts`).

## ccusage `statusline`

Sources: <https://github.com/ccusage/ccusage>, [docs/guide/statusline.md](https://github.com/ccusage/ccusage/blob/main/docs/guide/statusline.md)

- **Health:** MIT, about 19.5k stars, v20.0.51 released 2026-11-14
  ([releases](https://github.com/ccusage/ccusage/releases)). The core is now Rust. The
  `statusline` command is marked **Beta**.
- **Output is fixed:**
  `🤖 Sonnet 4 (high) | 💰 $1.23 session / $0.06 today / $0.06 block (1h 0m left) | 🔥 $0.02/hr | 🧠 50,000 (25%)`.
  There is **no git/worktree segment**. Customization is limited to flags:
  `--context-low-threshold` / `--context-medium-threshold` (percent, default 50/80),
  `--visual-burn-rate`, `--cost-source`, and `--offline`. "Custom format templates" are
  listed under "coming soon" ([statusline.md](https://github.com/ccusage/ccusage/blob/main/docs/guide/statusline.md)).
- **Runtime:** the npm `ccusage` package is a Node wrapper (`src/cli.js`) that **spawns** a
  per-platform native binary (`@ccusage/ccusage-<platform>-<arch>`)
  ([apps/ccusage/src/cli.js](https://github.com/ccusage/ccusage/blob/main/apps/ccusage/src/cli.js),
  [apps/ccusage/package.json](https://github.com/ccusage/ccusage/blob/main/apps/ccusage/package.json)).
  It also aggregates local usage logs for today/block cost, which is more work than we need.
- **Verdict:** a usage reporter with a status line attached, not a configurable prompt.
  Not a candidate.

## CCometixLine (`@cometix/ccline`) v2

Sources: <https://github.com/Haleclipse/CCometixLine>, <https://www.npmjs.com/package/@cometix/ccline>

- **Health:** MIT (`Cargo.toml`), about 3.6k stars, v2.0.1 released 2026-10-24, 29 open
  issues ([releases](https://github.com/Haleclipse/CCometixLine/releases)). v2 is a rewrite
  with a new config format ([README](https://github.com/Haleclipse/CCometixLine/blob/master/README.md)).
- **Config model:** the closest to Starship. `~/.claude/ccline/config.toml` has
  `separator = " | "` and `[[line]]` with `left`/`right` item lists. Each item is a metric
  id or a table with:
  - `format` / `compact` / `minimal` templates, where `{field}` inserts a value and
    `[ ... ]` drops its contents when a field inside is empty
  - `icon`, `color` / `fg` / `icon_color` (palette token, name, hex, 256)
  - `levels` (for example `[[50, "warn"], [80, "crit"]]`) with `coloring`
  - metric `options` (for example `dir`'s `style`)
  - plus palettes and a TUI configurator
  ([README, Configuration](https://github.com/Haleclipse/CCometixLine/blob/master/README.md)).
- **Can it reproduce the layout?** Close, but no.
  - Detached HEAD: **matches** (`@` + 7-char oid)
    ([src/metrics/code.rs](https://github.com/Haleclipse/CCometixLine/blob/master/src/metrics/code.rs)).
  - Context template fields are `pct`, `tokens`, `window`, `left`, `cc_pct`, so
    `ctx {tokens}/{window} ({pct}%)` is expressible
    ([src/metrics/usage.rs](https://github.com/Haleclipse/CCometixLine/blob/master/src/metrics/usage.rs)).
    Three differences:
    - The token source **prefers `total_input_tokens`** over the `current_usage` sum, the
      reverse of our spec.
    - Formatting trims decimals: `100k` and `1M`, not `100.0k` and `1.0M`
      ([src/metrics/fmt.rs](https://github.com/Haleclipse/CCometixLine/blob/master/src/metrics/fmt.rs)).
    - It hides `tokens` when it is 0.
  - Thresholds: `levels` work on **% of the window**, not absolute tokens. Severity drives
    only the bar/gauge **fill** color, not the text of one field
    ([src/widgets/mod.rs](https://github.com/Haleclipse/CCometixLine/blob/master/src/widgets/mod.rs)).
    So "only the used number turns red over 400k" is not expressible.
  - `dir` with `style = "parent"` gives the last two components of the **current dir**,
    not the git top-level, so it is wrong when you are in a subdirectory
    ([src/metrics/code.rs](https://github.com/Haleclipse/CCometixLine/blob/master/src/metrics/code.rs)).
- **Runtime:** a native Rust binary installed through npm
  (`npm install -g @cometix/ccline`, which puts the binary at `~/.claude/ccline/ccline`),
  so startup is fast. Forking it means maintaining Rust, while the team wants Bun/TS.

## Others

- **cc-statusline** (<https://github.com/chongdashu/cc-statusline>): `npx @chongdashu/cc-statusline init`
  asks a few questions and **generates a bash script**. That script needs `jq` at runtime
  for context/tokens (per its README). No format string or thresholds. Last push
  2026-02-16. This is our current bash approach with less control.
- **claude-hud** (<https://github.com/jarrodwatts/claude-hud>): a Claude Code plugin
  (`/plugin install claude-hud`) that shows a HUD of context, active tools, agents and
  todos. It needs Node >= 22.18 and keeps settings in
  `~/.claude/plugins/claude-hud/config.json`. Popular (about 29k stars), but a different
  product with no format-string layout.

## Gap matrix against the spec

| Requirement | ccstatusline | claude-powerline | ccusage | CCometixLine v2 |
|---|---|---|---|---|
| `current_usage` sum as used | yes | yes | yes (via `context_window`) | no (prefers `total_input_tokens`) |
| `100.0k/1.0M (10%)` format | yes, via 5 merged widgets | no | no | no (`100k/1M`) |
| Color only the used number, by absolute 250k/400k | no | no | no | no (% based, fill only) |
| Last two parts of git top-level | no | no | no git | no (cwd based) |
| Detached HEAD `@sha` | no (`(sha)`) | no (`detached`) | no git | yes |
| Hide git segment and separator outside a repo | yes | yes | n/a | yes (`[ ]` groups / empty items) |
| Starship-like format string + per-module style/symbol/disabled | no | no | no | partial (per-item templates, not one `$module` string) |

## Recommendation: build

1. **No tool reproduces the agreed layout exactly.** Every candidate fails at least two
   hard requirements. All of them miss the most visible one: coloring only the used
   number by absolute thresholds. Adopting means renegotiating the spec the user agreed to.
2. **The Starship model is not available off the shelf.** ccstatusline and
   claude-powerline are widget/segment lists edited through a TUI or web UI. CCometixLine
   has per-item templates and thresholds but no single `$module` format string, and it is
   Rust.
3. **Forking costs more than building.** ccstatusline is a 3.1 MB bundle with about 100
   widgets, an Ink/React TUI, jj support, powerline rendering and update checks. We need
   4 modules. CCometixLine is Rust, which fails the Bun/TS distribution goal. A fork of
   either would carry far more code than the feature needs, plus constant upstream drift.
4. **Building is cheap and mostly done.** `reference/prototype.ts` already matches the bash
   output at about 19 ms per run with one git call (HANDOFF.md). What is left is
   structure, the format-string engine, config loading, `init`, and the detached-HEAD fix.
5. **Install story stays ours.** All the Node tools document `npx ...@latest` in
   `statusLine`, which HANDOFF.md rejects. We can ship a global bin plus a safe
   `init` instead.

**Borrow, don't fork:**

- CCometixLine's template rule, where `[ ... ]` drops when a field inside it is empty. It
  maps well onto Starship's `( ... )` conditional groups. So do its `levels`-style
  thresholds, but make them absolute and applicable to a single variable's style.
- ccstatusline's defensive `context_window` parsing: `current_usage` can be a number or an
  object, may be null before the first API call, and percentages need clamping
  ([src/utils/context-window.ts](https://github.com/sirmalloc/ccstatusline/blob/main/src/utils/context-window.ts)).
- claude-powerline's note that `current_usage` is null before the first API call and
  again after compaction
  ([src/segments/context.ts](https://github.com/Owloops/claude-powerline/blob/main/src/segments/context.ts)).

**When adopting would make sense:** if the user is willing to drop absolute-threshold
coloring and the two-component path, ccstatusline (which they already used) gets about 80%
of the way there with zero code. That trade is the user's call, but it gives up both the
spec and the Starship-style config goal.
