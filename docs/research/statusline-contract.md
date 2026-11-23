# Claude Code `statusLine` contract: stdin payload, output, settings

Research for issue #4. Researched 2026-11-22 against Claude Code **v2.1.330**.

## Sources

| Tag | Source |
|---|---|
| [SL] | Status line docs: https://code.claude.com/docs/en/statusline (raw markdown: https://code.claude.com/docs/en/statusline.md) |
| [SR] | Settings reference, `statusLine` key: https://code.claude.com/docs/en/settings-reference#statusline |
| [CL] | Changelog: https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md |
| [LIVE] | Payloads captured locally from Claude Code 2.1.330 (method below) |

### How the live capture was done

The user's `~/.claude/settings.json` was **not** touched. Claude Code was started in a
detached tmux session with an extra settings layer:
`claude --settings /tmp/slcap/settings.json`, where that file held only
`{"statusLine":{"type":"command","command":"/tmp/slcap/cap.sh"}}`. The script saved
stdin to a file, saved `COLUMNS`/`LINES`, and printed a two-line test string with
16-color, 256-color, truecolor and OSC 8 escapes. Three payloads were captured: session
start, and two after one trivial prompt ("Reply with just the word OK"). A second run used
a script that sleeps 12 s to test timeouts.

## Headline answers

