# Starship's config model in detail

Research for issue #3 (part of #1). Checked on 2026-11-22 against Starship **v1.27.0**
(the latest release, 2026-10-17) and `main` at commit
[`152cb69`](https://github.com/starship/starship/tree/152cb69dffc0260d260ea617a7ec763e7314d55c).

Source links below are permalinks to that commit. `SRC` stands for
`https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c`.

---

## 0. Headline finding: Starship already ships a Claude Code status line

This changes the framing of the project. PR
[#7234](https://github.com/starship/starship/pull/7234) was merged on 2026-04-03 and released in
v1.25.x. Since then Starship has had a `starship statusline claude-code` subcommand (alias `claude`).
It reads the Claude Code JSON payload from stdin and renders a **profile** named `claude-code`.

- Docs: <https://starship.rs/advanced-config/#statusline-for-claude-code>
  ([repo copy](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/docs/advanced-config/README.md))
- CLI: [`src/main.rs` `Statuslines::ClaudeCode`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/main.rs) maps to `Target::Profile("claude-code")`.
- Entry point: [`print::prompt_with_claude_code`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs). It deserializes stdin. On a parse
  error it logs and falls back to `Default`. It then forces `Shell::Unknown`, so no shell escaping
  happens, and calls the normal `get_prompt`.
- Default profile: `claude-code = "$claude_model$git_branch$claude_context$claude_cost"`
  ([`starship_root.rs` `default_profiles`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/configs/starship_root.rs)).
- It has three modules:
  - `claude_model`: variables `model`, `model_id`, `effort`, plus a `model_aliases` map.
  - `claude_context`: variables `gauge`, `percentage`, `input_tokens`, `output_tokens`,
    `curr_*_tokens`, `total_tokens`. A `display` threshold array picks `style` or hides the module.
  - `claude_cost`: variables `cost`, `duration`, `api_duration`, `lines_added`, `lines_removed`,
    with the same kind of threshold `display` array.

  Sources: [configs](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/configs/claude_context.rs), [modules](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/modules/claude_model.rs),
  [docs](https://starship.rs/advanced-config/#claude-context).

```json
{ "statusLine": { "type": "command", "command": "starship statusline claude-code" } }
```

```toml
[profiles]
claude-code = '$claude_model$claude_context$claude_cost$git_branch'

[claude_model]
format = '[$model]($style) | '
style = 'purple'

[[claude_context.display]]   # threshold -> style; the highest matching threshold wins
threshold = 0
style = 'green'
[[claude_context.display]]
threshold = 60
style = 'yellow'
```

Our planned output in HANDOFF.md overlaps with this, but not exactly:

- Starship colors the context segment by **percentage** and shows a gauge. Our spec colors by
  absolute token count.
- It has no "worktree = last two path components" segment.
- Its cost segment has no `n/a` fallback. A missing value hides the module.

Planning (#6/#7) should decide between building our own package, shipping a Starship config, or doing both.

---

## 1. Config location, override, and load failures

- The default path is `~/.config/starship.toml`. The `STARSHIP_CONFIG` env var replaces that path
  entirely ([docs](https://starship.rs/config/#config-file-location);
  [`get_config_path_os`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/context/mod.rs)).
- The format is TOML and every key is optional. Missing keys take struct defaults through
  `#[serde(default)]` ([`StarshipRootConfig`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/configs/starship_root.rs)).
- Warnings and errors go to a log file, never to the prompt. The file is
  `~/.cache/starship/session_${STARSHIP_SESSION_KEY}.log`, and `STARSHIP_CACHE` overrides the
  directory ([docs](https://starship.rs/config/#logging)).
- A JSON schema exists for editor validation: `"$schema" = 'https://starship.rs/config-schema.json'`
  ([`starship_root.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/configs/starship_root.rs)).

### Invalid config is never fatal

Each kind of failure falls back at its own level:

| Failure | Behavior | Source |
|---|---|---|
| File missing | Debug log, all defaults | [`read_config_content_as_str`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/config.rs) |
| File unreadable, or TOML syntax error | `log::error!`, the **whole config is ignored**, all defaults | [`config_from_file`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/config.rs) |
| Unknown key in a module table | `log::warn!`, then deserializes again ignoring unknown keys; the rest of the table is kept | [`ModuleConfig::from_config`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/config.rs) |
| Wrong type for a key (e.g. `disabled = "yes"`) | `log::warn!("Failed to load config value")`, and **that module's whole config falls back to defaults** | [`ModuleConfig::load`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/config.rs) |
| Top-level `format` fails to parse | `log::error!`, the prompt becomes a literal `>` | [`load_formatter_and_modules`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs) |
| Profile name not found | `log::error!`, the prompt becomes `>` | same |
| A module's `format` fails to parse | `log::warn!`, the module returns `None` and disappears | e.g. [`claude_model.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/modules/claude_model.rs) |
| Invalid style string at render time | `parse_style_string` returns `None`, the text renders **unstyled** (debug log only) | [`parse_style_string`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/config.rs) |
| Invalid style in a `Style`-typed field | serde error "Invalid style string", handled like a wrong type above | [`deserialize_style`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/config.rs) |
| Unknown palette name | `log::warn!`, no palette is used | [`get_palette`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/config.rs) |
| Unknown `$name` in the top-level format | Debug log, renders nothing | [`handle_module`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs) |
| `TERM=dumb` | Prints `Starship disabled due to TERM=dumb > ` | [`get_prompt`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs) |

---

## 2. Format-string grammar (the pest spec)

The whole language is about 50 lines of PEG in [`src/formatter/spec.pest`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/spec.pest).
It parses into four node types: `Text`, `Variable`, `TextGroup{format, style}` and
`Conditional(format)` ([`model.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/model.rs),
[`parser.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/parser.rs)).

```pest
expression     = _{ SOI ~ value* ~ EOI }
value          = _{ text | variable | textgroup | conditional }
variable       = { "$" ~ (variable_name | variable_scope) }
variable_name  = @{ ('a'..'z' | 'A'..'Z' | "_") ~ char* }   // char = [A-Za-z0-9_]
variable_scope = _{ "{" ~ variable_scoped_name ~ "}" }     // ${custom.foo}
text           = { (string | escape)+ }
escape         = _{ "\\" ~ escaped_char }
escaped_char   = { "[" | "]" | "(" | ")" | "\\" | "$" }
textgroup      = { "[" ~ format ~ "]" ~ "(" ~ style ~ ")" }
format         = { value* }
style          = { (variable | string)* }
conditional    = { "(" ~ format ~ ")" }
```

### `$variable`

- `$name` matches `[A-Za-z_][A-Za-z0-9_]*` ([docs](https://starship.rs/config/#variable)).
- `${...}` accepts any characters except braces and the escapable characters. Use it for dotted
  names such as `${custom.foo}` and `${env_var.HOME}`
  ([spec.pest](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/spec.pest); [docs, custom](https://starship.rs/config/#custom-commands)).
- In the **top-level** format, each variable names a module. In a **module's** format, the variables
  are the ones that module exposes, covered in section 5
  ([docs, terminology](https://starship.rs/config/#terminology)).
- If no mapper claims a variable, it stays `None` and renders as nothing. This is not an error
  ([`StringFormatter::map` doc comment](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/string_formatter.rs)).

### `[text](style)` text groups

- The bracket part is a full nested format string, so groups can nest. The parenthesis part is a
  style string, which may contain `$variables` such as `($style)`
  ([docs](https://starship.rs/config/#text-group)).
- Example: `'[a [b](red) c](green)'` renders `a` and `c` in green and `b` in red.
- An inner style **replaces** the outer one rather than merging with it. A styled variable segment
  with no style of its own inherits the enclosing group's style through `set_style_if_empty`
  ([`string_formatter.rs` `parse_format`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/string_formatter.rs)).
- An empty group `[](style)` still emits an empty segment, so `prev_fg`/`prev_bg` have something to
  refer to. Powerline separators rely on this
  ([`parse_textgroup`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/string_formatter.rs)).

### `( ... )` conditional groups: exactly when they render

- The docs say a conditional group "will not render if all variables inside are empty"
  ([docs](https://starship.rs/config/#conditional-format-strings)).
- The implementation is `should_show_elements`. It collects **every variable inside the group,
  recursively**, including those in nested text groups and conditionals. The group renders if **any**
  of them is present ([source](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/string_formatter.rs)). "Present" means:
  - `Plain` / `NoEscapingPlain`: a non-empty string.
  - `Styled` (a module's segments, in the top-level format): at least one segment with non-empty text.
  - `Meta`, a variable whose value is itself a format string, such as `$symbol`: Starship checks the
    variables inside that format string, recursively.
  - `None` (unmapped or absent) or `Err`: not present.
- What this means in practice:
  - `(some text)` contains no variables, so it **never** renders.
  - `(@$region)` renders only when `region` is non-empty.
  - A typical `$symbol` value is plain text with no variables. On its own it **never makes a
    conditional render**. That is why the defaults put the symbol outside the conditional, as in
    `'[$symbol($version )]($style)'`.

```toml
[aws]
format = 'on [$symbol($profile )(\($region\) )]($style)'
```

### Escaping

- `$ [ ] ( ) \` must be escaped with a backslash to appear literally
  ([docs, strings](https://starship.rs/config/#strings); `escaped_char` in spec.pest).
- TOML adds its own escaping layer. A basic `"..."` string needs `\\$`, while a literal `'...'`
  string takes `\$` as written. `format = '\[\$\] '` and `format = "\\[\\$\\] "` are equivalent, so
  **use single-quoted TOML** ([docs](https://starship.rs/config/#strings)).
- A style string cannot contain `)` or any other escapable character: `style = (variable | string)*`,
  and `string` excludes `escaped_char` ([spec.pest](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/spec.pest)).
- Output escaping: variable values and literal text go through `shell_prompt_escape`, which escapes
  `\`, `$` and backticks for bash and `%` for zsh. `map_no_escaping` lets a value skip it. In
  statusline mode the shell is `Unknown`, so nothing is escaped
  ([`shell_prompt_escape`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/string_formatter.rs);
  [`prompt_with_claude_code`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs)).
- Newlines are ordinary text, written with `'''` multi-line TOML or `\n`. A trailing `\` inside a
  `"""` string joins lines ([docs](https://starship.rs/config/#strings)).

---

## 3. Style-string grammar

Spec: <https://starship.rs/advanced-config/#style-strings>. Implementation:
[`parse_style_string` / `parse_color_string`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/config.rs).

- Tokens are **separated by whitespace** and **case-insensitive**. Their order does not matter, except
  that the **last** fg or bg color wins.
- Attributes: `bold`, `italic`, `underline`, `dimmed`, `inverted`, `blink`, `hidden`,
  `strikethrough`. The token is `dimmed`, not `dim`. `inverted` swaps fg and bg (ANSI reverse).
- A color is written as `fg:<color>`, `bg:<color>`, or a bare `<color>`, which sets the foreground.
- `<color>` can be any of:
  - `black red green yellow blue purple cyan white`, optionally prefixed with `bright-`.
    `bright-black` is dark gray, and the name is `purple`, not `magenta`.
  - `0`–`255`: an 8-bit ANSI color (`Color::Fixed`).
  - `#RRGGBB`: must be exactly 7 characters, otherwise it is invalid.
  - A name defined in the active palette (section 6).
  - `prev_fg` / `prev_bg`: the previous segment's fg or bg color. `fg:prev_bg bg:prev_fg` is the
    usual powerline pattern.
- `none`:
  - A bare `none` or `fg:none` **voids the whole style**. `parse_style_string` returns `None` and the
    text is unstyled, whatever else the string contains.
  - `bg:none`, or `bg:` with any invalid color, only resets the background.
- Any other unrecognized token makes the **whole** string invalid, so the text renders unstyled.
- `''` means explicitly no styling.

Sources: [docs](https://starship.rs/config/#style-strings); the tests
`table_get_styles_plain_and_broken_styles` and `table_get_styles_with_none` in
[config.rs](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/config.rs).

```toml
style = 'bold fg:27 bg:#1e1e2e'
style = 'bright-yellow italic'
format = '[](fg:prev_bg bg:blue)'    # powerline separator
```

---

## 4. Top level: `format`, `$all`, `$fill`, `right_format`, profiles

Root options ([docs](https://starship.rs/config/#prompt);
[`StarshipRootConfig`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/configs/starship_root.rs)):

| Option | Default |
|---|---|
| `format` | `'$all'` |
| `right_format` | `''` |
| `continuation_prompt` | `'[∙](bright-black) '` |
| `scan_timeout` | 30 (ms) |
| `command_timeout` | 500 (ms) |
| `add_newline` | `true` |
| `follow_symlinks` | `true` |
| `palette`, `palettes` | none |
| `profiles` | none (the internal `claude-code` profile is built in) |

- **`$all`** expands to every module in the hard-coded `PROMPT_ORDER` that is **not referenced
  explicitly** in `format` or `right_format`. So `format = '$directory$all'` puts `directory` first
  and keeps everything else in default order
  ([`all_modules_uniq`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs); [docs, right prompt](https://starship.rs/advanced-config/#enable-right-prompt)).
  The `claude_*` modules are not in `PROMPT_ORDER` ([starship_root.rs](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/configs/starship_root.rs)).
- **`$custom`** expands to every `[custom.*]` module that isn't referenced explicitly, in definition
  order, skipping any with `disabled = true` ([`handle_module`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs)).
- **`$fill`** is a module that emits a special fill segment
  ([docs](https://starship.rs/config/#fill); [`painted.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/painted.rs)).
  - At paint time, each fill on a line gets an equal share of the columns left over. Leftover columns
    go to the first fills.
  - Without a known width, or without room, a fill keeps its natural width.
  - The width comes from `--terminal-width`, then `terminal_size()`, and defaults to **80**
    ([`default_width`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/context/mod.rs)).
- **`right_format`** is a second format string for shells that support a right prompt. Newlines in it
  are stripped ([docs](https://starship.rs/advanced-config/#enable-right-prompt);
  [`get_prompt`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs)).
- **`[profiles]`** are named alternative top-level formats, chosen by target.
  - The statusline uses this mechanism.
  - A user profile overrides the internal `claude-code` one.
  - A profile has no `right_format`.

  Source: [`load_formatter_and_modules`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs).
- `disabled = true` on a module removes it even when `format` references it
  ([`is_module_disabled_in_config`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/context/mod.rs)).
- Source detail: `add_newline` applies to every target except Continuation. That includes profiles,
  so it also affects the Claude statusline ([`get_prompt`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs)).

```toml
format = '$directory$git_branch$fill$cmd_duration$line_break$character'
right_format = '$time'
add_newline = false
```

---

## 5. Modules: config, variables, and disappearing

Every module follows the same pattern
([`claude_model.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/modules/claude_model.rs), [`fill.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/modules/fill.rs)):

1. **Load config.** The module calls `context.new_module("name")`, then
   `XConfig::try_load(module.config)`. The config struct uses `#[serde(default)]` and a hand-written
   `Default` with fields such as `format`, `symbol`, `style` and `disabled`
   ([`claude_model` config](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/configs/claude_model.rs)).
2. **Bail out when there is nothing to show.** It returns `None` when `config.disabled` is set. It
   also returns `None` when data is missing: no Claude payload (`context.claude_code_data.as_ref()?`),
   no git repo, no matching files.
   - **Returning `None` is how a module disappears.**
   - The top-level formatter then sees an empty `Styled` value, so any surrounding `(...)` collapses.
   - Signature: `pub fn module<'a>(context: &'a Context) -> Option<Module<'a>>`.
3. **Map variables.** It builds a `StringFormatter` from `config.format` and maps variables in layers
   ([`string_formatter.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/formatter/string_formatter.rs)):
   - `map_meta`: variables whose value is **another format string**, almost always `symbol`. That
     lets `symbol = '[](bold red) '` carry its own style.
   - `map_style`: variables that only work **inside a style** `( )`. By convention this is `style`.
     Some modules add per-variable styles, such as `added_style` and `deleted_style` in git_metrics
     ([git_metrics config](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/configs/git_metrics.rs)).
   - `map` / `map_no_escaping`: data variables such as `model` or `version`. Returning `None` leaves
     the variable empty.
4. **Render.** `.parse(default_style, ctx)` produces the segments. If parsing fails, the module logs
   a warning and returns `None`.

The docs list each module's **Options** and **Variables**. The variables table says `symbol` "mirrors
the value of option `symbol`" and that `style*` can only be used inside a style string
([docs, claude_model](https://starship.rs/advanced-config/#claude-model)).

Per-variable styles example:

```toml
[git_metrics]
added_style = 'bold green'
deleted_style = 'bold red'
format = '([+$added]($added_style) )([-$deleted]($deleted_style) )'
```

Threshold-driven styles, such as `[[battery.display]]` and `[[claude_context.display]]`, are a
convention individual modules follow, not part of the core language. Each threshold entry sets
`$style` or hides the module ([claude_context config](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/configs/claude_context.rs)).

### `[custom.*]` modules

Custom modules are user-defined and run a shell command
([docs](https://starship.rs/config/#custom-commands); [`custom.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/modules/custom.rs)).

- Display conditions: `when` (a bool, or a shell command that shows the module on exit 0),
  `detect_files` / `detect_folders` / `detect_extensions`, `require_repo`, `os`.
- Execution: `command`, `shell`, `use_stdin`, `ignore_timeout`, `unsafe_no_escape`.
- Display: `symbol`, `style`, `format` (default `'[$symbol($output )]($style)'`), `disabled`.
- The command's output is exposed as `$output`. Place a module in the format with `${custom.name}`.

```toml
format = '$directory${custom.k8s}$character'

[custom.k8s]
command = 'kubectl config current-context'
when = 'command -v kubectl'
symbol = '☸ '
style = 'blue'
```

`env_var` works the same way: define `[env_var.FOO]` and reference it with `${env_var.FOO}`
([`handle_module`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs)).

---

## 6. Palettes and presets

**Palettes** ([docs](https://starship.rs/config/#prompt); [`parse_color_string`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/config.rs)):

- `palette = 'name'` selects `[palettes.name]`, a map from names to colors.
- Palette names are looked up **after** hex and number parsing but **before** the built-in color
  names, so a palette can redefine `blue`.
- Palette entries are parsed without a palette, so one entry cannot refer to another.

```toml
palette = 'mine'

[palettes.mine]
blue = '21'
mustard = '#af8700'
```

**Presets** ([starship.rs/presets](https://starship.rs/presets/)):

- A preset is just a complete TOML file bundled into the binary.
- It is written out with `starship preset <name> -o ~/.config/starship.toml`
  (the `Preset` subcommand in [`main.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/main.rs); [nerd-font preset](https://starship.rs/presets/nerd-font)).
- There is no merging or inheritance. Applying a preset replaces the config file wholesale.
- Bundled presets include nerd-font-symbols, no-nerd-font, bracketed-segments, plain-text-symbols,
  no-runtime-versions, pastel-powerline, tokyo-night, gruvbox-rainbow, catppuccin-powerline, jetpack
  and pure-preset ([preset docs dir](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/docs/presets)).

## 7. Timeouts

- **`command_timeout`** (default 500 ms) bounds every external command: `git`, tool `--version`
  calls, `custom.command`.
  - On timeout Starship logs a `warn!` and the command yields `None`, which usually makes the module
    disappear ([`utils/mod.rs` exec](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/utils/mod.rs); [docs](https://starship.rs/config/#prompt)).
  - `custom.*.ignore_timeout = true` opts a custom module out
    ([docs](https://starship.rs/config/#custom-commands)).
- **`scan_timeout`** (default 30 ms) bounds the directory listing used by `detect_*`. On timeout it
  warns and continues with partial results
  ([`DirContents::from_path_with_timeout`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/context/mod.rs)).
- Modules render in parallel using rayon ([`print.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/print.rs)).

---

## Implications for a Claude Code status line

### Build or adopt

`starship statusline claude-code` already exists, so settle this before #6/#7 commit to a design.
What our package would add:

- No install friction for coworkers: Bun/npm instead of a Rust binary plus TOML.
- The exact agreed output spec: token-count color thresholds, the `owner/repo:branch` worktree
  segment, and `n/a` for a missing cost.
- Simpler config.

A cheap alternative is to ship a `starship.toml` snippet that gets close.

### Worth copying (high value, low cost)

- **One top-level format string with `$segment` references**, plus per-segment
  `format` / `style` / `symbol` / `disabled`. This is the most recognizable part of Starship.
- **Text groups `[...](style)` and conditional groups `(...)`, with Starship's exact semantics.** A
  conditional renders if *any* variable inside is non-empty, and symbol-only content doesn't count.
  This rule replaces hand-coded "drop the separator when the git segment is missing" logic: put
  separators inside the conditional, e.g. `'$model( | $git)'`.
- **Segments disappear by returning null or empty.** Missing data hides a segment instead of raising
  an error.
- **A subset of the style grammar**: the 8 named colors and their `bright-` variants,
  `bold` / `dimmed` / `italic` / `underline`, `0`–`255`, `#rrggbb`, `bg:` / `fg:`.
  - Copy the "invalid style means unstyled, never crash" behavior.
  - Consider accepting `magenta` as an alias for `purple`, since our spec says magenta.
- **Per-variable style variables or threshold arrays.** Our context segment colors only the *used*
  number, by token thresholds. That fits `[$used]($used_style)/$limit ($pct%)`, with `used_style`
  chosen from a `[[context.display]]` list of thresholds.
- **Config loading that never fails.**
  - A TOML error means defaults plus one log line.
  - An unknown key means a warning.
  - A bad segment config means that segment uses defaults.
  - The status line must always print something. Claude Code users won't read a log file, so consider
    showing config errors in the line itself.
- **An env override like `STARSHIP_CONFIG`** (e.g. `CLAUDE_STATUSLINE_CONFIG`) and an XDG default path.
- **Single-quoted TOML in docs and examples**, to avoid double escaping.

### Overkill for a single-line status bar

- `$all` with a hard-coded global order. With about 5 segments, require an explicit order instead.
- `right_format`, `continuation_prompt`, `add_newline`, shell-specific escaping (statusline mode uses
  `Shell::Unknown` anyway) and `TERM=dumb` handling.
- `$fill`. Claude Code runs the command without a TTY, so width detection falls back to 80 columns
  and the fill would be wrong.
- `prev_fg` / `prev_bg`, `inverted` / `blink` / `hidden` / `strikethrough`, and the powerline
  machinery. Nice to have later.
- Palettes and presets as built-in mechanisms. A handful of example configs in the docs is enough.
- `[custom.*]` with `when` / `detect_*` / `shell` / `os`. If we offer custom segments at all, a
  minimal `command` + `format` + timeout is enough. Running shell commands on every status line
  refresh costs latency and is a safety risk.
- `scan_timeout`, since we don't scan directories. Keep a single `command_timeout` for git (run with
  `--no-optional-locks`).

### Parsing complexity

- **Size.** The grammar is small and fully specified by spec.pest, about 50 lines. A hand-written
  recursive-descent parser in TS is roughly 150 lines with four node types: Text, Var,
  Group(format, style), Cond(format). Port the pest rules 1:1 and reuse the test cases from
  `string_formatter.rs` and `parser.rs`.
- **Edge cases to get right:**
  - The escape set is `\ $ [ ] ( )`.
  - `${dotted.name}` is a variable.
  - A style string is literal text plus `$vars`, concatenated *first* and parsed *afterwards*.
  - An inner style replaces the outer one (no merging).
  - Unstyled child segments inherit the parent's style.
  - An empty `[](style)` still emits a segment.
  - Conditionals check variables recursively, including meta variables such as `$symbol`.
- **Two levels of variables.** At the top level, `$x` is a segment's output: a list of styled spans.
  Inside a segment, `$x` is data. Keep "spans with styles" as the intermediate representation and
  serialize to ANSI only at the end, collapsing redundant SGR codes the way `Painted` does
  ([`painted.rs`](https://github.com/starship/starship/blob/152cb69dffc0260d260ea617a7ec763e7314d55c/src/painted.rs)).
- **Two escaping layers** (TOML first, then the format string) are the main source of user
  confusion. Document them.
- **Unknown variables render empty without any message.** The runtime path has to stay quiet, so
  consider a `--check` / `doctor` command that reports unknown variables and invalid styles.
