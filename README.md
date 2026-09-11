# changelog-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Keep a Changelog 1.1 and Conventional Commits, together: a
`CHANGELOG.md` read into releases, a commit subject read into its
parts, the next version derived from a range of commits, a release
section rendered in the standard shape, and a document checked against
the standard with reasons.

- `clogdoc` — the document, and the six kinds;
- `clogconv` — Conventional Commits, and the mapping neither standard
  defines;
- `clogbump` — the next version, and the rule below 1.0 that changes it;
- `clogrender` — the standard shape, into the caller's buffer;
- `clogcheck` — a document against the standard, with reasons;
- `clogerr` — the short list of things that are not a changelog at all.

```
novo pkg add changelog-nv
novo pkg build
novo test
```

## The one example that will work

This is a publish.

```novo ignore
use clogdoc
use clogcheck
use clogrender

fn cut(text: Str, v: Version, when: CivilDate) -> Result<Str, ClogError>
    let d = clogdoc.parse(text)!
    let d2 = clogdoc.promote(d, v, when)!
    Ok(clogrender.document(d2))
```

`promote` refuses a document with no Unreleased section, and one whose
Unreleased section is empty — because a version whose notes nobody
wrote is what this package exists to make hard to publish by accident.

## The load-bearing interface: Unreleased is a release with no version

```novo ignore
pub struct ClogRelease
    version: ?Version
    date: ?CivilDate
    yanked: Bool
    sections: [ClogSection]
    …
```

Keep a Changelog says a document opens with an `## [Unreleased]`
section, and that a release is cut by giving that section a version and
a date. Almost every tool models the two as different things — a list
of pending lines, and then a list of past releases — and pays for it
three times:

- **`clogcheck.check` runs the same rules over it.** A `### Fixed` with
  no entries under it is the same fault whether or not the section has
  shipped, and a tool whose pending section is a different type cannot
  check it until it is too late to fix cheaply.
- **`clogrender.release_into` writes it in the same shape**, so what a
  person reads in the pending section is what they will read in the
  released one.
- **`promote(doc, version, date)` is a two-field change.** That is the
  publish. A design with two types has to re-serialise the file to cut
  a release, which is how a changelog loses its own formatting on the
  day it matters most.

The date is the **caller's**, because a `core` package has no clock and
a changelog stamped with a date the library invented is one nobody can
reproduce. Same rule as tar-nv's and zip-nv's headers.

## The second decision worth arguing: the commit-type mapping is a value

Conventional Commits has an **open** type vocabulary — `feat` and `fix`
are the only two it names. Keep a Changelog has exactly **six** kinds.
Nothing anywhere defines a mapping between them.

git-cliff puts one in a configuration file for exactly that reason, and
a library that hard-coded one silently drops a project's `perf:`
commits from its changelog: not an error, not a warning, just a section
that never appears.

So `ClogMap` is a value the caller supplies, and three things follow:

- `default_map()` is a defensible starting point with its choices
  argued in place. Two are worth naming here: **`docs` is hidden**,
  which is right for a library and wrong for a documentation project,
  and is the first line most callers will change; and **`revert` →
  Removed** is wrong when the thing reverted was itself a removal, and
  no mapping can know that.
- `ClogRule.hidden` is a field rather than an absence from the list,
  because "this type is deliberately not in the changelog" and "nobody
  has decided about this type" are different states.
- **`unmapped_types(map, commits)`** is the call that keeps the silent
  drop from being silent: a project that runs it over its own history
  is told which of its types are about to vanish, before the changelog
  is written.

## The third: the next version needs the current one

```novo ignore
pub fn next_version(current: Version, commits: [ClogCommit], m: ClogMap) -> Version
```

Over a range of commits: any breaking change is a major bump, any
`feat` a minor, any `fix` a patch, the largest wins. **And then the 0.x
rule changes it** — below 1.0.0 a breaking change is a *minor* bump and
everything else is a patch, because `0.x` is where a package says it is
still finding its shape and a project that let Conventional Commits
drive `major` would have reached 1.0.0 in its third week.

A function that answered a bump from the commits alone would be
answering half the question, and the half it left out is the half that
differs between every pre-1.0 package on a registry and every post-1.0
one. `bump_of` is the commit half on its own, for a caller that wants
to report "these commits contain a breaking change" without a version
in hand.

