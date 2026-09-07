# hprompt

Line-editing & history for **H#**, in the spirit of Rust's [`rustyline`](https://github.com/kkawakam/rustyline).

```sh
bytes add hprompt
```

## Honest limitation — read this first

`std -> term`'s `enable_raw` / `disable_raw` / `read_key` are documented,
literal no-ops in the current H# runtime — there is no termios/raw-mode
primitive anywhere in the interpreter or the LLVM backend, confirmed
directly against the interpreter's own source. `std -> io`'s `read_char`
is a documented no-op too. That means **no H# program, this library
included, can currently do live, redraw-as-you-type, arrow-key-driven
line editing** — the thing `rustyline`'s core is actually built on.

What `hprompt` *can* honestly give you, all built on the real
`io::read_line` (a working, `fgets`-based, canonical-mode stdin read
that every H# program already uses):

- **In-line editing already works for free.** Backspace, Ctrl-W,
  Ctrl-U, cursor movement within the current line — the kernel's own
  tty line discipline handles all of that before H# ever sees the
  bytes. Nothing here needs to (or could) reimplement it.
- **History recall** via bash-style expansion instead of arrow keys,
  since arrow keys can't be intercepted: type `!!` + Enter to reuse
  the last line, `!3` for entry 3 (1-based), `!git` for the most
  recent entry starting with "git".
- **Tab-completion** as a "finish the word, press Tab, press Enter"
  workflow — a trailing literal Tab byte, which canonical mode passes
  straight through — not a live dropdown.
- **Ctrl-C is not interceptable.** Canonical-mode stdin delivers it as
  a real `SIGINT` that the OS default-kills the process with, before
  any H# code runs. See `std -> signal` if you need to catch that
  separately from line reading.

If/when H# ships real raw-mode + single-key-read primitives, only
`editor_readline_with`'s internals need to change — `History`,
`Completer`, and the bash-style expansion are all backend-agnostic
already.

One more thing worth knowing: the underlying `io::read_line` native
(`hsh_env_read_line` in the compiler runtime) returns `""` both for a
real EOF (stdin closed / Ctrl-D) *and* for the user pressing Enter on
a blank line — `fgets` returning `NULL` and reading an empty line are
indistinguishable by the time the string reaches H#. `hprompt` can't
fix that at this layer, so `Editor`'s `blank_line_is_eof` field
(default `true`) lets *you* pick which behavior fits your program
instead of the library silently assuming one for you.

## Why every function is free-standing (no `ed.readline()`)

A `use "std -> x" from "alias"` import gets every top-level `fn`,
`struct`, and `enum` it defines renamed to `alias_Name` when the
compiler inlines it (`hsharp-compiler`'s `modules.rs` —
`mangle_module_items`). That same pass documents, in its own comment,
that it does *not* rewrite pattern-matches on an imported enum, and
cross-module static calls shaped like `alias::Type::method()` have no
matching case in the interpreter's `call_path` dispatch (only bare
`Type::method` 2-segment paths do). Plain `alias::function(...)` calls
are the one call shape guaranteed to resolve correctly on both
backends, so that's the entire public surface: every "object" here is
just a `History` / `Completer` / `Editor` value threaded through free
functions, builder-style — the same pattern `std -> cli`'s `ArgParser`
already uses, for the same reason. Update a value by reassigning the
result of a call:

```h#
h = hp::history_add(h, line).0   ;; or destructure: let (h, added) = ...
```

not `h.add(line)`.

`ReadResult` is a plain tagged struct (`{ kind: "line" | "eof", line:
string }`) rather than an enum, for the same reason: matching an
imported enum is the documented-incomplete path above, so this
sidesteps it. Use `hp::result_is_line(r)` / `hp::result_is_eof(r)` and
read `.line`.

## Quick start

