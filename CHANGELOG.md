# Changelog

All notable changes to graphql-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-03

### Changed

- Internal: ladders rewritten as match. The escape reader, the lexer's string scanner, the executor's dispatch and the SDL reader read as one match on the byte or the word, a failed step is propagated with `!`, and the private hex-digit readers answer an optional. No public signature or behaviour changed.

This release requires novo 0.23.0 or later.

## 0.1.0 — 2026-10-12

The **implementation**.  Every one of the 112 `todo()` bodies the
interface release published is written, every suite is green, and no
signature changed: this release fills in bodies and moves `stability`
from `draft` to `experimental`.

The measured count is 112 — 15 in `gqlcheck`, 13 in `gqlerr`, 14 in
`gqlexec`, 18 in `gqlhttp`, 13 in `gqlintro`, 22 in `gqllang` and 17 in
`gqlsdl`.  The task that commissioned this work said 119, and that
number is not in the tree: `grep -c 'todo("' src/*.nv` answers 112
against the stub, and `grep -c 'todo()'` matches one line per file —
the module header's sentence — which is where a wrong total comes
from.

The suites are nine files under `tests/` and 200 `@test` functions
between them, all green: `novo test tests` answers `9 module(s) under
tests — 9 passed, 0 failed`, `novo test --isolate` answers the same for
each file on its own, and no run leaks.  Every line under `src/` is
covered by one of them or excused by a `cov: skip` marker: the publish's
own dry run measures 98% of 3082 instrumented lines under `src/`, over
the nine suites, with 39 regions excused.  The lines no test can reach
are the guards and fall-throughs a caller has already established — a
match's own completeness, a built-in table whose names are all answered
above, the two printer keywords a `SCALAR`/`UNION` answer returns
before, and the subscription root `execute` refuses before it can ask
for one — and each carries the marker and its reason in place.

### Added

- The bodies, module by module.  `gqlerr` and `gqllang`'s and
  `gqlsdl`'s parsers, the `gqlcheck` rule set (each rule answering the
  specification's own section number), `gqlintro`'s introspection
  document and its policy, `gqlexec`'s executor and `gqlhttp`'s
  transport.
- `gqlexec` executes: `complete` walks the selection set and mirrors
  its shape, aliases key the response by response key, `__schema`,
  `__type` and `__typename` are answered from the schema by projection
  (with fragments and `@skip`/`@include`), and § 6.4.4's propagation is
  a typed walk — `LandedNull` rides up to the nearest nullable field
  and `null_lands_at` names that field from the code.
- `tests/graphql_nv_edge_tests.nv` — the four shapes a shallow test
  never sees: a three-level selection pinned as ONE string (a flattened
  response passes a value-only test), `{ a: name b: name }` pinned with
  both keys, the propagation of a null on a two-level non-null chain
  with the error path asserted, and the three § 5 refusals — unknown
  field, wrong argument type, subfield of a scalar — each pinned to its
  LINE AND COLUMN together with the assertion that the request is an
  `Err` before any resolver runs.
