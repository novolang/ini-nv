# ini-nv

An INI file is a text file of `key = value` lines grouped under
`[section]` headers. There is no specification for it. What a file means
is what the program that wrote it meant, so this package reads it under
a **dialect** the caller names, and ships the dialects of Python's
[configparser](https://docs.python.org/3/library/configparser.html) and
Rust's [rust-ini](https://docs.rs/rust-ini). It also writes a file back
with its comments and its order intact, and converts a document into
[config-core-nv](https://novo-lang.org/packages/config-core-nv)'s value
tree.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

A **section** is a `[name]` header and the lines under it. An **entry**
is a key, a delimiter and a value. A **fallback section**, written
`[DEFAULT]` by configparser, supplies a value to every section that
does not have one of its own.

The implementations disagree about what a line means, and each
disagreement is a field of the dialect.

| Line | One reading | The other |
| --- | --- | --- |
| `a = b ;c` | The value is `b ;c` (configparser) | The value is `b` (rust-ini) |
| `a: b` | An entry (configparser) | A syntax error (many C parsers) |
| `[p]` twice | Refused, merged, or a second section | depending on the dialect |
| `User = x` | The key is `user` (configparser) | The key is `User` (everyone else) |

`IniOptions` is that set of decisions as one value.

| Field | Default here | What it decides |
| --- | --- | --- |
| `delimiters` | `=` and `:` | Which characters separate a key from its value |
| `comment_prefixes` | `#` and `;` | Which characters start a whole-line comment |
| `inline_comment_prefixes` | none | Which characters start a comment after a value |
| `continuations` | on | Whether a further-indented line continues the value above it |
| `lowercase_keys` | off | Whether a key is lowercased on the way in |
| `implicit_section` | `""` | The section a key before the first header goes into, or a refusal |
| `allow_empty_values` | on | Whether a delimiter with nothing after it is an empty value |
| `allow_no_value` | off | Whether a bare key with no delimiter is an entry |
| `duplicate_keys` | refuse | What the same key twice means |
| `duplicate_sections` | refuse | What the same header twice means |
| `default_section` | `DEFAULT` | The fallback section's name, or `""` for none |
| `trim_values` | on | Whether whitespace is stripped from both ends of a value |

Three constructors are three answers somebody already depends on.
`iniread.defaults` is this package's own. `iniread.configparser`
lowercases keys, refuses a key before the first header, and keeps
`[DEFAULT]`. `iniread.rust_ini` treats `;` as an inline comment, keeps
every duplicate, has no fallback section, and puts a key before the
first header into the unnamed section.

A **document** keeps the file's trivia: a section holds its entries in
order, and an entry holds the comments written above it, the comment
written after it, and its own delimiter. `iniwrite.round_trip` of a
parsed document is the text it was parsed from.

**Interpolation** is a reference from one value to another.
configparser has two spellings, and this package has both plus the
option of neither.

| Style | Spelling | What it can reach |
| --- | --- | --- |
| `IniInterpBasic` | `%(name)s` | This section, then the fallback section |
| `IniInterpExtended` | `${name}`, `${section:name}` | Anywhere in the document |
| `IniInterpNone` | — | Nothing. `%` and `$` are ordinary characters |

`IniInterpNone` is what a systemd unit or a `.gitconfig` needs: neither
format has interpolation and both use `%` and `$` for something else.

## Install

```
novo pkg add ini-nv
```

## Example

```novo
use inidoc
use inierror
use iniread
use iniwrite

fn main() [io]
    let text = "[web]\n# the port the server listens on\nport = 8080\nhost = localhost\n"

    // Read the text the way Python's configparser would read it.
    match iniread.parse(text, iniread.configparser())
        Err(f)  => println(inierror.message(f))
        Ok(doc) =>
            // One value, exactly as the file wrote it.
            println(inidoc.get(doc, "web", "port") ?? "unset")

            // Change that value and write the file back out. The
            // comment above the key and the order of the file stay.
            let next = inidoc.set(doc, "web", "port", "9090")
            println(iniwrite.render(next, iniwrite.standard()))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented:
ini-nv.<module>.<fn>` panic. The tests are the specification the
implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `iniread` | The dialect, the three named constructors, the parse, and the line classifier the parse is built on. |
| `inidoc` | The document, the section and the entry, the reads over them, and the edits that keep the comments. |
| `iniinterp` | The two interpolation spellings: expand one value, expand a document, list the references in a value, and find every dangling one. |
| `iniwrite` | Rendering: the byte-for-byte round trip, three styles, a render into a caller's buffer, and the length one would take. |
| `inicfg` | The conversion into config-core-nv's value tree, and back out of it. |
| `inierror` | Every reason a file has no reading under a dialect, each with the line and the column. |

## How to choose an entry point

**`iniread.read` takes text and this package's own dialect.** One
argument, for a caller who has no particular tool's file.

**`iniread.parse` takes text and a dialect.** Use it, with
`iniread.configparser` or `iniread.rust_ini`, when you know which
program wrote the file.

**`inidoc` is how a document is read and changed.** `get` answers a
value, `set` changes one and leaves the comment above it, and
`get_all` answers every value of a repeated key.

**`iniinterp.value` is `inidoc.get` with the references resolved.** It
is a separate call on purpose; see rule 3.

**`inicfg.to_config` is the one call a layered configuration reader
makes.** It answers a table of sections, each a table of keys.

## The rules a user needs

1. **The dialect decides what a line means, and there is no right
   default.** `a = b ;c` is a value of `b ;c` under this package's own
   dialect and under configparser's, and a value of `b` under
   rust-ini's.
2. **An inline comment is off by default.** `url = http://x/#frag`
   keeps its fragment. That is configparser's behaviour, and it
   surprises people. `iniread.rust_ini` turns `;` on.
3. **`inidoc.get` answers the raw text, always.** configparser expands
   references on the way out of its `get`, so a program that reads a
   value, changes something else and writes the file back has replaced
   every reference with whatever it resolved to that day. Here the
   expansion is `iniinterp.value`, a call of its own.
4. **A chain of references resolves and a cycle is named.** `a = %(b)s`
   with `b = %(a)s` answers `IniInterpolationCycle` at the value that
   closed the loop, rather than a depth limit.
   `iniinterp.INTERPOLATION_DEPTH_LIMIT` is 10.
5. **`iniinterp.check` finds every dangling reference in one pass.**
   Run it at startup, so that a setting nobody reads until Tuesday does
   not fail then.
6. **The same key twice has four possible meanings, and the document
   carries which.**

   | Policy | Whose default it is |
   | --- | --- |
   | `IniDupRefuse` | configparser with `strict=True`, and this package |
   | `IniDupFirstWins` | most hand-written C parsers |
   | `IniDupLastWins` | configparser with `strict=False`; a repeated section merges |
   | `IniDupKeepAll` | rust-ini's multi-value properties, and a systemd unit's repeated `After=` |

7. **`inidoc.get_all` is how a repeated key is read**, and a repeated
   key is the only list INI has. It becomes a list in the config tree.
8. **A parsed document rendered back is the file it came from.**
   `iniwrite.round_trip` makes no style decisions. `iniwrite.render`
   takes a style, for a document a caller built rather than read.
9. **`inidoc.set` keeps the comment and `inidoc.remove` takes it
   away.** A changed value keeps the sentence that explains it; a
   removed key takes its explanation with it, because a file holding
   the explanation of a setting nobody has is worse than one that lost
   both.
10. **A section name is one key, dots included.** `[a.b]` is a section
    whose name contains a dot, not a table `a` holding a table `b`. INI
    has no nesting, and two tools that write `[a.b]` mean different
    things by it.
11. **Every value is a string.** `inicfg.to_config` answers strings
    throughout. `inicfg.to_config_typed` runs each value through
    config-core-nv's own `cfgvalue.infer_scalar`, so a port number from
    an INI file and one from the environment are typed by the same
    rule.
12. **The fallback section is not flattened by default.**
    `IniDefaultsAsSection` keeps `[DEFAULT]` as a section of its own.
    Copying its values into every section would make a file's fallback
    outrank a later layer's explicit setting once config-core-nv merges
    them, which is backwards. `IniDefaultsFlattened` is there for a
    caller reading one file and layering nothing.
13. **`[]` is refused.** An empty section name is a section nothing can
    ask for. configparser accepts it and rust-ini does not.
14. **A key before the first section header is refused unless the
    dialect names a section for it.** `IniOptions.implicit_section` is
    that name, and a `.gitconfig` fragment or a `pip.conf` snippet
    needs it set.
15. **Every fault carries a 1-based line and a 1-based byte column.**
    The column is 1 when the whole line is the problem. `inierror`
    never sees the source text, so a message that quotes the offending
    line is built by the caller.
16. **`inierror.kind_name` is stable across releases.** The spellings
    are lower case with hyphens, such as `missing-delimiter`, because
    programs quote them in their own messages and tests.
17. **Text is UTF-8.** novo-lang's `Str` is UTF-8, so a caller holding
    other bytes decodes them first.

## What is not included

- **A type inference of this package's own.** See rule 11.
- **Nesting.** See rule 10.
- **An encoding option.** See rule 17.
- **`.gitconfig`'s subsections**, written `[remote "origin"]`. That is
  a real dialect this package does not have. `iniread.classify` is
  public so a caller can write the few lines that do, without deriving
  the comment and continuation rules again.
- **TOML.** It is a different format that people sometimes call INI.
  [toml-nv](https://novo-lang.org/packages/toml-nv) is that package.
- **Any input or output.** Nothing here opens a file. The caller holds
  the text, and a renderer answers a string or appends to a buffer the
  caller owns.
- **A microcontroller build.** A document is a list of lists of
  strings, which is not what a microcontroller has.

## Related packages

- [config-core-nv](https://novo-lang.org/packages/config-core-nv) is
  the value tree and the precedence rules a layered configuration is
  made of. This package depends on it so that `inicfg.to_config` can
  answer that tree directly. It is `core` and has no dependencies of
  its own, so a program that only wanted to read an INI file pays for
  one package of arithmetic.
- [config-nv](https://novo-lang.org/packages/config-nv) stacks several
  sources of configuration in precedence order. `inicfg.to_config` is
  the whole of its INI adapter.
- [toml-nv](https://novo-lang.org/packages/toml-nv),
  [yaml-nv](https://novo-lang.org/packages/yaml-nv) and
  [dotenv-nv](https://novo-lang.org/packages/dotenv-nv) are the other
  configuration formats on the registry. TOML and YAML carry types and
  nesting; a `.env` file is a flat list of strings with shell quoting.
- [datafile-nv](https://novo-lang.org/packages/datafile-nv) answers
  which directory an application's INI file lives in on each platform.

## Tests

```bash
novo test --isolate tests/iniread_tests.nv   # 8 tests: the dialects and the parse
novo test --isolate tests/inicfg_tests.nv    # 8 tests: the config tree
```

The vectors are configparser's own documented examples: the
`ssh_config`-shaped file its page opens with, and its two interpolation
examples, one for `BasicInterpolation` and one for
`ExtendedInterpolation`. A reviewer can check them against that page
rather than against this package. `rust-ini`'s documented behaviour is
the reference for `iniread.rust_ini`.

The suite asserts that `a = b ;c` reads two ways under two dialects,
that configparser lowercases a key and this package's own dialect does
not, that all four duplicate policies do what they say, that a repeated
key becomes a list in the config tree, that a parsed document renders
back to its own text, that a reference chain resolves, and that a cycle
is reported as a cycle.

The tests compile today and fail at run, each on the `not implemented`
panic that is its body. That is the expected state of an interface
release. They turn green one at a time as bodies land.

## Implementation status

Nothing is implemented, apart from the three constants. Every function
here is declared with its signature and its effect row, and every body
is a `todo()`.

| Item | Implemented |
| --- | --- |
| `inidoc.DEFAULT_SECTION`, `inicfg.CONFIG_DEPTH`, `iniinterp.INTERPOLATION_DEPTH_LIMIT` | yes (they are constants) |
| `inierror.fault`, `.kind_name`, `.message` | no |
| `inidoc.empty`, `.empty_with` | no |
| `inidoc.section_names`, `.section`, `.has_section` | no |
| `inidoc.keys`, `.local_keys`, `.get`, `.get_local`, `.get_all` | no |
| `inidoc.entry`, `.has_key`, `.missing_key_fault` | no |
| `inidoc.set`, `.set_with_comment`, `.remove` | no |
| `inidoc.add_section`, `.remove_section`, `.rename_section`, `.overlay` | no |
| `iniread.defaults`, `.configparser`, `.rust_ini` | no |
| `iniread.parse`, `.read`, `.classify` | no |
| `iniread.is_section_name`, `.is_key` | no |
| `iniinterp.style_name`, `.value`, `.expand`, `.expand_document` | no |
| `iniinterp.references`, `.has_references`, `.escape`, `.check` | no |
| `iniwrite.standard`, `.compact`, `.windows` | no |
| `iniwrite.round_trip`, `.render`, `.render_into`, `.render_section` | no |
| `iniwrite.check`, `.rendered_len` | no |
| `inicfg.defaults_name`, `.to_config`, `.to_config_typed` | no |
| `inicfg.section_to_config`, `.from_config`, `.is_writable` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