**What no commit message can say**, and `bump_at_least` is the way out:
`docs/publishing.md` § Choosing the version makes "breaking" a test
about whether a program that used the old version still compiles and
behaves correctly — so raising the toolchain floor is a break with
every signature identical. `clogbump` cannot see that, says so, and
takes the correction.

## `check` cannot fail

A changelog that does not follow the standard is the **answer** — a
list of findings a person acts on — and modelling it as an error would
make the ordinary case, a real project's real file, travel the failure
path. schema-nv and openapi-nv make the same split for the same reason.

So `clogerr.ClogError` is deliberately a short list: two cases where
there is nothing to hand back (a document the markdown walker refuses,
one that nests past the bound) and four that belong to an *edit* rather
than a read. Everything else — a missing title, a release with no date,
sections in the wrong order, a version that does not parse — is a
`ClogIssue`.

`ClogSeverity` splits those again. `ClogMustFix` is a document a
consumer will read **wrongly**: an unknown `###` kind that a grouping
consumer drops, a duplicate kind where "the Added section" is
ambiguous, releases that are not newest-first when every consumer takes
the first as the latest. `ClogShouldFix` is a document that is merely
not in the standard's shape. A publish gate that refused on the second
class could not be turned on for an existing project; one that refuses
on `must_fix` can.

## Dependencies, and why each one is a type rather than a convenience

**semver-nv `^0.1.4`** — the version. A release heading carries one, a
bump produces one, and comparing two is how `check` finds a changelog
whose releases are out of order. Answering a `Str` would make every
consumer parse it again, and `novo pkg publish`'s own monotonicity rule
is exactly this comparison.

**calendar-nv `^0.0.2`** — the date. Keep a Changelog's heading is
`## [1.2.3] - 2026-09-11`, an ISO 8601 calendar date, and `CivilDate`
is that value. A `Str` would have made "is this changelog in
chronological order" a string comparison that happens to work for
four-digit years.

**markdown-nv `^0.0.1`** — the headings. A changelog *is* markdown, and
deciding what is a heading is CommonMark's business: a setext heading
(`Added` over a row of dashes) is one, a `###` inside a fenced code
block is not, and a regular expression over lines gets both wrong.
git-cliff's parser is that regular expression, and its failure is
silent — a section simply does not appear. `clogdoc.parse_with` takes
markdown-nv's own `MdOptions`, so the nesting bound and the extension
set are the caller's.

## The layer, and why

`core`. A changelog is a string the caller already holds and a commit
subject is another one; parsing either is arithmetic over bytes, and
rendering writes into a buffer the caller owns. Nothing is read, no
repository is opened — **the commits arrive as values**, which is what
keeps the package `[]` and what makes it testable against a fixture
instead of against a clone.

The one place a stream could have entered is writing a document out:

```novo ignore
pub fn document_to<W: Write[e]>(w: W, d: ClogDoc) -> ?IoError [e]
```

The clause is `[e]`, bound by the caller's `Write` impl, so a file
costs `[fs]` and an in-memory buffer costs nothing. `*_into(out, off,
…)` is the primitive under all three shapes, so a tool rendering a
hundred releases into one page allocates once.

## `@tier(embedded)` is not claimed

Deliberately. Every value here is a list of lists of byte ranges over a
document, and there is no device that reads a changelog. A claim would
be one the probe could only keep by never allocating in a package whose
whole job is building sections.

## The reference implementations

Two, and they disagree, which is why both are named.

`keep-a-changelog` (the Ruby gem and the specification it implements)
supplies the document model: the six kinds, the Unreleased section, the
`[YANKED]` marker, the link-reference headings, and the ordering rules.
Its own `spec/` fixtures are the parse vectors.

`git-cliff` supplies the commit half: the Conventional Commits parse,
the type-to-section mapping as configuration, and the derived bump.
What is *not* ported is its template language — a changelog's shape is
the standard's, and a project that wants another shape has
`clogdoc.entry_texts` and its own writer.

The Conventional Commits vectors are the specification's own examples,
including the two the regular-expression implementations get wrong:
`BREAKING CHANGE:` as a footer token containing a space, and a body
paragraph whose last line contains a colon and is not a footer.

## The consumers, and what adopting this would take