- **`cost.total_cost_usd` is correct.** It is documented [SL] and was present in every
  live payload [LIVE]. It is a float in USD (live value `0.18479620000000002`, so round it
  when you format it). It is `0` at session start, not absent. It is an estimate computed
  client-side at list price (or at the org's `modelPricing` rates), not your bill. It resets
  to $0 on `/clear` [SL]. It was added in 1.0.85 ("session cost info") [CL]. The `/clear`
  reset was fixed in 2.1.211 [CL].
- **Multi-line output: yes.** Each line of stdout is its own row [SL]. Confirmed live: two
  rows rendered [LIVE].
- **ANSI colors: yes.** 16-color SGR (`\033[3Xm`) and 256-color (`\033[38;5;Nm`) were
  passed through live. Truecolor (`\033[38;2;R;G;Bm`) was **downsampled to the nearest
  256-color index** inside tmux (`TERM=tmux-256color`, `COLORTERM=truecolor`) [LIVE].
  Claude Code re-renders your escapes through its own renderer instead of copying bytes
  straight to the terminal. Use 16 or 256 colors, and treat truecolor as "may be
  approximated". Text you leave uncolored, including text after `\033[0m`, rendered in
  Claude Code's dim default color (`38;5;145`), not the terminal's default foreground
  [LIVE].
- **OSC 8 hyperlinks: yes, if the terminal supports them.** [SL] documents them (iTerm2,
  Kitty, WezTerm. Not Terminal.app). Claude Code must detect hyperlink support. Override
  with `FORCE_HYPERLINK=1`, and since 2.1.314 `FORCE_HYPERLINK=false|no|off` turns them
  off [CL]. SSH and tmux may strip the sequences [SL]. Live, the OSC 8 link was passed
  through and rewritten with an `id=` parameter and the ST terminator (`\e]8;id=...;URL\e\\`)
  [LIVE].
- **Timeouts:** no fixed timeout is documented. Output is shown only after the script
  **exits 0**. A non-zero exit or empty output blanks the status line. A slow script keeps
  the old output on screen until it finishes. If a new update triggers while the script is
  still running, the running script is **cancelled** [SL]. Live: a 12 s script was not
  killed and its output was shown [LIVE].
- **Refresh:** event-driven with a **300 ms debounce**, plus an optional `refreshInterval`
  timer [SL]. Details below.

## Settings schema (`statusLine`)

From [SR] and [SL]:

| Key | Type | Required | Meaning |
|---|---|---|---|
| `type` | `"command"` | yes | The only documented type. |
| `command` | string | yes | Runs **in a shell**. Can be a script path or an inline command such as a `jq` one-liner. `~` works in the doc examples. |
| `padding` | number (characters) | no | Extra horizontal spacing added on top of the UI's built-in spacing. Default `0`. |
| `refreshInterval` | number (seconds), min `1` | no | Re-runs the command every N seconds in addition to the event triggers. Unset means events only. Added in 2.1.97 [CL]. |
| `hideVimModeIndicator` | boolean | no | Hides the built-in `-- INSERT --` text, for when your script renders `vim.mode` itself. |

- Scope: any settings file (user, project, local, managed, `--settings`) [SR].
- Gates [SR]: the status line is **off** when managed settings set `disableAllHooks`, or
  until the folder's workspace trust dialog is accepted. It is **narrowed to managed
  settings only** (your value is skipped without warning) under `allowManagedHooksOnly`,
  under `disableAllHooks: true` set outside managed settings, or with `--safe-mode`.
- `subagentStatusLine` was removed in 2.1.323. Subagent rows always use the default look
  [CL]. Ignore it.
- `/statusline` (the slash command) writes a script into `~/.claude/` and updates settings
  for you [SL].

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh",
    "padding": 0,
    "refreshInterval": 5,
    "hideVimModeIndicator": false
  }
}
```

## Execution environment and output

- stdin gets one JSON object. Your script's **stdout** is displayed. stderr is not
  displayed, and `claude --debug` logs it [SL].
- Output is captured, not attached to a TTY, so `tput cols` does not work. Read the
  **`COLUMNS`** and **`LINES`** env vars instead [SL] (added in 2.1.153 [CL]). Live values
  matched the pane: `COLUMNS=160`, `LINES=40` [LIVE].
- Keep output short. Notifications (MCP errors, auto-update, the context-low warning)
  share the status line row on the right outside fullscreen rendering and can truncate
  your output on narrow terminals [SL].
- Multi-line output combined with escape codes is "more prone to rendering issues" [SL].
  2.1.141 fixed rows being dropped when a line is wider than the terminal [CL]. 2.1.329
  fixed a script that prints endlessly freezing the prompt [CL].
- The status line hides during the help menu and permission prompts, and it uses no API
  tokens [SL].
- The custom status line is a separate row above the built-in footer badges. With it
  configured, some footer hints such as `? for shortcuts` are not shown [SL].

## Refresh cadence and triggers

From [SL]. The script runs once at session start or resume, then again when:

- a new assistant message arrives
- `/compact` finishes
- the permission mode changes
- vim mode toggles
- the `statusLine.command` setting changes (this one skips the debounce and runs at once)
- the `refreshInterval` timer elapses, if set
- a `rate_limits.*.resets_at` time in the last payload is reached
- a warm `prompt_cache.expires_at` in the last payload is reached

Updates are **debounced at 300 ms**. A new trigger **cancels an in-flight script**. Event
triggers can go quiet while the main session is idle (for example, while it waits on
background subagents). Use `refreshInterval` for clocks or git state that changes outside
the session [SL]. Live: one prompt produced two runs after the start-up run, one before and
one after the response [LIVE].

Implication for us: the command runs often and gets killed on overlap. Keep it a single
fast process with at most one `git --no-optional-locks` call, and never `bunx ...@latest`.

## Stdin payload fields

Types: `str`, `num`, `bool`, `obj`, `arr`. "Absent" means the key is missing. "null" means
the key is present with value `null`. Unless noted, each row is from the [SL] field table
and the "Full JSON schema" accordion.

### Always present (observed in every live payload)

| Field | Type | Semantics |
|---|---|---|
| `session_id` | str (UUID) | Unique session id. |
| `transcript_path` | str | Path to the session's `.jsonl` transcript under `~/.claude/projects/...`. |
| `cwd` | str | Current working directory. Same value as `workspace.current_dir`. The docs prefer `workspace.current_dir`. |
| `model.id` | str | Model id, e.g. `claude-opus-5-5`. |
| `model.display_name` | str | e.g. `Opus 5.5` live. The docs example shows `Opus`, but live includes the version. |
| `workspace.current_dir` | str | Current working directory. |
| `workspace.project_dir` | str | Directory Claude Code was launched in. Can differ from `cwd` after the directory changes. |
| `workspace.added_dirs` | arr of str | Directories added with `/add-dir` or `--add-dir`, `[]` if none (2.1.47 [CL]). |
| `version` | str | Claude Code version, e.g. `2.1.330`. |
| `output_style.name` | str | Current output style, e.g. `default`. |
| `cost.total_cost_usd` | num (float) | Estimated session cost in USD (see headline). `0` at start. |
| `cost.total_duration_ms` | num | Session wall-clock time. Adds up across resumes and excludes time the session is not running. |
| `cost.total_api_duration_ms` | num | Total time spent waiting on API responses. |
| `cost.total_lines_added` / `cost.total_lines_removed` | num | Lines of code changed in the session. |
| `context_window.total_input_tokens` | num | Tokens currently in context from the **latest** response: `input + cache_creation + cache_read`. `0` before the first response. Was cumulative until fixed in 2.1.132 [CL]. |
| `context_window.total_output_tokens` | num | Output tokens of the latest response. `0` before the first response. |
| `context_window.context_window_size` | num | Max context in tokens. Live: `1000000` for Opus 5.5. |
| `context_window.used_percentage` | num or **null** | Input-only percentage (`input + cache_creation + cache_read`, excludes output). **null early in the session**. Added in 2.1.6 [CL]. Live it was an integer (`4`). |
| `context_window.remaining_percentage` | num or **null** | `100 - used`. null early. |
| `context_window.current_usage` | obj or **null** | Per-category counts from the last API call: `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`. **null until the first response**, and null or all zeros after `/compact` until the next call. Treat both as empty. Added in 2.0.70 [CL]. |
| `exceeds_200k_tokens` | bool | Whether the latest response's total tokens (input + cache + output) exceed a fixed 200k, regardless of window size (1.0.88 [CL]). |
| `fast_mode` | bool | Whether fast mode is on. |
| `thinking.enabled` | bool | Whether extended thinking is on (2.1.119 [CL]). |

### Conditionally present (may be absent)

| Field | Type | When present |
|---|---|---|
| `session_name` | str | Custom name (`--name` or `/rename`) or an AI-generated title. Absent at start. Live it appeared after the first prompt (`"Ok"`). |
| `prompt_id` | str (UUID) | After the first user input. Matches OTel `prompt.id`. |
| `effort.level` | `low`/`medium`/`high`/`xhigh`/`max` | Only when the model supports the effort parameter (2.1.119 [CL]). Live: `medium`. |
| `workspace.git_worktree` | str | Only inside a linked git worktree. Absent in the main working tree (2.1.97/2.1.98 [CL]). Live: the worktree dir name. |
| `workspace.repo.{host,owner,name}` | str | Only inside a git repo with an `origin` remote. Parsed from origin. GitLab subgroups appear as `owner: "group/subgroup"`. |
| `pr.number` / `pr.url` | num / str | Only while an open PR (or GitLab MR) exists for the branch (2.1.145 [CL]). |
| `pr.review_state` | `approved`/`pending`/`changes_requested`/`draft` | Can be absent even when `pr` is present. |
| `pr.kind` | `"mr"` | Only for GitLab merge requests (2.1.234 [CL]). |
| `vim.mode` | `NORMAL`/`INSERT`/`VISUAL`/`VISUAL LINE` | Only when vim mode is enabled. |
| `agent.name` | str | Only with `--agent` or agent settings. |
| `worktree.{name,path,branch,original_cwd,original_branch}` | str | Only in a Claude `--worktree` session (2.1.69 [CL]). `branch` and `original_branch` are absent for hook-based worktrees. |
| `rate_limits.five_hour` / `.seven_day` | `{used_percentage: num 0-100, resets_at: epoch s}` | Claude.ai Pro/Max/Ultra only, after the first response (2.1.80 [CL]). **Each window can be missing on its own.** Live: only `seven_day` was present. A window is dropped once its `resets_at` passes. |
| `rate_limits.spend_limit` | `{used_percentage, resets_at, used_usd?, limit_usd?, period?}` | Only behind a Claude apps gateway with spend limits (2.1.251, and the USD fields need 2.1.284 [CL]). The USD fields can lag about 5 minutes and can be absent. |
| `prompt_cache` | obj | After the main conversation's first response (2.1.251 [CL]). Fields: `warm`, `caching_observed`, `ttl` (`5m`/`1h`), `expires_at` (epoch s or null), `requests`, `misses`, `expected_rebuilds`, `hit_ratio` (0-1 or null), `cache_write_tokens`, `miss_recache_tokens`, `last_miss_at` (epoch s or null), `last_miss_cause` (obj or null, 2.1.260), `miss_causes` (obj), `recache_tokens_if_cold` (num or null). |

## Annotated example payload (real, captured live after one prompt)

Captured from 2.1.330 [LIVE]. Comments added. The paths are the real worktree paths.

```jsonc
{
  "session_id": "3f71520b-353b-4345-849c-c13aff388b9b",
  "transcript_path": "/Users/<user>/.claude/projects/<slug>/3f71520b-....jsonl",
  "cwd": "/Volumes/.../claude-statusline/.claude/worktrees/agent-afc675e8146d88781", // == workspace.current_dir
  "prompt_id": "69165a01-2d68-4fb6-9abc-7cd27313e024", // absent before first prompt
  "effort": { "level": "medium" },                 // absent if model has no effort param
  "session_name": "Ok",                            // absent before a title/name exists
  "model": { "id": "claude-opus-5-5", "display_name": "Opus 5.5" },
  "workspace": {
    "current_dir": "/Volumes/.../agent-afc675e8146d88781", // prefer this, then cwd
    "project_dir": "/Volumes/.../agent-afc675e8146d88781", // launch dir
    "added_dirs": [],
    "git_worktree": "agent-afc675e8146d88781",     // only inside a linked worktree
    "repo": { "host": "github.com", "owner": "AbsolutOD", "name": "claude-statusline" } // only with an origin remote
  },
  "version": "2.1.330",
  "output_style": { "name": "default" },
  "cost": {
    "total_cost_usd": 0.18479620000000002,  // float USD, client-side estimate, 0 at start, resets on /clear
    "total_duration_ms": 16570,
    "total_api_duration_ms": 3413,
    "total_lines_added": 0,
    "total_lines_removed": 0
  },
  "context_window": {
    "total_input_tokens": 35752,     // == 2 + 22748 + 13002 (current context, not cumulative)
    "total_output_tokens": 4,
    "context_window_size": 1000000,
    "current_usage": {               // null before the first response, null or zeros right after /compact
      "input_tokens": 2,
      "output_tokens": 4,
      "cache_creation_input_tokens": 22748,
      "cache_read_input_tokens": 13002
    },
    "used_percentage": 4,            // null before the first response; input-only
    "remaining_percentage": 96
  },
  "exceeds_200k_tokens": false,
  "prompt_cache": {                  // absent before the first response
    "warm": true, "caching_observed": true, "ttl": "1h", "expires_at": 1795406705,
    "requests": 1, "misses": 0, "expected_rebuilds": 0, "hit_ratio": 0.36367196240769745,
    "cache_write_tokens": 22748, "miss_recache_tokens": 0, "last_miss_at": null,
    "last_miss_cause": null, "miss_causes": {}, "recache_tokens_if_cold": 35752
  },
  "fast_mode": false,
  "thinking": { "enabled": true },
  "rate_limits": {                   // subscribers only; each window can be missing on its own
    "seven_day": { "used_percentage": 2, "resets_at": 1795640400 }
  }
}
```

The **session-start** payload was the same shape minus `prompt_id`, `session_name`,
`prompt_cache` and `rate_limits`, with `total_cost_usd: 0`, `total_input_tokens: 0`,
`current_usage: null`, `used_percentage: null` and `remaining_percentage: null` [LIVE].

## Implications for the package (HANDOFF spec check)

- `cost.total_cost_usd`: the field name is right. Format with `toFixed(2)`. "n/a" only
  happens on very old Claude Code (before 1.0.85). Expect `$0.00` at session start, not
  `n/a`.
- Context "used" = `current_usage.input_tokens + cache_creation_input_tokens +
  cache_read_input_tokens` matches the documented `used_percentage` formula [SL]. The
  fallback `context_window.total_input_tokens` is the same sum on 2.1.132+ [SL][CL][LIVE],
  so the fallback is safe. Handle `current_usage: null` (not just absent) and
  `used_percentage: null`.
- `model.display_name` is now `"Opus 5.5"`, not `"Opus"`. The output spec example shows
  `Opus`, so decide whether to keep the full name.
- Directory: `workspace.current_dir` → `cwd` → `process.cwd()` is fine. Both are always
  present. `workspace.git_worktree` and `workspace.repo` could replace some git calls, but
  they have no branch name, so we still need one git call (or `worktree.branch`, which only
  exists in `--worktree` sessions).
- Uncolored segments render in Claude Code's dim gray, not the terminal default [LIVE].
  Plain ANSI 31-36 render as specified.
- Multi-line, 256-color and OSC 8 links are available if we ever want them.
- Always exit 0 and always print something. Otherwise the line goes blank [SL].

## Open / not verified

- No numeric timeout is documented. Live, a 12 s script was allowed to finish. A longer
  hard cap may exist but was not tested.
- Truecolor downsampling was seen inside a tmux pane. Whether a native truecolor terminal
  (iTerm2, Ghostty) gets 24-bit color passed through was not tested.
- `padding` behavior was not tested visually.
