# changelog-nv

[Keep a Changelog 1.1](https://keepachangelog.com/en/1.1.0/) is a convention for
writing a project's `CHANGELOG.md`, "a curated, chronologically ordered list of
notable changes for each version of a project".
[Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) is a
convention for writing a commit subject line so that a tool can read it. This
package brings both to novo-lang, and joins them: a changelog is read into
releases, a commit subject is read into its parts, and the commits of a range
become the next version and the release section that announces it.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A **changelog** is a file at the root of a project that records what changed in
each released version. Keep a Changelog fixes its shape. The file opens with a
`# Changelog` title and a paragraph of boilerplate. Under that comes one `##`
heading per version, newest first, each carrying the version in brackets and the
date it was released: `## [1.2.3] - 2026-09-11`. Under each version heading come
`###` headings, one per **kind** of change. There are exactly six kinds: Added,
Changed, Deprecated, Removed, Fixed and Security. Under each kind is a list of
bullets, one per change.

The topmost `##` heading is the **Unreleased section**, written `##
[Unreleased]`. It collects the changes that have been made but not yet
released. Cutting a release means giving that section a version and a date.

A **conventional commit** is a commit whose subject line has the shape
`<type>[(<scope>)][!]: <description>`. The type is a noun such as `feat` or
`fix`. The optional scope names the part of the project the change touched. The
optional `!` marks a breaking change. A commit message may also carry
**footers** after its body, of the shape `Token: value` or `Token #value`, and
one of those tokens is `BREAKING CHANGE`.

The two conventions do not know about each other. Conventional Commits has an
open type vocabulary and names only `feat` and `fix`. Keep a Changelog has six
closed kinds. Nothing in either document says which type goes in which kind, so
this package takes that mapping as a value the caller supplies.

Every value this package produces is read out of a string the caller already
holds. No file is opened, no repository is cloned and no commit is fetched. The
commits arrive as values.

## Install

```
novo pkg add changelog-nv
```

## Example

```novo
use std.list
use semver
use civil
use clogdoc
use clogconv
use clogbump
use clogcheck
use clogrender

// The bytes a release tool has just read from CHANGELOG.md.
fn source() -> Str
    "# Changelog\n\n## [Unreleased]\n\n### Added\n\n- a new flag\n\n## [0.4.2] - 2026-08-01\n\n### Fixed\n\n- a thing\n"

fn main() [io]
    match clogdoc.parse(source())
        Err(_) => println("the text is not a changelog")
        Ok(d)  =>
            // Every rule of the standard this document breaks. Never an error.
            println("${list.len(clogcheck.check(d))} findings")

            // The commits in the range being released, one parsed subject each.
            let commits = [clogconv.parse_subject("feat!: the old flag is gone")]

            // The newest version already in the file, and the one to cut next.
            let current = clogdoc.latest_version(d) ?? Version { major: 0, minor: 0, patch: 0, pre: "", build: "" }
            let next = clogbump.next_version(current, commits, clogconv.default_map())
            println("${next.major}.${next.minor}.${next.patch}")

            // Give the Unreleased section that version and the caller's date.
            match civil.date(2026, 9, 15)
                Err(_)   => println("not a date")
                Ok(when) =>
                    match clogdoc.promote(d, next, when)
                        Err(_)  => println("there is nothing to promote")
                        Ok(cut) => println(clogrender.document(cut))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `clogdoc` | The document: a parsed `CHANGELOG.md`, its releases, its six kinds, its entries and its link definitions, and the three calls that edit one. |
| `clogconv` | Conventional Commits: a subject or a whole message read into type, scope, breaking flag, description and footers, and the map from a commit type to a changelog kind. |
| `clogbump` | The next version: what a range of commits implies, and what that becomes once the rule for versions below 1.0.0 is applied. |
| `clogrender` | Writing a document or one release back out in the standard's shape, into a buffer, into a string, or into a sink the caller supplies. |
| `clogcheck` | Checking a document against the standard. Answers a list of findings, each with a severity, a message and a byte offset. |
| `clogerr` | The error type. Every case is either text that cannot be read as a document at all, or an edit the document cannot accept. |

## How to choose an entry point

**A tool that reads or edits an existing file starts at `clogdoc`.** Call
`parse` on the text, read the releases and entries out of the result, and call
`promote`, `add_entry` or `set_link` to change it.

**A tool that derives a release from git history starts at `clogconv`.** Parse
each commit subject, then hand the list to `clogbump.next_version` for the
version and to `clogrender.release_from_commits` for the section.

**A gate that only says yes or no starts at `clogcheck`.** Use
`is_conforming` for a document on its own and `ready_to_publish` for a document
plus the version about to be cut.

`clogrender` offers each output in three shapes. `*_into(out, off, …)` appends
into a buffer the caller sized and hands it back; it is the primitive, and a
tool writing a hundred releases into one page allocates once. `*(…) -> Str`
returns a fresh string. `*_to(w, …)` writes into any sink implementing `Write`,
so a file costs the file's effects and an in-memory buffer costs none.

## The rules a user needs

1. **The Unreleased section is a release with no version.**
   `ClogRelease.version` is an optional. `None` means the section is the
   Unreleased one. The same checker, the same renderer and the same accessors
   serve both, and `clogdoc.is_unreleased` is the question. Keep a Changelog
   1.1, "What makes a good changelog?".
2. **The six kinds are closed.** `ClogKind` has exactly the six headings the
   standard names. A document with a `### Performance` section is a `clogcheck`
   finding and not a seventh kind, so a reader can tell a typo from a decision.
   Keep a Changelog 1.1, "Types of changes".
3. **`clogdoc.parse` refuses only text it cannot walk as markdown.** A missing
   title, a release with no date, sections in the wrong order and a version that
   does not parse are all `clogcheck` findings about a document that was read.
   Point the package at a real project's file and it answers.
4. **`clogcheck.check` cannot fail.** It answers a list of every finding, not
   the first one. Gate on `must_fix`, which is the findings a consumer will read
   wrongly: an unknown `###` heading, a duplicate kind, releases that are not
   newest-first. `should_fix` findings are documents that are merely untidy.
5. **`promote` takes the date as an argument.** This package has no clock. It
   refuses a document with no Unreleased section, and one whose Unreleased
   section holds no entries. Keep a Changelog 1.1, "What makes a good
   changelog?".
6. **The commit-type mapping is a value you supply.** `clogconv.default_map()`
   is a starting point, and it hides `docs`, which is right for a library and
   wrong for a documentation project. A type the map says nothing about is
   dropped from the changelog. Run `clogconv.unmapped_types` over your own
   history before you write one. Conventional Commits 1.0.0, specification item
   16.
7. **A breaking change has two spellings.** A `!` before the colon in the
   subject and a `BREAKING CHANGE:` footer mean the same thing. Call
   `clogconv.is_breaking`, which reads both. Conventional Commits 1.0.0,
   specification items 12 and 13.
8. **`BREAKING CHANGE` is the one footer token containing a space.** Every
   other token replaces whitespace with `-`. Conventional Commits 1.0.0,
   specification item 9.
9. **`clogbump.next_version` takes the current version, not only the commits.**
   Below 1.0.0 a breaking change is a minor bump and everything else is a patch.
   Above it, a breaking change is a major bump, a `feat` a minor and a `fix` a
   patch, and the largest wins. Semantic Versioning 2.0.0, item 4, and
   novo-lang's own registry rule for `0.x` releases.
10. **Some breaking changes are in no commit message.** Raising the toolchain
    version a package needs breaks a consumer with every signature identical.
    `clogbump.bump_at_least` is how a caller raises an answer this package
    cannot derive.
11. **An entry's text is a byte range into the source.** `clogdoc.entry_text`
    turns one into a string. Nothing reads inside it, so an entry's inline code,
    links and markup survive a parse and a render unchanged.
12. **Rendering reorders sections into the standard's order.** That is the one
    place this package changes the order of anything, and it happens only when
    a section is written from scratch. `clogdoc.parse` keeps the document's own
    order so that `clogcheck` can report it.

## What is not included

- **Reading a repository.** No commit is fetched and no `.git` directory is
  opened. The commits arrive as values, which is what makes the package
  testable against a fixture rather than against a clone.
- **A clock.** `promote` stamps the date it is given. A date the library
  invented would be one nobody could reproduce.
- **git-cliff's template language.** A changelog's shape is the standard's. A
  project that wants another shape reads the entries out with
  `clogdoc.entry_texts` and writes its own.
- **A profile for documents in another shape.** This package models Keep a
  Changelog. A release file that uses `##` per release with prose bullets is a
  different format.
- **Running on a microcontroller.** Every value here is a list of sections over
  a document held in memory, and no device reads a changelog.
- **A second parser for markdown.** Deciding what counts as a heading is
  CommonMark's business, so the walk is markdown-nv's. A heading written as text
  over a row of dashes is a heading. A `###` inside a fenced code block is not.

## Related packages

- [semver-nv](https://novo-lang.org/packages/semver-nv) is the version type this
  package reads out of a release heading and produces from a bump. Comparing two
  of its values is how `clogcheck` finds releases that are out of order.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is the date type. A
  release heading carries an ISO 8601 calendar date, and `CivilDate` is that
  value.
- [markdown-nv](https://novo-lang.org/packages/markdown-nv) is the CommonMark
  parser the document walk is built on. `clogdoc.parse_with` takes its options,
  so the nesting bound and the extension set are the caller's.
- [toml-edit-nv](https://novo-lang.org/packages/toml-edit-nv) does for a
  `novo.toml` what this package does for a `CHANGELOG.md`: it edits a file in
  place and keeps everything it did not change.
- `std.markdown` in the standard library renders markdown to HTML. It has no
  model of a changelog and no notion of a release.

## Tests

```bash
novo test tests                          # every suite
novo test tests/clogdoc_tests.nv         # the document, and promoting one
novo test tests/clogconv_tests.nv        # the commit parse and the mapping
novo test tests/clogbump_tests.nv        # the next version
novo test tests/clogcheck_tests.nv       # the findings and their severities
```

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
changelog-nv.<module>.<fn>` panic, because every body is a `todo()`. The tests
are the specification the implementation will have to satisfy.

The document fixtures come from the `spec/` directory of the `keep-a-changelog`
Ruby gem, which implements the standard. The commit fixtures are the
Conventional Commits specification's own examples, including the two that
regular-expression implementations get wrong: a `BREAKING CHANGE:` footer token
containing a space, and a body paragraph whose last line contains a colon and is
not a footer.

The suite asserts that the same breaking commit takes `1.4.2` to `2.0.0` and
`0.4.2` to `0.5.0`, that `clogconv.unmapped_types` names a type the map is
silent about, that `clogcheck.check` answers a list rather than an error, and
that `promote` dates the Unreleased section and leaves every other byte alone.

## Implementation status

| Item | Implemented |
| --- | --- |
| `clogdoc.ClogKind`, `.ClogEntry`, `.ClogSection`, `.ClogRelease`, `.ClogDoc`, `.ClogLink` | declared |
| `clogdoc.parse`, `.parse_with`, `.position_of`, `.empty_doc` | no |
| `clogdoc`'s fourteen readers, from `unreleased` to `link_of` | no |
| `clogdoc.kind_name`, `.kind_from_name`, `.kinds`, `.kind_order` | no |
| `clogdoc.promote`, `.add_entry`, `.set_link` | no |
| `clogconv.ClogCommit`, `.ClogFooter`, `.ClogRule`, `.ClogMap` | declared |
| `clogconv.parse_subject`, `.parse_message`, `.is_conventional` | no |
| `clogconv.is_breaking`, `.breaking_description`, `.footer`, `.footers_of`, `.type_ok` | no |
| `clogconv.subject_of`, `.entry_of` | no |
| `clogconv.default_map`, `.with_rule`, `.kind_of`, `.has_rule`, `.unmapped_types`, `.commits_for` | no |
| `clogbump.ClogBump` | declared |
| `clogbump`'s ten functions, from `bump_of` to `follows` | no |
| `clogrender`'s sixteen functions, from `release_into` to `needs_escape` | no |
| `clogcheck.ClogSeverity`, `.ClogIssue` | declared |
| `clogcheck`'s eleven functions, from `check` to `ready_to_publish` | no |
| `clogerr.ClogError` and `impl Error for ClogError` | declared |
| `clogerr.error_at`, `.is_parse_error`, `.render`, `.default_depth` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
