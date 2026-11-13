# tui

A [Bubble Tea](https://github.com/charmbracelet/bubbletea) / [Lip Gloss](https://github.com/charmbracelet/lipgloss) /
[Bubbles](https://github.com/charmbracelet/bubbles)-inspired terminal UI toolkit for **H#**, distributed as a `bit` package.

Model → Update → View, a synchronous "Cmd" pipeline, ANSI styling with padding / borders / margins / alignment,
layout helpers, mouse + modifier-key input, and a set of ready-made components. **100% H#** — there is no `extern`
FFI block anywhere in this library.

## Install

```
bit add tui
```

## Quick start

```h#
use "bit -> tui" from "tui"

fn init() -> any is
    return tui::spinner_new("dot")
end

fn update(model: any, msg: any) -> (any, fn() -> any) is
    if tui::should_quit(msg) is
        return (model, tui::cmd_quit())
    end
    if tui::msg_is_tick(msg) is
        return (tui::spinner_tick(model), tui::cmd_none())
    end
    return (model, tui::cmd_none())
end

fn view(model: any) -> string is
    return tui::spinner_view(model) + " doing work... (press q to quit)"
end

fn main() is
    tui::run(init, update, view)
end
```

## Source layout

`src/lib.h#` is the public face and is **generated** — it only contains thin `pub fn` wrappers. The code lives in
small modules, one per concern:

| module | contents |
|---|---|
| `tl_text` | width/padding, UTF-8 helpers, `truncate_str`, `wrap_text`, case-insensitive search |
| `tl_term` | raw mode, alt screen, cursor, `draw`, mouse on/off, title |
| `tl_keys` | key + mouse decoding (`poll_key`, `is_special_key`) |
| `tl_msg` | `Msg`, message readers/predicates, commands (`cmd_*`) |
| `tl_color` | colors, `colorize`, `gradient`, `color_lerp` |
| `tl_style` | the `Style` box model (colors, attrs, size, align, padding, border, margin, wrap) |
| `tl_layout` | `join_*`, `place`, `box_titled`, `statusbar`, `indent`, `hline` |
| `tl_program` | `run`, `run_no_tick`, `run_with` + options |
| `tl_spinner` `tl_progress` `tl_textinput` `tl_textarea` `tl_viewport` `tl_list` `tl_checklist` `tl_table` `tl_tabs` `tl_paginator` `tl_help` `tl_timer` `tl_confirm` `tl_chart` | components |

Why a facade? H# has no re-exports: an item of `mod tl_x` is called `tl_x_<name>`, while a user writes `tui::<name>`
(= `tui_<name>`). The wrappers keep every public name exactly as it was.

Adding a function: write it as `pub fn` in the right module (put `;;@internal` on the line above for helpers that
only other modules need), then

```
python3 tools/gen_facade.py   # regenerate src/lib.h#
python3 tools/check.py        # static checks (block balance, call resolution, arity, facade completeness)
```

## API overview

**Program loop** — `run(init, update, view)` (idle tick every ~80 ms), `run_no_tick(...)` (blocks for keys), and
`run_with(init, update, view, opts)` with `options_new()` `options_tick_ms(o, ms)` `options_mouse(o, on)`
`options_alt_screen(o, on)` `options_hide_cursor(o, on)` `options_title(o, t)`. `update` first receives one
`resize` message with the terminal size, and another whenever the size changes. ctrl+c always quits.

**Messages** — `key_msg` `tick_msg` `resize_msg` `mouse_msg` `quit_msg` `none_msg` `custom_msg`; readers `msg_kind`
`msg_key` `msg_data` `msg_width` `msg_height` `msg_x` `msg_y` `msg_mouse_button` `msg_mouse_action`; predicates
`msg_is` `msg_is_key` `msg_is_key_in` `msg_is_any_key` `msg_is_tick` `msg_is_resize` `msg_is_quit` `msg_is_custom`
`msg_is_mouse` `msg_is_click` `msg_is_wheel_up` `msg_is_wheel_down` `is_ctrl_c` `should_quit`.

**Commands** — `cmd_none()` `cmd_quit()` `cmd_custom(data)` `cmd_tick(ms)` `cmd_delay(ms, data)`, or your own
zero-argument closure returning a message.

**Keys** — `"up" "down" "left" "right" "home" "end" "pgup" "pgdown" "insert" "delete" "backspace" "tab"
"shift+tab" "enter" "esc" "space" "f1".."f12" "ctrl+a".."ctrl+z"`, `"alt+<key>"`, `"ctrl|alt|shift+<arrow>"`, or the
character itself (including `ą ę ś …`). Mouse (after `options_mouse`): clicks, drags and the wheel.

**Style** — `style_new()` and `style_fg/bg/bold/italic/underline/faint/strikethrough/reverse/width/height/max_width/
align/valign/padding/margin/border/border_color/wrap`, fluent `.with_*(...)` + `.render(text)`, `style_render`.
Borders: `none normal rounded thick double ascii hidden`. Colors: `""`, `"#rrggbb"`, a 256-palette index, or a name
(`red`, `bright_cyan`, …); `colorize` `colorize_bg` `gradient` `color_lerp` `rgb_hex`.

**Layout** — `join_horizontal(blocks, gap)` `join_horizontal_align(blocks, gap, valign)` `join_vertical`
`join_vertical_align` `place` `center_block` `box_titled` `statusbar` `indent` `hline` `block_width` `block_height`;
text helpers `visible_len` `pad_left/right/center` `truncate_str` `wrap_text` `strip_ansi` `utf8_take/skip/chars`.

**Components**

| component | functions |
|---|---|
| spinner | `spinner_new(kind)` `spinner_custom` `spinner_kinds` `spinner_tick` `spinner_view` … (11 kinds) |
| progress | `progress_new` `_set` `_incr` `_set_width` `_set_chars` `_show_percent` `_set_gradient` `_view` |
| textinput | prompt, password echo, char limit, scrolling width, word jumps, `ctrl+w/k/u` |
| textarea | multi-line editor with line numbers, placeholder, scrolling |
| viewport | scroll by line/page/half page, vim keys, mouse wheel, `viewport_scroll_percent` |
| list | selection, wrap-around, case-insensitive filter (`list_set_filter`), wheel |
| checklist | multi-select with `space` / `a` / `n`, `checklist_selected[_indices]` |
| table | headers, fixed or auto column widths, selectable row, scrolling |
| tabs | `←/→`, `tab`, digits `1-9` |
| paginator | dots, `paginator_view_text`, `pages_for`, `page_bounds` |
| help | `help_view`, `help_view_width` (wraps) |
| confirm | yes/no prompt, `y` / `n` / `enter` |
| timer | stopwatch and countdown (`timer_new_countdown`, `timer_done`, `format_duration_ms`) |
| charts | `sparkline`, `bar_chart` |

See `examples/` for six complete programs (spinner, text input, dashboard, tabs+table+charts+mouse, multi-step
form, text editor) and `tests/tui_test.h#` for tests of the pure functions.

## A note on types

Every public function that hands you back one of this library's own values (a message, a `Style`, a `Spinner`, …)
types it as `any`, and so do the examples — `let sp = tui::spinner_new("dot")`, never `let sp: Spinner = ...`.
H# mangles a package's type names with the alias you chose, so there is no single fixed spelling for your code to
name. Field access and the accessor functions (`tui::msg_key(msg)`, …) work regardless.

## How keys are read (and why not `getchar`)

`__builtin_io_read_char` is `getchar()`. In raw mode with a read timeout (`stty min 0 time 1`) the first idle read
returns zero bytes, which stdio treats as **end of file — and EOF is sticky**: every later `getchar()` returns at
once and never sees a key, so arrows, `q` and ctrl+c are dead while the loop spins. `tui` therefore reads keys
straight from `/dev/tty` with `dd` (one short-lived child per poll, honouring the raw-mode timeout), decodes the
bytes itself, and rebuilds multi-byte characters with `printf`. This also means non-ASCII input now arrives intact.
Requirements: a POSIX shell with `dd`, `od`, `tr`, `printf` and `stty` (coreutils) and a real tty.

## Honest limitations

- Run programs through the C++/native build or `h# preview`; the loop needs a real terminal (`/dev/tty`).
- Idle polling costs one tiny child process per poll (~10 per second); the frame is only redrawn when it changed.
- Text-input / text-area cursors count ASCII characters; the *view* is UTF-8 safe.
- `style_render` word-wraps only with `style_wrap(true)` + a width; otherwise long lines are left as they are.
- Cmds run synchronously on the render thread — keep them fast.
- This version was checked statically (`tools/check.py`) and the key-reading pipeline was tested on a real pty; the
  H# sources themselves could not be compiled in the sandbox it was written in (no h# toolchain). Please build the
  examples and the tests and report anything the compiler objects to.
