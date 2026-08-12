# Hunkset Query Language Reference

The selector language shared by every `jj-hunk` verb and by `list --spec`. Format is
auto-detected: anything that is not JSON/YAML is parsed as a hunkset.

## Contents

- [Operators and precedence](#operators-and-precedence)
- [File-level predicates](#file-level-predicates)
- [Content-level predicates](#content-level-predicates)
- [Line ranges](#line-ranges)
- [Identity](#identity)
- [Semantic predicates](#semantic-predicates)
- [Pattern syntax](#pattern-syntax)
- [The file-level / content-level split](#the-file-level--content-level-split)
- [Decorator attribution differs per language](#decorator-attribution-differs-per-language)

## Operators and precedence

Lowest to highest precedence:

| Operator | Meaning | Example |
|----------|---------|---------|
| `x \| y` | Union | `type(insert) \| type(delete)` |
| `x & y` | Intersection | `type(insert) & glob("src/**")` |
| `x ~ y` | Difference | `all() ~ type(delete)` |
| `~x` | Negation | `~type(delete)` |
| `(x)` | Grouping | `(type(insert) \| type(replace)) & file("x")` |

Union and intersection chain freely. **Difference does not:**

```
$ jj-hunk list --spec 'all() ~ type(delete) ~ type(insert)'
Error: failed to parse hunkset:
all() ~ type(delete) ~ type(insert)
                     ^
unexpected '~' -- difference does not chain; add parentheses, as in '(a ~ b) ~ c'
```

This differs from jj's revsets, where `~` is left-associative. `!` is not an operator
here; `~` covers both negation and difference. `all` and `none` may be written bare when
they are the whole expression (`--spec 'all'` works).

## File-level predicates

| Function | Description |
|----------|-------------|
| `all()` | Every change in the diff |
| `none()` | Nothing |
| `file("path")` | Exact file path match — the whole path, not a suffix |
| `glob("src/**/*.rs")` | Glob pattern on the file path |
| `extension("rs")` | File extension, written without the dot |
| `status(modified)` | `modified`, `added`, `removed`, `renamed`, `copied` |

`status()` takes a **bare identifier, not a string**. A deleted file is `removed` (there
is no `deleted`); an invalid value is rejected with the list of valid ones.

Paths are **relative to the current directory**, so from `src/` you write
`file("svc.rs")`, and `file("src/svc.rs")` matches nothing:

```
$ cd src && jj-hunk list --spec 'file("svc.rs")' --format text
M svc.rs
  hunk 0 insert hunk-e338f6bc (before 6+0 after 6+1) in UserService
```

Hunk **IDs** are not cwd-relative — see [hunk-ids.md](hunk-ids.md).

### Renames match under both paths

`file()`, `glob()` and `extension()` reach a renamed or copied file by its new path *and*
its old one, which is what `--include`/`--exclude` have always done. Three consequences:

- `~glob("secret/*")` **drops** a file renamed out of `secret/` — the diff still spells
  `secret/keys.txt` on its left side, so keeping it would hand you what you excluded.
- `extension()` matches both sides of a rename that changed the extension: `mod.txt` →
  `mod.rs` answers to both `extension("rs")` and `extension("txt")`.
- Matching only. The spec a selection produces is always keyed by the **new** path.

Applies only when jj reports a rename or copy. A move too large for jj's rename detection
is an ordinary add plus delete, and each path names its own change.

## Content-level predicates

| Function | Description |
|----------|-------------|
| `type(insert)` | Insertions only |
| `type(delete)` | Deletions only |
| `type(replace)` | Replacements (removed + added) only |
| `content("text")` | Added or removed text contains "text" |
| `added("text")` | Added text contains "text" |
| `removed("text")` | Removed text contains "text" |

## Line ranges

| Function | Description |
|----------|-------------|
| `lines(10..20)` | Hunks touching lines 10-20, in either the before or the after file |
| `before_line(10..20)` | Hunks in the "before" line range |
| `after_line(10..20)` | Hunks in the "after" line range |

**Ranges include both endpoints**, despite the Rust-looking `..`. `lines(10..20)` includes
line 20, and `lines(7..7)` selects the hunk on line 7. The same applies to `depth()`.

## Identity

| Form | Meaning |
|------|---------|
| `id("hunk-e338f6bc")` | Select by hunk ID — full, short, or any unambiguous prefix |
| `id(hunk-e338f6bc)` | The quotes are optional on an id |
| `id("hunk-e338f6bc", "hunk-7844072c")` | Multiple IDs in one call; forms may be mixed |

`id()` is the one predicate that **errors instead of returning nothing**. Every other
predicate narrows a set, so an empty result is legitimate; `id()` resolves a name, so a
name resolving to nothing is a mistake:

```
Error: hunkset evaluation error: hunk id 'hunk-deadbeef' matches no hunk in this diff -- ids change when the hunk's content or its file changes, so one copied from an earlier listing may be stale. Run 'list' again for the current ids.
```

This fires on `list --spec` too, unlike the JSON-spec validator. An ambiguous prefix is
also an error, with the candidates named rather than guessed at:

```
Error: hunkset evaluation error: hunk id 'hunk-1' is ambiguous -- it matches 2 hunks: hunk-14a47eb4 (f.txt), hunk-1c12dc45 (f.txt). Use more characters, or exact:"<full-id>".
```

The bare `hunk-` is rejected outright: `id() does not accept 'hunk-' -- valid values are:
a hunk id such as hunk-4c1b1b3... (or an unambiguous prefix)`.

`id()` resolves ids, it does not pattern-match them: `substring:`, `glob:` and `regex:`
are rejected on it. Only `exact:` is meaningful, and it means "do not treat this as an
abbreviation".

## Semantic predicates

Backed by tree-sitter. **Only present in a binary built with the `semantic` feature.**

| Function | Description |
|----------|-------------|
| `function("name")` | Hunks inside a function/method with exactly that name |
| `scope("ClassName")` | Hunks inside a class/struct/impl/module with exactly that name |
| `annotation("test")` | Hunks in functions/scopes whose annotation text contains "test" |
| `decorator("route")` | Alias for `annotation()` |
| `doc()` | Hunks that are doc comments |
| `import()` | Hunks that are import/use/require statements |
| `toplevel()` | Hunks not inside any function or scope |
| `depth(0..1)` | Hunks at nesting depth 0 or 1 |

Without the feature, each fails explicitly rather than returning nothing:

```
Error: hunkset evaluation error: function() requires the 'semantic' feature (build with --features semantic)
```

**Supported languages:** Rust, Python, JavaScript, TypeScript, Go, C, C++, Java, Ruby,
C#, Scala, Swift, PHP, Bash, Elixir, Erlang, Haskell, OCaml, Zig, Lua.

For a file in an unsupported language, semantic predicates contribute nothing. A warning
is printed **only when no file in the diff could be parsed at all** — if some file was
analyzed, an empty result is a real answer about real metadata and stays silent:

```
warning: function() found no semantic metadata -- no parser is available for: notes.txt, only.kt. The empty result reflects missing language support, not an absence of matches.
```

`toplevel()` and `depth()` also exclude unparsed files, so an unsupported file is never
mistaken for genuinely top-level code.

## Pattern syntax

String arguments accept a pattern prefix. **The prefix goes outside the quotes.**

| Form | Meaning |
|------|---------|
| `exact:"text"` | Exact match |
| `substring:"text"` | Substring match |
| `glob:"pattern"` | Glob pattern |
| `regex:"pattern"` | Regular expression |

Inside the quotes is a parse error that tells you to move it:

```
$ jj-hunk list --spec 'function("regex:handle.*")'
Error: failed to parse hunkset:
function("regex:handle.*")
                         ^
pattern prefix 'regex:' must go outside the quotes -- write regex:"..." instead of "regex:..."
```

### Bare-string defaults differ per predicate

| Predicate | Bare-string default | Meaning |
|-----------|--------------------|---------|
| `function()`, `scope()` | **exact** | `function("alpha")` does NOT match `alpha_beta` |
| `file()`, `extension()` | **exact** | `file("svc.rs")` does NOT match `src/svc.rs` |
| `content()`, `added()`, `removed()` | substring | `added("TODO")` matches any line containing TODO |
| `annotation()`, `decorator()` | substring | `annotation("test")` matches `#[test]` |

To match a family of identifiers, opt in:

```bash
jj-hunk list --spec 'function(substring:"test")'   # test_a, my_test, test_b ...
jj-hunk list --spec 'function(glob:"test_*")'
jj-hunk list --spec 'function(regex:"^handle_")'
```

Quotes may be dropped around a single bare word, which then follows the same table —
`content(let)` is the substring `let`. Anything containing a space, a dot or a slash needs
the quotes:

```
$ jj-hunk list --spec 'file(svc.rs)'
Error: failed to parse hunkset:
file(svc.rs)
        ^
unexpected '.' -- a range is written '..', as in lines(10..20)
```

### A malformed glob is an error, everywhere

`glob()`, `file(glob:)`, `--include` and `--exclude` all reject one outright rather than
matching nothing — which matters most under `~`, where "matches nothing" would have
inverted into "matches everything":

```
Error: hunkset evaluation error: invalid glob 'src/[': unclosed '[' -- a character class needs a matching ']'
```

## The file-level / content-level split

Five kinds of change have no hunks: a binary file, a symlink retarget, a mode-only flip, a
pure rename or copy, and an add or remove of an empty file. Each carries a stand-in hunk
holding a path and a status and no text, *marked* as a stand-in.

| Reach them (select them whole) | Never reach them |
|--------------------------------|------------------|
| `all()`, `file()`, `glob()`, `extension()`, `status()`, any negation `~x` | `content()`, `added()`, `removed()`, `lines()`, `id()` |

The right column is deliberate, not an oversight. The marking is what does the work: empty
text is not unmatchable text, and both `content("")` and `lines(0..N)` would be true of
it. A question about content is refused rather than answered about bytes that were never
diffed — a symlink's target and a renamed file's body are not in any hunk.

```
$ jj-hunk list --spec 'all()' --format text
M blob.bin [binary]
A brand_new_empty.txt
M config [symlink, whole-file only]
R moved.txt (tomove.txt -> moved.txt)
M script.sh [mode 100644 -> 100755, not selectable]
M text.txt
  hunk 0 replace hunk-1a94e681 (before 2+1 after 2+1)
    - world
    + WORLD

$ jj-hunk list --spec 'content("WORLD")' --format text
M text.txt
  hunk 0 replace hunk-1a94e681 (before 2+1 after 2+1)
    - world
    + WORLD

$ jj-hunk list --spec 'lines(0..999)' --format text
M text.txt
  hunk 0 replace hunk-1a94e681 (before 2+1 after 2+1)
    - world
    + WORLD
```

`glob("**")` behaves like `all()` here and reaches all six.

## Decorator attribution differs per language

If a hunk touches **only** a decorator/attribute line and not the function body, the
grammar decides whether that line counts as part of the function:

| Attributes to the FUNCTION | Attributes to the enclosing SCOPE only |
|---------------------------|----------------------------------------|
| Java, C#, Swift, Scala, PHP, JavaScript | Rust, Python, TypeScript (`.ts`, `.tsx`) |

So for a one-line change to `@Test(timeout = 1)` above `my_test`, `function("my_test")`
selects it in Java and selects nothing in Rust, Python or TypeScript. Note that
JavaScript and TypeScript disagree on identical source text.

When a selection must cover decorators portably, use `scope()`, a line range, or explicit
hunk IDs instead of `function()`.
