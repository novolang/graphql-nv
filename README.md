# graphql-nv

GraphQL is a query language for APIs, and a server-side runtime for
answering those queries against a typed schema. It is specified by the
[GraphQL specification, October 2021 edition](https://spec.graphql.org/October2021/),
and the HTTP binding is
[GraphQL over HTTP](https://graphql.github.io/graphql-over-http/draft/).
This package brings the schema language, the query language, the
validation rules, introspection, an executor and that HTTP binding to
novo-lang. It is a port of
[async-graphql](https://github.com/async-graphql/async-graphql) and
[strawberry](https://strawberry.rocks).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What GraphQL is

A server publishes a **schema**: a set of types, each with named
fields, and one type nominated as the query root. The schema is written
in the **schema definition language** (SDL), which specification
section 3 defines. `type User { name: String }` is a type with one
field.

A client sends a **document** written in the query language. A document
holds **operations** and **fragments**. An operation is a `query`, a
`mutation` or a `subscription`, and it carries a **selection set**: the
fields the client wants, nested to follow the schema's shape.
`{ me { name } }` asks the query root for `me` and asks that value for
`name`. The response mirrors the selection set, so a client gets the
fields it asked for and no others.

A **fragment** is a named selection set that other selections include
with `...Name`, so a repeated group of fields is written once. An
**alias** renames a field in the response, so `{ a: me b: me }` asks
for the same field twice under two keys.

A field may take **arguments**, and an operation may declare
**variables** that the request supplies values for separately from the
document. Section 6.1.2 coerces those values against their declared
types before anything runs.

**Validation** is section 5: a list of rules that decide whether a
document can be executed against a schema. A document that breaks one
is refused before execution, so a server never starts work on a query
it cannot finish. The rules cover unknown fields, fragment cycles,
conflicting selections, unused variables and more.

**Execution** is section 6. The executor walks the selection set and
calls a **resolver** for each field: a function the deployment writes
that answers that field's value for that parent value. A field that
fails contributes an entry to the response's `errors`, and section
6.4.4 decides how far the resulting `null` propagates up the response.

**Introspection** is section 4. The schema itself is queryable through
the meta-fields `__schema`, `__type` and `__typename`, which is how
every client tool, editor plugin and code generator discovers a schema.

This package implements the October 2021 edition. Three of its
additions over the June 2018 edition change what an implementation
does. An interface may implement other interfaces (section 3.7), which
makes conformance a transitive walk rather than a lookup. A directive
may be declared `repeatable` (section 3.13), and introspection answers
`__Directive.isRepeatable`. A custom scalar may carry
`@specifiedBy(url:)` (section 3.5.7), which is the only
machine-readable meaning a custom scalar has.

The specification defines no transport. GraphQL over HTTP is the
convention: a POST whose JSON body carries `query`, and optionally
`operationName`, `variables` and `extensions`.

## Install

```
novo pkg add graphql-nv
```

## Example

```novo
use std.list
use gqlcheck
use gqllang
use gqlsdl

fn main() [io]
    // A schema, written in the schema definition language.
    match gqlsdl.build("type Query { me: User } type User { name: String }")
        Err(f) => println("the schema is wrong: ${f.message()}")
        Ok(schema) =>
            // Parse the client's query. The limits are applied as the
            // parse descends, before any validation rule runs.
            match gqllang.parse_executable("{ me { name } }", gqllang.default_limits())
                Err(f) => println("the query did not parse: ${f.message()}")
                Ok(doc) =>
                    // Check the query against the schema. This cannot
                    // fail: a query that does not fit answers a list.
                    let problems = gqlcheck.validate(schema, doc)
                    if list.len(problems) == 0
                        println("the query is valid")
                    else
                        for e in problems
                            println(e.message)
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
is almost entirely red on purpose: nearly every test reaches a
`not implemented: graphql-nv.<module>.<fn>` panic. The exceptions are
three tests in `tests/gqlexec_tests.nv`, which assert about the shape
of the resolver trait rather than about any body, and which the
compiler decides before the run starts.

## What the package contains

| Module | Contents |
| --- | --- |
| `gqlerr` | An error as section 7.1.2 shapes one, the faults that end an exchange instead of annotating one, and the question of which of them is safe to show a client. |
| `gqllang` | The query language: the document types, the parser, the printer, and the limits applied as the parse descends. |
| `gqlsdl` | The schema language: the schema types, the parser, the builder, the printer, and the lookups and subtype questions the other modules ask. |
| `gqlcheck` | Section 5's validation rules, variable coercion, and the individual checks published on their own. |
| `gqlintro` | Section 4's introspection: the meta-fields, the policy that decides whether they answer, and a schema read back out of another server's introspection reply. |
| `gqlexec` | Execution: the resolver trait, the field context, the response, and the two entry points that run a request. |
| `gqlhttp` | The HTTP binding: reading a request out of a body or a query string, content negotiation, the status codes, and the response headers. |

## How to choose an entry point

**`gqlexec.execute` runs a request.** It takes the schema, a resolver,
the request, the root value as JSON and the parse limits, and answers
a response or a fault. **`execute_document` takes a document already
parsed**, for a deployment that parses once and runs many times, such
as one serving persisted queries.

**`gqlhttp.read_post`, `read_get` and `read_form` turn a transport into
a request.** Each takes a `GqlHttpPolicy`, which decides whether GET is
accepted at all and how large a body may be.

**`gqlsdl.build` is the only call here that answers a `Result` about a
schema.** Everything wrong with a schema is wrong there.
**`gqlcheck.validate` cannot fail**: a document that does not satisfy
the schema is the answer, as a list of errors.

**`gqllang`, `gqlsdl`, `gqlcheck`, `gqlintro` and `gqlerr` declare no
effects and name nothing in `gqlexec` or `gqlhttp`.** A linter, a code
generator or a schema registry uses those five and never runs a query.

## The rules a user needs

1. **A resolver is one trait with an effect parameter.**
   `gqlexec.GqlResolver[e]` has three methods, and a deployment
   implements it once. `execute` is charged whatever that
   implementation declares, so a server whose resolvers read a database
   declares `[net]` and one answering from memory declares nothing.
2. **The dispatch over field names is yours to write.** A resolver
   matches on `(ctx.type_name, ctx.field_name)`. That is the code
   async-graphql's `#[Object]` macro and strawberry's decorator
   generate; this language reserves and refuses `@derive`, so it is
   written where a reader can see it.
3. **`GqlNoResolver` is not a failure.** It means the schema and the
   resolver disagree, which is a deployment fault rather than something
   a client did. `gqlexec.check_coverage` finds every such field at
   start-up.
4. **A response with `errors` is still a 200, as long as execution
   began.** `gqlhttp.status_for` is the whole table, and it takes the
   negotiated media type because that is what the status depends on.
   Answering 500 for a failed resolver makes every client library that
   retries on 5xx retry a partial success forever.

   | Case | `application/json` | `application/graphql-response+json` |
   | --- | --- | --- |
   | Execution began, with or without errors | 200 | 200 |
   | A request error | 200 | 400 |
   | A variable coercion failure | 200 | 400 |

5. **No `data` member and `"data": null` are different answers.** The
   first means the request never executed. The second means it did and
   the root field failed. Section 7.1 makes the distinction and client
   libraries branch on it. `GqlResponse.data` is empty text for the
   first and the literal `null` for the second, and `gqlexec.has_data`
   tells them apart.
6. **A mutation over GET is refused.** `GqlHttpPolicy.allow_get_mutation`
   is false and stays false: a mutation a browser can prefetch is a
   mutation a crawler will perform. `gqlhttp.refuses_get_mutation`
   answers the question and the refusal carries
   `gqlhttp.allow_header()`.
7. **A GraphQL endpoint must not accept `application/x-www-form-urlencoded`.**
   A form on another site can send that content type and cannot send
   JSON, so accepting it is a cross-site request forgery on every
   mutation. `gqlhttp.reads_content_type` answers `false` for it, and
   `gqlhttp.looks_preflighted` is the check behind that reasoning.
8. **The parse limits are applied as the parse descends.**
   `{a{a{a{...}}}}` costs the server its stack before any validation
   rule has run. Aliases have their own ceiling, because `{a:me b:me
   c:me ...}` is one field resolved a thousand times from a short
   document and a depth limit does not see it.
9. **A fragment cycle is a single-request denial of service.**
   `fragment A { ...B }` with `fragment B { ...A }` expands forever.
   It is section 5.5.2.2, and `gqlcheck.fragment_cycle` answers the
   cycle it found, so a diagnostic can print `A -> B -> A`.
10. **`gqllang.parse_executable` refuses a type-system definition by
    name.** A query document carrying `type User { ... }` is a client
    trying to change the schema, and dropping it quietly would leave a
    deployment no way to notice.
11. **Arguments reaching a resolver are already coerced.** Section
    6.4.1 applies defaults and substitutes variables before the field
    is resolved, so `ctx.args_json` is read rather than checked.
12. **`ctx.type_name` is always a concrete type**, never an interface
    or a union name. Section 6.4.1 resolves the abstract type first, by
    calling the resolver's `gql_type_of`.
13. **`ctx.field_name` is the schema's name and `ctx.response_key` is
    the alias.** Dispatch on the first. The second is a name the client
    chose.
14. **A resolver is called once per field per parent object.** A list
    of a hundred users with three fields each is three hundred calls.
    Batching is the deployment's own business, and
    `GqlFieldCtx.context` is where a batch loader lives.
15. **Turning introspection off without turning suggestions off buys
    nothing.** A validation error that says "did you mean" discloses
    the schema one field at a time. `gqlintro.closed()` turns off both.
16. **Locations are one-based, and columns are counted in characters.**
    Section 7.1.2 requires the first. A column counted in bytes points
    into the middle of a name as soon as one carries an accent.
17. **A machine-readable code belongs in `GqlError.extensions`, not in
    `message`.** `message` is prose for a developer, and a client that
    matched on prose breaks when the prose improves.

## Sizes and limits

| `GqlLimits` field | What it bounds | `default_limits()` |
| --- | --- | --- |
| `max_depth` | The deepest selection set | 15 |
| `max_breadth` | The most selections in one set | a positive number |
| `max_bytes` | The most bytes in the document | a positive number |
| `max_aliases` | The most aliases in the document | a positive number |
| `max_spreads` | The most fragment spreads | a positive number |

`gqllang.no_limits()` sets every bound to zero, meaning no ceiling. It
is a named function rather than five fields a caller zeroes, so a
deployment that uses it has said so. `gqllang.limits` takes all five.

## What is not included

- **Subscriptions.** `gqllang` parses one and `gqlsdl` accepts a
  subscription root type, but `gqlexec.execute` refuses one with
  `GqlUnsupported("subscription")`. The missing half is the transport,
  the graphql-ws sub-protocol over a WebSocket. When it lands it is
  `gqlexec.subscribe` beside `execute`, and no current signature
  changes. `gqlexec.subscription_gap` states this from the code.
- **`@defer` and `@stream`.** Neither is in a ratified specification.
- **Federation and schema stitching.** A separate package over
  `gqlsdl`.
- **Batched data loading.** Every solution is a cache with a lifetime,
  and a library cannot choose that lifetime.
  `GqlFieldCtx.context` is where a caller's loader lives.
- **Custom scalar input validation.** Whether `"2026-13-45"` is a valid
  `Date` is the scalar's own question. Section 5.6.1's custom-scalar
  half is listed by `gqlcheck.unchecked_rules` from the code, and
  output coercion is the resolver's `gql_serialize_scalar`.
- **A socket.** This package reads a body a server handed it.

## Related packages

- [openapi-nv](https://novo-lang.org/packages/openapi-nv) describes a
  REST API as a document. Take it when the client asks for a fixed
  resource. Take this package when the client chooses its fields.
- [schema-nv](https://novo-lang.org/packages/schema-nv) validates JSON
  against a compiled schema, and its validation cannot fail either.
- [form-nv](https://novo-lang.org/packages/form-nv) parses a query
  string to the WHATWG rules, keeping every value under a repeated key.
  `gqlhttp.read_get` needs that, because a request carrying two `query`
  parameters is a request a parser must not silently narrow. This
  package depends on it.
- [session-nv](https://novo-lang.org/packages/session-nv) is where the
  signed-in person that a resolver's `context` names comes from.
- [websocket-nv](https://novo-lang.org/packages/websocket-nv) is what
  subscriptions will need. This package does not depend on it, so a
  query-only deployment does not link it.
- `std.json` in the standard library is the JSON this package's
  resolvers answer and its requests arrive as. This package does not
  depend on serde-nv.

## Tests

```bash
novo test tests/gqlerr_tests.nv     # errors, faults, and what is safe to show
novo test tests/gqllang_tests.nv    # the query parser, the printer, the limits
novo test tests/gqlsdl_tests.nv     # the schema language and the subtype rules
novo test tests/gqlcheck_tests.nv   # section 5's rules
novo test tests/gqlintro_tests.nv   # introspection and the policy
novo test tests/gqlexec_tests.nv    # execution and the resolver trait
novo test tests/gqlhttp_tests.nv    # the transport, the statuses, the refusals
```

The normative source is the October 2021 specification: section 3 for
the type system, section 4 for introspection, section 5 for the
validation rules, section 6 for execution and section 7 for the
response. The status codes come from the GraphQL over HTTP
specification. The reference implementations are async-graphql and
strawberry.

The suite asserts that a fragment cycle is reported as the cycle it is,
that a document exceeding a limit is refused during the parse, that a
mutation over GET is refused with `Allow: POST`, that a form-encoded
content type is not read, that a response carrying errors is still a
200 when execution began, and that no `data` member and a null `data`
are distinguishable.

Three tests in `tests/gqlexec_tests.nv` pass today. They implement the
resolver trait twice, once at `GqlResolver[]` and once at
`GqlResolver[fs]`, and call `execute` against both: the test over the
first declares `[io]` and the test over the second declares
`[io, fs]`. The compiler decides that before the run starts, so those
three assert about the interface rather than about a body.

Every other test compiles today and fails at run, each on the
`not implemented: graphql-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| Every `pub struct`, `pub enum` and `pub trait` in the seven modules | the types are declared |
| `gqlerr.error`, `.error_with_code`, `.at`, `.at_path`, `.with_extensions` | no |
| `gqlerr.code_of`, `.path_text`, `.error_json` | no |
| `gqlerr.errors_of`, `.is_request_error`, `.is_configuration_fault`, `.is_client_safe` | no |
| `gqlerr.GqlFault.message` | no |
| `gqllang.default_limits`, `.no_limits`, `.limits` | no |
| `gqllang.parse_executable`, `.operation_of`, `.fragment_of` | no |
| `gqllang.print_document`, `.print_selections`, `.print_value`, `.print_type` | no |
| `gqllang.parse_value`, `.null_value`, `.parse_type`, `.named_type` | no |
| `gqllang.is_non_null`, `.is_list`, `.inner_type` | no |
| `gqllang.depth_of`, `.alias_count`, `.field_count`, `.response_key`, `.field_selection` | no |
| `gqlsdl.parse`, `.build`, `.build_from`, `.print_schema` | no |
| `gqlsdl.type_named`, `.field_named`, `.arg_named`, `.possible_types` | no |
| `gqlsdl.is_subtype`, `.is_input_type`, `.is_output_type`, `.is_composite` | no |
| `gqlsdl.builtin_scalars`, `.builtin_directives`, `.is_reserved_name` | no |
| `gqlsdl.deprecations`, `.breaking_changes` | no |
| `gqlcheck.validate`, `.is_valid`, `.validate_only`, `.validate_except` | no |
| `gqlcheck.rule_names`, `.rule_section`, `.unchecked_rules` | no |
| `gqlcheck.fragment_cycle`, `.merge_conflicts`, `.variable_problems`, `.impossible_fragments`, `.directive_problems` | no |
| `gqlcheck.coerce_variables`, `.value_fits`, `.selection_included` | no |
| `gqlintro.defaults`, `.closed`, `.is_meta_field`, `.answers` | no |
| `gqlintro.schema_value`, `.type_value`, `.document`, `.schema_of` | no |
| `gqlintro.introspection_query`, `.meta_type_names`, `.type_kind_name`, `.directive_locations`, `.suggest_field` | no |
| `gqlexec.execute`, `.execute_document` | no |
| `gqlexec.request`, `.with_variables`, `.with_operation` | no |
| `gqlexec.response_json`, `.empty_response`, `.error_response`, `.has_data`, `.with_extensions` | no |
| `gqlexec.null_lands_at`, `.runs_serially`, `.check_coverage`, `.subscription_gap` | no |
| `gqlhttp.defaults`, `.post_only`, `.read_post`, `.read_get`, `.read_form` | no |
| `gqlhttp.refuses_get_mutation`, `.allow_header`, `.reads_content_type` | no |
| `gqlhttp.negotiate`, `.status_for`, `.status_for_fault`, `.content_type`, `.response_headers`, `.fault_body` | no |
| `gqlhttp.looks_preflighted`, `.preflight_headers`, `.persisted_hash`, `.persisted_not_found` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
