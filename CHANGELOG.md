# Changelog

All notable changes to changelog-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `clogdoc` — `ClogRelease` with an optional version, the six closed
  kinds, entries as byte ranges into the document, the link
  definitions, and the three edits: `promote`, `add_entry`, `set_link`.
- `clogconv` — the Conventional Commits parse with both spellings of a
  breaking change, and `ClogMap` as a value with `unmapped_types`
  beside it.
- `clogbump` — `bump_of` for the commits alone and `next_version` for
  the answer under the pre-1.0 rule, with `bump_at_least` for the
  breaks no commit message mentions.
- `clogrender` — `*_into` / `*` / `*_to<W: Write[e]>` for the document
  and for one release, the heading alone, and `release_from_commits`.
- `clogcheck` — sixteen findings with a severity each, and
  `check_for_publish`, the three questions a publish asks in one.
- `clogerr` — the six things that are not a changelog at all.

### Known

- **Unreleased is a release with no version**, which is what makes
  `promote` a two-field change and what lets the same check and the
  same renderer serve both.
- **The commit-type mapping is a value**, because neither standard
  defines one and a hard-coded mapping drops a project's own types
  silently; `unmapped_types` is the call that reports it before the
  changelog is written.
- **`next_version` takes the current version**, because below 1.0.0 a
  breaking change is a minor bump and a function over commits alone is
  answering half the question.
- **`check` cannot fail.** A non-conforming changelog is the answer,
  and `ClogSeverity` splits what a consumer will read wrongly from what
  is merely untidy — so a publish gate can be turned on for a project
  that already has a file.
- Three `core` dependencies, each for a type that crosses the boundary:
  semver-nv for the version, calendar-nv for the date, markdown-nv for
  what counts as a heading.
- **`@tier(embedded)` is not claimed**, and the README says why.
- **`novo pkg publish` does not enforce a changelog rule today.** The
  README names the finding and the call that would.
- **An open codegen defect was hit and not designed around.** Comparing
  a `?ClogRelease` against `None` emits a phi LLVM refuses, because the
  payload carries optionals of its own; the type stayed as it is, the
  suite uses the filing's documented `match` workaround, and the
  sighting went on the filing.
- The scaffold's `src/changelog.nv` was dropped for six prefixed
  modules.

### Design notes

- `docs/publishing.md` has no Changelog section. The rules are spread
  across "What ships" (the allow-list) and "Choosing the version"
  (record the answer), and neither is checked at publish time.
  `clogcheck.check_for_publish(doc, cutting)` is the call that would
  check all three: the document conforms, the Unreleased section
  carries something, and the version being cut is greater than every
  version already in the document.
- `docs/releases.md` is a changelog in a different shape: prose
  bullets under `##` headings rather than six kinds under `###` ones.
  Adopting this package for it means either moving that file to the
  standard's shape or growing a second document profile here.
- The clock is the one tension with the `core` layer, and it resolves
  the way sarif-nv's did: `promote` takes the date as a parameter.
  calendar-nv would not have helped, because it has no clock either.
  The dependency is for the type and the comparison, not for "now".
