# ini-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

INI, read the way the tool that wrote it meant it.

There is no INI specification, and that is the whole design problem.
`a = b ;c` is a value of `b ;c` to Python's `configparser` and a value
of `b` to Rust's `rust-ini`.  `[p]` twice is an error, a merge, or a
second section depending on whom you ask.  `User` is a key called
`user` to configparser and `User` to everyone else.  A package that
picked one set of answers would be right about a third of the files it
met.

So the dialect is a **value the caller picks**, and three named
constructors are the three answers somebody already depends on:
`iniread.defaults()`, `iniread.configparser()`, `iniread.rust_ini()`.
A caller reading a file written by a particular tool names that tool and
gets that tool's reading.

Six modules.

| surface | module | reach for it when |
| --- | --- | --- |
| the **dialect and the read** | `iniread` | you are turning text into a document |
| the **document** | `inidoc` | you are reading settings, or changing them |
| the **expansion** | `iniinterp` | the file uses `%(name)s` or `${section:name}` |
| the **writer** | `iniwrite` | you are writing the file back out |
| the **config tree** | `inicfg` | you are layering this file with others |
| the **faults** | `inierror` | you are reporting what was wrong with somebody's file |

## Adding it, and checking it

```bash
novo pkg add ini-nv            # into your novo.toml
novo pkg build                 # type- and effect-check the package
novo test --isolate tests/iniread_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: ini-nv.<module>.<fn>`.  They turn
green one at a time as bodies land.

## The one example that will work

```novo
use inidoc
use iniread

fn main() [io]
    match iniread.read("[web]\nport = 8080\nhost = localhost\n")
        Err(f)  => println("line ${f.line}")
        Ok(doc) => println(inidoc.get(doc, "web", "port") ?? "unset")
    // 8080
```

## The load-bearing interface

`inidoc.IniDocument` — and specifically the fact that it **keeps the
trivia**.

An INI file is almost always a file a person wrote and will read again.
It has comments explaining why a timeout is 45, blank lines grouping
related keys, and an order that means something.  So a section holds its
entries in order, an entry holds the comments written above it and the
inline comment written after it and its own delimiter, and

```novo norun:pseudo
iniread.parse(text, options)  |>  iniwrite.round_trip
```

is the identity on a file nobody changed.  A tool that reads somebody's
`~/.config/app.ini`, changes a port and writes it back hands them a
one-line diff.

Everything else follows from that.  `inidoc.set` is in the document
module rather than in the writer, because it changes a value and leaves
the comment that explains it.  `inidoc.remove` takes the comment WITH
the key, because a file left holding an explanation of a setting nobody
has is worse than one that lost both.  And the cost of the whole shape —
a lookup is a scan rather than a hash probe — is not a cost on a file of
forty keys.

## Interpolation is a step, not something a read does

configparser expands `%(name)s` on the way **out of `get`**.  So a
program that reads a value, changes something unrelated and writes the
file back has silently replaced every reference with whatever it
resolved to that day.

Here `inidoc.get` answers the **raw** text, always, and a caller that
wants the expansion calls `iniinterp.value`.  Both spellings are
supported because configparser has both:

| style | spelling | scope |
| --- | --- | --- |
| `IniInterpBasic` | `%(name)s` | this section, then the fallback |
| `IniInterpExtended` | `${name}`, `${section:name}` | anywhere in the document |
| `IniInterpNone` | — | `%` and `$` are ordinary characters |

`IniInterpNone` is what a caller reading a systemd unit or a
`.gitconfig` wants: neither format has interpolation, and both use `%`
and `$` for something else.

A chain resolves — configparser's own example has `my_pictures`
pointing at `my_dir` pointing at `home_dir` — and a **cycle is named as
one**.  `a = %(b)s` with `b = %(a)s` answers `IniInterpolationCycle` at
the value that closed the loop, not "depth limit exceeded", because to
the person who typed it those are one problem with one fix.
`iniinterp.check` finds every dangling reference in a document in one
pass, which is what a start-up path should run before a setting nobody
reads until Tuesday fails then.

## The four duplicate policies

All four are somebody's default, so all four are here, per document:

| policy | who does this |
| --- | --- |
| `IniDupRefuse` | configparser with `strict=True` — its default, and ours |
| `IniDupFirstWins` | most hand-written C parsers |
| `IniDupLastWins` | configparser with `strict=False`; a repeated section merges |
| `IniDupKeepAll` | rust-ini's multi-value properties; a systemd unit's repeated `After=` |

`inidoc.get_all` is what reads the last one, and it is the reason a
repeated key becomes a `CfgList` on the way into the config tree — a
repeated key really is a list, and it is the only list INI has.

## The config tree, and the three decisions it makes

`inicfg.to_config` is the one call `config-nv`'s ini adapter makes.  It
answers a `config-core-nv` `ConfigValue`: a table of sections, each a
table of keys.  Three of its decisions could reasonably have gone the
other way, so all three are arguments or named in the signature.

**The fallback is not flattened by default.**  configparser's
`[DEFAULT]` supplies a value to every section that lacks one.  Copying
those values into each section would make a file's own fallback outrank
a *later layer's* explicit setting once `config-core-nv` merges — a
default from `/etc` beating a value from the environment, which is
exactly backwards.  So `IniDefaultsAsSection` is the default;
`IniDefaultsFlattened` is there for a caller reading one file and
layering nothing.

**A section name is one key, dots included.**  `[a.b]` becomes the key
`"a.b"`, not a table `a` holding a table `b`.  INI has no nesting, two
tools that both write `[a.b]` mean different things by it, and a tree
built by splitting would silently merge sections a reader can see are
distinct.

**Every value is a string unless the caller asks otherwise.**  INI has
no types.  `to_config` answers `CfgStr` throughout; `to_config_typed`
runs each value through `config-core-nv`'s own `cfgvalue.infer_scalar`
— its rule and not a second one, which is the point: a port number that
arrives from an INI file and one that arrives from the environment then
reach `get_int` the same way.

## Why this depends on config-core-nv, and why that is affordable

`config-core-nv` is `core` and has **no dependencies of its own**, so a
consumer who only wanted to read an INI file pays for one package of
pure arithmetic.  The reverse direction is what that package's own
manifest refuses: an adapter living there would put a parser in the
closure of a package whose subject is precedence.

What it buys is that `config-nv` gains its third format by calling one
function rather than by carrying a second copy of the mapping rules
above — and that the rules live next to the parser whose output they
describe.

## What is deliberately not here

**Type inference of its own.**  Every value is a string.
`config-core-nv.cfgvalue.infer_scalar` is where a string becomes typed,
and using its rule rather than a second one is why an INI value and an
environment variable behave the same downstream.

**Nesting.**  `[a.b]` is a section whose name contains a dot.  See
above.

**Encodings.**  novo-lang's `Str` is UTF-8, so this reads UTF-8.  A
caller holding other bytes decodes them first.

**`.gitconfig`'s subsections** — `[remote "origin"]` — and **TOML.**
The first is a real dialect this package does not have; `iniread.classify`
is public so that a caller can write the fifteen lines that do, without
re-deriving the comment and continuation rules.  The second is a
different format that people call INI, and it is toml-nv.

## The layer, and why

`core`.  A line scan over text the caller already holds, a document of
strings, and a renderer that answers a string or appends to the caller's
buffer.  No function declares an effect: the file is `config-nv`'s to
open, and this package never learns where it came from.

**No device claim.**  There is no `tests/embedded_probe.nv`: the
document is a list of lists of strings, which is not what a
microcontroller has.

## The reference implementation

Python's `configparser` (PSF) and Rust's `rust-ini` (MIT), both named
rather than blended — `iniread.configparser()` and
`iniread.rust_ini()` are each meant to answer what that library answers.
The test vectors are configparser's own documented examples: the
`ssh_config`-shaped file its page opens with, and the two interpolation
examples for `BasicInterpolation` and `ExtendedInterpolation`.  A
reviewer can check them against that page rather than against this
package.

## What depends on this

`config-nv`, for its third format — `inicfg.to_config` is the whole of
its ini adapter.  The row on the grid says so: *"the third format
config-nv layers"*.

## Status

| function | implemented |
| --- | --- |
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