```h#
use "std -> hprompt" from "hp"

fn main() is
    let mut ed = hp::editor_new()
    ed = hp::editor_set_prompt(ed, "myshell> ")
    ed = hp::editor_set_completer(ed, hp::completer_from_words(["help", "quit", "status"]))
    ed = hp::editor_set_tab_complete(ed, true)
    let (e1, _ok) = hp::editor_load_history(ed, ".myshell_history")
    ed = e1

    while true is
        let (e2, result) = hp::editor_readline(ed)
        ed = e2

        if hp::result_is_eof(result) is
            break
        end

        let line = result.line
        if line == "quit" is
            break
        end
        write("you said: " + line)
    end

    hp::editor_save_history(ed, ".myshell_history")
end
```

## API

### `History`

| function | does |
|---|---|
| `history_new()` | empty history, capacity 1000 |
| `history_with_capacity(n)` | empty history, capacity `n` |
| `history_add(h, line) -> (History, bool)` | add a line; `bool` is false if blank or a consecutive duplicate |
| `history_len(h) -> int` | |
| `history_is_empty(h) -> bool` | |
| `history_clear(h) -> History` | |
| `history_get(h, idx) -> string` | 0-based, `""` if out of range |
| `history_last(h) -> string` | `""` if empty |
| `history_all(h) -> [string]` | oldest first |
| `history_search_prefix(h, prefix) -> [string]` | most-recent-first |
| `history_search_contains(h, needle) -> [string]` | most-recent-first |
| `history_expand(h, input) -> string` | bash-style `!!` / `!N` / `!prefix` expansion |
| `history_save(h, path) -> bool` | one entry per line, oldest first |
| `history_load(h, path) -> (History, bool)` | appends; `bool` is false if the file doesn't exist |

### `Completer`

| function | does |
|---|---|
| `completer_new()` | empty word list |
| `completer_from_words(words)` | |
| `completer_add_word(c, w) -> Completer` | |
| `completer_complete(c, partial) -> [string]` | prefix matches, insertion order |

### `Editor`

| function | does |
|---|---|
| `editor_new()` | prompt `"> "`, auto-history on, bang-expansion on, tab-complete off, blank-is-eof on |
| `editor_set_prompt(e, prompt) -> Editor` | |
| `editor_set_completer(e, c) -> Editor` | |
| `editor_set_tab_complete(e, bool) -> Editor` | |
| `editor_set_blank_line_is_eof(e, bool) -> Editor` | |
| `editor_readline(e) -> (Editor, ReadResult)` | reads one line using `e`'s prompt |
| `editor_readline_with(e, prompt) -> (Editor, ReadResult)` | one-off prompt, doesn't change `e` |
| `editor_add_history_entry(e, line) -> bool`* | |
| `editor_load_history(e, path) -> (Editor, bool)` | |
| `editor_save_history(e, path) -> bool` | |
| `editor_clear_history(e) -> Editor` | |
| `editor_history_len(e) -> int` | |

\* fields are `pub`, so `e.history`/`e.prompt`/etc. are also directly
readable — the setters above exist for the common cross-boundary
"build a new value and reassign" idiom.

### `ReadResult`

`{ kind: "line" | "eof", line: string }` — check with
`result_is_line(r)` / `result_is_eof(r)`.

## Testing notes

Every function above was exercised through `hsharp-parser` +
`hsharp-interpreter` directly (bypassing only the LLVM-dependent parts
of the toolchain, which this environment couldn't build LLVM 21 for)
with piped stdin, including the tab-completion round trip and a
save/load-history round trip through a real file. In the course of
that testing, a real, reproducible interpreter bug turned up: a plain
variable mutation placed *after* an `if` whose true branch ran, inside
a `while` loop body, silently fails to take effect on the next
iteration (confirmed with a 4-line repro with no structs, arrays, or
library code involved at all — just `while`/`if`/reassignment). Every
loop in `src/lib.h#` that needs a counter to advance therefore
advances it *before* the `if`, with a `let cur = i` local carrying the
pre-advance index into the `if` — see the comment above `history_add`
in the source for the specifics.

## License

MIT