- Three cases added to `tests/gqlhttp_tests.nv`, which the table there
  does not reach: a response carrying errors and no `data` member is
  the modern media type's 400, a `GqlSchemaInvalid` fault is a 500
  under both media types (it is the deployment's fault and not the
  client's, which is why `gqlerr.is_configuration_fault` exists), and
  the legacy media type's fault body carries the `"data": null` its
  clients were read by, where the modern one must not carry `data` at
  all.
- `tests/graphql_nv_impl_tests.nv` — 50 tests, one section per module,
  written to the coverage rule rather than to the design: each
  `tests/<module>_tests.nv` states what a module PROMISES, and what the
  oracle suites do not reach is a body's branches — a lexer's escape
  table, a printer's block string, a validator's second rule for the
  same section, an executor's list path.  The assertions are
  deliberately shallow, one call and one thing asserted about its
  answer; the deep ones stay where they are.  It also pins the
  `gqlcheck` fragment-spread recursion as a regression.
- Ten cases added to `tests/gqllang_tests.nv` (29 in all) and nineteen
  to `tests/gqlsdl_tests.nv` (38 in all): the lexer's tokens and its
  fault positions, a refusal's line AND column for each of the three
  fault kinds, the printer's four wrapper shapes, a description read as
  a value (an escape, a block string's `\"""`, and one
  `gqllang.parse_value` refuses), a definition written with no braces at
  all, and § 4.5's meta-types answering by name.

### Fixed

- Eleven defects found while writing the bodies and closing the coverage
  gate.  All of them were written in this release, so all of them are
  fixed in it rather than reported against a published one.

  - `gqlcheck`'s variable-use walk recursed through fragment spreads
    with no guard, so `validate` of `{ ...A } fragment A on Query { ...B
    } fragment B on Query { ...A count(n: $x) }` was SIGSEGV — the
    denial of service § 5.5.2.2's cycle rule exists to prevent.  The
    walk now carries the fragments it has entered (`first_use_from`) and
    the document answers both the cycle and the undeclared variable.
    `tests/graphql_nv_impl_tests.nv` pins it as a regression.
  - `gqlcheck.check_operation` compared a directive's declared locations
    against the lowercase `op_word` ("query"), so a custom directive
    declared `directive @d on QUERY` was refused on every operation.
    `op_location` is the § 3.13 uppercase spelling the schema is
    written in; a diagnostic still says "query".
  - A refusal said "may not appear on a inline fragment":
    `location_article` supplies the article the location word needs.
  - `gqlexec.plain_text` and `gqlexec.truth_of` were statement-position
    matches with a fall-through statement after them, so both answered
    the fall-through for every variable — `__type(name: $n)` answered
    null however the request supplied `$n`, and
    `fields(includeDeprecated: $keep)` dropped deprecated members
    however the query asked.  Each match is now its helper's whole body
    (`variable_text`, `variable_truth`), which is the shape
    `gqlcheck.variable_truth` already used; the "Known" entry below
    names the filing.
  - `gqlintro.jroot` wrote a root type as the bare name, so
    `queryType`/`mutationType`/`subscriptionType` were strings where
    § 4.5 writes `__Type` objects.  The document then read back with NO
    roots at all, because the reader (`root_name`) and the introspection
    query both expect `{"name": …}`.  It writes the object, and null for
    a root a schema does not have.
  - `gqllang.print_type` read `non_null`'s LENGTH where it had to read
    the marker at the LEVEL it was writing, so the four § 2.11 shapes
    printed wrongly in one direction — `[String]` as `[String!]`, and
    `[String]!` as `[String!]!`.  The array is deliberately short where
    the innermost named type is nullable; only the level says which
    wrapper a marker belongs to.  `base_type`, `wrap_list` and
    `inner_type` keep the same invariant — index 0 is the named type,
    the last index is the outermost wrapper — and
    `tests/gqlsdl_tests.nv` pins all four shapes as one string.
  - `gqllang.field_count` counted a fragment's fields twice for a
    document that has both an operation and fragments: once expanded
    through the operation that spreads them, and again when it measured
    the fragments themselves.  `{ ...A a: me } fragment A on Q { b: me
    }` answered 3 where the document names two fields.  The second pass
    now runs only for a document of fragments with no operation to be
    measured through.
  - `gqlsdl`'s printer wrote a type definition's keyword with the
    introspection word, so `print_schema` emitted `object Query { … }`
    and `input_object Point` — documents this package's own `parse`
    refuses, from the function a registry's diff rests on.  `sdl_word`
    is § 3's written form (`type`, `input`, `scalar`, `enum`,
    `interface`, `union`).
  - The same printer wrote a union with no members as `union U = `,
    which re-reads as a union whose first member is whatever the NEXT
    definition begins with — a type the deployment did not write there,
    accepted silently.  The `=` is written only when there are members.
  - Two directive definitions with one name were answered TWICE, which
    sends the reader looking for a second mistake; the duplicate-name
    rule reports it once, as it already did for types.
  - `gqlsdl` checked a field's directives and not its arguments', so
    `me(a: String @nope): String` was accepted where `me: String @nope`
    was refused.  An argument's directives are references like any
    other, and `ARGUMENT_DEFINITION` is a location § 3.13 names.
