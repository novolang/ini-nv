# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.3 — 2026-09-25

Every field of `IniOptions` is now declared `var`.  Under novo 0.10.0 a
field is assigned only when it is declared that way, and the way to
choose a dialect is to take `iniread.defaults()` and set the fields that
differ.  This is a change to a public declaration, but no program that
built against 0.0.2 stops building.  Every body is still `todo()`.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md).

`IniFault` now declares the `impl Error` its own `Result` positions
require.  `Result<T, E>` has carried the bound `E: Error` since SPEC
§ 3.4, and the compiler enforced it only when `E` was declared in the
module that named it — so `Result<_, inierror.IniFault>` was accepted
across modules with no impl anywhere.  The impl is the signature this
package always meant; nothing else about the interface changed.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Six modules.  `iniread` is the dialect and the read, `inidoc` the
  document and the edits on it, `iniinterp` the two interpolation
  spellings, `iniwrite` the writer, `inicfg` the conversion to a
  `config-core-nv` tree, and `inierror` the one fault type.
- **The dialect is a value the caller picks**, because INI has no
  specification and two real implementations disagree on almost every
  line.  `defaults()`, `configparser()` and `rust_ini()` are three
  named answers, and every field of `IniOptions` is a place two
  libraries differ — most visibly the inline comment, which
  configparser does not strip and rust-ini does, so `a = b ;c` is two
  different values depending on who wrote the file.
- **The document keeps the trivia**, which is what makes
  `iniread.parse` then `iniwrite.round_trip` the identity on a file
  nobody changed: every comment, every blank-line grouping, each
  entry's own delimiter and each comment's own prefix.  A tool that
  changes one value hands a person a one-line diff.
- Editing is on the document rather than in the writer for the same
  reason: `set` leaves the comment that explains the value, and
  `remove` takes it, because a file holding an explanation of a setting
  nobody has is worse than one that lost both.
- **Interpolation is a step of its own**, not something a read does.
  configparser expands on the way out of `get`, so a program that reads
  a value and writes the file back destroys the references it never
  touched.  Here `get` answers raw text always and `iniinterp.value`
  expands.  Both spellings — `%(name)s` and `${section:name}` — with
  chains resolved, `%%` and `$$` as literals, and a cycle named as a
  cycle rather than as a depth limit, because to the person who typed it
  those are one problem.  `iniinterp.check` finds every dangling
  reference in one pass, for a start-up path that would rather not
  discover one on Tuesday.
- **Four duplicate policies**, because all four are somebody's default:
  refuse (configparser strict), first wins, last wins with sections
  merging, and keep-all — which is rust-ini's multi-value property and a
  systemd unit's repeated `After=`, and the only list INI has.
- `inicfg.to_config` is **the one call `config-nv`'s ini adapter
  makes**, and its three decisions are arguments rather than
  assumptions: the fallback section is NOT flattened by default,
  because copying a file's own defaults into every section would make
  them outrank a later layer's explicit setting; a section name is one
  key, dots included, because INI has no nesting and splitting would
  merge sections a reader can see are distinct; and every value is a
  string unless the caller asks for `config-core-nv`'s own
  `infer_scalar`, so that an INI value and an environment variable reach
  `get_int` the same way.
- **One dependency, and the direction is the affordable one.**
  `config-core-nv` is `core` and has none of its own, so a consumer who
  only wanted to read a file pays for one package of pure arithmetic.
  The reverse — the adapter living in `config-core-nv` — is what that
  package's manifest refuses, because it would put a parser in the
  closure of a package whose subject is precedence.
- **No device claim.**  There is no `tests/embedded_probe.nv`: the
  document is a list of lists of strings.

The test vectors are configparser's own documented examples — the
`ssh_config`-shaped file its page opens with, and the two interpolation
examples — so a reviewer can check the port against that page rather
than against this package.