**`novo pkg publish`** is the first, and this lane found that the rule
it is supposed to enforce **is not enforced today**. `docs/publishing.md`
says a `CHANGELOG.md` is on the tarball's allow-list (§ What ships) and
that the version decision is recorded in one (§ Choosing the version),
and nothing checks either: a publish of a package whose changelog has
no entry for the version being cut succeeds, and the consumer deciding
whether to upgrade reads a file that does not mention the release.

The call that would change it is
`clogcheck.check_for_publish(doc, cutting)`, which asks the three
questions a publish needs in one — the document conforms, the
Unreleased section carries something, and the version being cut is
greater than every version already in the document. The third is the
registry's own `MONOTONIC` refusal asked one step earlier, where the
answer is still cheap to act on: today it arrives as an HTTP refusal
after the tarball has been built.

**The monorepo's own release cadence** is the second. `docs/releases.md`
opens with an `## Unreleased` section that accumulates one line per
change until a cut is called, and `docs/releases/unreleased.md` holds
the detail. That is Keep a Changelog's discipline in a file that is not
a `CHANGELOG.md`, and `clogdoc.promote` is what cutting it would be —
but the shape differs (`##` per release, prose bullets rather than six
kinds), so adopting it means either the file moves to the standard's
shape or this package grows a second document profile. Naming the
choice is this lane's job; making it is not.

**A commit hook** — `clogconv.parse_subject` plus `clogconv.type_ok` —
is the smallest consumer and the one that pays for itself first.

**`novo pkg publish --dry-run`'s facets line** already prints
`stability`, effects, tiers and the shard verdict. `clogcheck.check`'s
findings belong beside them, because the dry run is the moment an
author is looking.

## What a row wanted to widen

Nothing widened. Every function in this package is `[]` except
`document_to`, whose row is its caller's.

Four findings:

1. **`docs/publishing.md` has no § Changelog.** The brief names one.
   The rules are real but are spread across § What ships (the
   allow-list) and § Choosing the version (record the answer), and
   neither is checked at publish time. The section and the check are
   both worth having, and § The consumers above names the call.
2. **The clock is the one tension with `core`, and it resolves the same
   way sarif-nv's did.** `promote` stamps a date; a `core` package has
   none; the date is a parameter. calendar-nv would not have helped —
   it has no clock either — so the dependency is for the TYPE and the
   comparison, not for "now".
3. **`docs/releases.md` is a changelog in a different shape.** The
   project's own Unreleased discipline is prose bullets under `##`
   headings, not six kinds under `###` ones. This package models the
   standard; whether the project's file moves toward it is a decision
   for whoever owns the release cadence.
4. **A commit type nobody mapped is the silent failure of every tool in
   this space**, and it is the one thing a library can fix without
   choosing a mapping for its callers. `unmapped_types` is that, and it
   is the function most likely to justify the package on the day
   somebody adopts it.

## A toolchain defect this lane hit, and did not design around

Comparing a `?ClogRelease` against `None` reaches an open codegen
defect: an optional comparison whose payload comparison contains
another optional comparison emits a phi LLVM refuses to verify, and
`ClogRelease` carries `version: ?Version` and `date: ?CivilDate`
because the Unreleased section is a release with neither.

**The type was not changed to avoid it.** The optional version is the
load-bearing interface of the package, and bending it around a compiler
bug would have bought a worse design for a defect that is already filed
and open. The API suite spells that one assertion as a `match`, which
is the filing's own documented workaround and the same assertion, with
a comment at the site saying why.

This is the fourth unrelated package to reach it from the same ordinary
shape — a lookup that may miss, whose result has a field that may be
absent — and the sighting added two facts to the filing: the inequality
(`!= None`) lowers through the same path as the equality, and the
payload's optional fields being types from *other packages* does not
change the edge.

## The surface

| module | `pub fn` | `pub struct` | `pub enum` |
| --- | --- | --- | --- |
| `clogdoc` | 25 | 5 | 1 |
| `clogconv` | 16 | 4 | 0 |
| `clogbump` | 10 | 0 | 1 |
| `clogrender` | 16 | 0 | 0 |
| `clogcheck` | 11 | 0 | 2 |
| `clogerr` | 4 | 0 | 1 |
| **total** | **82** | **9** | **5** (34 variants) |

One `impl Error` block, for `ClogError`.