- **Validation and execution cost the document's size, not its paths.** A fragment spread along many paths was walked once per path by the variable checks, the unused-fragment search, the cycle search and the executor, so a 1.5 KB document of 24 doubling spreads took 43 s inside `default_limits()`. Each walk now shares one list of the fragments it has entered, and the executor skips a fragment already expanded into the same selection set, as CollectFields requires.

### Changed

- `novo.toml`: `stability = "experimental"`, the `novo` floor moves
  from the 0.8.9 the interface was drafted against to `>= 0.20.2`, the
  version to `0.1.0`, and the description no longer says "every body is
  a `todo()`".
- `README.md`: the status block and the implementation-status table are
  gone, the `default_limits()` column carries the values the code
  answers (100, 65536, 100, 256) rather than "a positive number", and
  the two paragraphs that described a red-on-purpose suite describe the
  green one.
- Every `tests/*_tests.nv` header loses its "RED ON PURPOSE" paragraph.

### Known

- **A call under an operator in a statement-position `match` arm is not
  diagnosed.**  `bugs/language/e2017-does-not-see-a-call-under-an-operator.md`
  is the filing: the error catalog documents that E2017 reaches exactly
  this shape ("this arm ends in a call producing Str"), and 0.20.2
  accepts `Some(k) => tagopt(k) ?? ""` and drops the arm's value in
  silence.  This package's own brush with it is the `gqlexec` fix above;
  the workaround — a `match` that is its function's whole body — is
  recorded at `variable_text` and `variable_truth`, and the two suites
  that hid it were all-literals, which is what to look for elsewhere.
- **An early `return` is not used where a call-bound local is live at
  the exit.**  On 0.20.2 it skips that local's release: every verdict
  green and `the run leaked`, exit 23.
  `bugs/memory-perceus/an-early-return-skips-the-release-of-a-call-bound-local-the-tail-moves.md`
  (`fixed_in: 0.20.3`) is the filing, and the workaround is recorded at
  its sites — `gqlsdl.sdl_from_tokens` (measured), `gqlexec.run_validated`
  (the same shape, avoided).  Collapse each back to the guard when the
  filing closes.
- **A `Str` join written in a `match` arm is leaked, whether or not the
  arm is taken.**  `bugs/language/a-join-in-a-match-arm-is-leaked-even-when-the-arm-is-not-taken.md`
  (`fixed_in: 0.20.3`) is the filing — filed from `exif-nv` on `Bytes`,
  with the `Str` face named as untested there.  It was measured here:
  `tests/gqlsdl_tests.nv`'s answer-building helpers put the join in a
  match arm (`Err(f) => "PARSE REFUSED: " + f.message()`), and the suite
  printed `37 passed, 0 failed` beside `4 object(s) leaked — exit 23` —
  one cell per call of the four the leak was bisected to.  Every arm's
  answer now goes through a helper whose whole body is the join (`cat`,
  `cat_sep`; `cat_sdl` in `gqlsdl` and `cat`/`spaced` in `gqllang` are
  the same shape), and the outer match's arms were split into their own
  functions, which is the sixth row of the filing's table.  The suite is
  green with no leak line and exit 0.  Note what did NOT reproduce: a
  standalone `novo build --llvm --leak-check` reduction of the `Str`
  face — the join in a single-level arm, in the inner arm of a nested
  match, with a call under the operator, and with a field read — was
  measured clean three times, so this package records the shape rather
  than a second reproduction.  Collapse the helpers back to `+` when the
  filing closes and re-measure.
- **A variant is never written through both its module and its enum.**
  `ranges.Range.Range4g` is `E6000`, an internal compiler error, on
  0.20.2; `bugs/language/a-module-qualified-enum-variant-is-an-ice.md`
  (`fixed_in: 0.20.3`) is the filing.  The spellings used here are the
  two that work: a bare variant with a `use
  <module>` (`GqlRequestMalformed(...)`, `GqlMutation`) and the
  module-qualified one without the enum (`gqlerr.GqlRequestMalformed`).
- **Subscriptions are parsed and not executed**, unchanged from 0.0.1:
  the missing half is the graphql-ws transport, and
  `gqlexec.subscription_gap` says so from the code.
