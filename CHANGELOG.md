# Changelog

All notable changes to graphql-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