- `gqlhttp.response_headers` emits `Content-Type` only.  The interface
  release's doc block promised `Allow` on a 405 as well, and a
  `GqlResponse` cannot know which method asked for it, so the doc now
  says where that header belongs: the caller attaches
  `allow_header()` to the 405 it answered.



The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- form-nv: `^0.0.1` to `^0.1.0`.

## 0.0.3 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- form-nv: `^0.0.1` to `^0.1.0`.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `gqlexec` — `GqlResolver[e]`, the load-bearing interface: one trait
  with an effect parameter, so `execute` is charged what the caller's
  resolver supplies rather than a union over every resolver anybody
  might write.  `GqlOutcome` separates a failure from a field the
  resolver does not know, `check_coverage` finds the second at
  start-up, and `null_lands_at` is § 6.4.4's propagation as one
  function.
- `gqllang` — the query language, `[]`.  Two entry points because
  § 2.2's document admits both grammars and no deployment wants both;
  `GqlLimits` checked as the parse descends, with aliases bounded
  separately from depth; `GqlTypeRef` carrying nullability per level,
  because `[User!]` and `[User]!` differ and a two-boolean type cannot
  say how.
- `gqlsdl` — the schema language, `[]`.  `parse` is syntax and `build`
  is resolution, answering separately because a diagnostic that said
  "invalid schema" for both would be useless for either; interfaces
  implementing interfaces, `@specifiedBy`, and `breaking_changes` with
  GraphQL's own asymmetry — removing an output field breaks clients and
  adding one does not.
- `gqlcheck` — § 5's rules, `[]`, and `validate` answers a LIST rather
  than a `Result`.  Every rule is named in the specification's own
  spelling and `validate_only`/`validate_except` take those names;
  `fragment_cycle` is the one rule that is a security control.
- `gqlintro` — `__schema`, `__type` and `__typename` from a schema
  value, `[]` — so a schema registry, a code generator or a linter
  produces an introspection document with nothing running.  `closed()`
  turns introspection AND the suggestions off, because either alone
  buys nothing.
- `gqlhttp` — the transport, `[]`.  `status_for` is the table:
  execution began is always 200, and a request error is 200 or 400 by
  the negotiated media type.  GET refuses a mutation with `Allow:
  POST`; `reads_content_type` refuses the form encodings, which is
  where a GraphQL endpoint's CSRF lives.
- `gqlerr` — two vocabularies, `GqlError` on the wire and `GqlFault`
  that never leaves the server, with `is_request_error` as the
  predicate that decides whether a response carries a `data` member at
  all.
- `tests/` — 107 API tests against the signatures.  Three pass, and
  they are the ones asserting the resolver trait can be implemented at
  two different effect rows — which is the design the release exists to
  publish.

### Known

- Every body is `todo()`.  `novo pkg build` is green and the shard rows
  that measure the design — `effect-budget`, `dep-layer`,
  `no-discharge-in-core` — pass.
- **Subscriptions are parsed and not executed.**  The missing half is
  the graphql-ws transport over a WebSocket, not the source event
  stream; `gqlexec.subscription_gap` says so from the code.
- **The language half wants a package of its own.**  Five of the seven
  modules are `[]` throughout, and a linter, a code generator, a schema
  registry and a gateway all want them without an executor.  Reported
  rather than acted on; the five are written so the split is a file
  move with no signature change.
- No `tests/embedded_probe.nv`: this is a `host` package, so it makes
  no device claim to check.

### Design notes

The resolver is one trait with an effect parameter rather than a table
of named functions. SPEC section 5.6 charges a generic function what
its bounded argument's implementation supplies, but a bounded value
reached through a container falls back to the union over every
implementation. A table of resolvers is such a container, so it would
have to name one fixed effect row, and the only row admitting every
resolver anybody might write is the whole host budget. Every GraphQL
server in the language would then declare `[io, fs, net, time, mutate]`
whether it read a database or answered from a constant.

Five of the seven modules — `gqlerr`, `gqllang`, `gqlsdl`, `gqlcheck`
and `gqlintro` — declare no effects and name neither `gqlexec` nor
`gqlhttp`, so a `graphql-core-nv` split would be a file move with no
signature change. A linter, a code generator, a schema registry and a
gateway each want those five and no executor, and today each has to
take a `host` package for one it will not call.
