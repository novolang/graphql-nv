# graphql-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

GraphQL, to the **October 2021** specification: the schema language,
the query language, § 5's validation rules, introspection, an executor,
and the HTTP transport a `web` application mounts.

It is what you reach for when a client wants to choose which fields it
gets.

It is a port of [async-graphql](https://github.com/async-graphql/async-graphql)
and [strawberry](https://strawberry.rocks).  Seven modules, and a reader
should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **executor** | `gqlexec` | you are running a query |
| the **transport** | `gqlhttp` | you are mounting an endpoint |
| the **schema** | `gqlsdl` | you are writing or reading SDL |
| the **query language** | `gqllang` | you are parsing a document |
| the **rules** | `gqlcheck` | you want to know why a query was refused |
| **introspection** | `gqlintro` | a tool wants your schema |
| the **errors** | `gqlerr` | you are deciding what to answer |

## Adding it, and checking it

```bash
novo pkg add graphql-nv          # into your novo.toml
novo pkg build                   # type- and effect-check the package
novo test --isolate tests/gqlexec_tests.nv
```

`novo test` is red today and that is the point of the release: nearly
every assertion fails with `not implemented: graphql-nv.<module>.<fn>`.
Three tests in `gqlexec_tests.nv` pass, and they are the ones asserting
about the *shape* the interface offers rather than about its bodies —
that the resolver trait can be implemented at two different effect
rows.

## The one example that will work

```novo
use gqlexec
use gqlhttp
use gqllang
use gqlsdl

struct Api
    db: Db

fn db_resolve(api: Api, ctx: GqlFieldCtx) -> GqlOutcome [net]
    match (ctx.type_name, ctx.field_name)
        ("Query", "me")   => GqlValue(api.db.current_user_json())
        ("User", "name")  => GqlValue(json_field(ctx.parent_json, "name"))
        _                 => GqlNoResolver

impl GqlResolver[net] for Api
    fn gql_resolve(self, ctx: GqlFieldCtx) -> GqlOutcome [net]
        db_resolve(self, ctx)
    fn gql_type_of(self, abstract_type: Str, value_json: Str) -> Str [net]
        json_field(value_json, "kind")
    fn gql_serialize_scalar(self, scalar: Str, value_json: Str) -> GqlOutcome [net]
        GqlValue(value_json)

fn handle(api: Api, schema: GqlSchema, body: Str, accept: Str) -> (Int, Str) [net]
    let media = gqlhttp.negotiate(accept)
    match gqlhttp.read_post(body, gqlhttp.defaults())
        Err(f) => (gqlhttp.status_for_fault(f, media), gqlhttp.fault_body(f, media))
        Ok(req) =>
            match gqlexec.execute(schema, api, req, "{}", gqllang.default_limits())
                Err(f) => (gqlhttp.status_for_fault(f, media),
                           gqlhttp.fault_body(f, media))
                Ok(r)  => (gqlhttp.status_for(r, media), gqlexec.response_json(r))
```

`handle` declares `[net]` because `Api`'s resolvers do.  A deployment
whose resolvers answer from memory writes the same code and declares
nothing.

## The load-bearing interface

`gqlexec.GqlResolver[e]` — one trait, with an **effect parameter**.

```novo norun:pseudo
pub trait GqlResolver[e]
    fn gql_resolve(self, ctx: GqlFieldCtx) -> GqlOutcome [e]
    fn gql_type_of(self, abstract_type: Str, value_json: Str) -> Str [e]
    fn gql_serialize_scalar(self, scalar: Str, value_json: Str) -> GqlOutcome [e]
```

and the shape it is **not** is the argument.

The obvious API is a **table**: a map from `(type, field)` to a named
function, registered at start-up, which is what a reader coming from
async-graphql or graphql-js expects.  It cannot be effect-polymorphic
here, and the reason is precise rather than a limitation to grumble
about.

SPEC § 5.6 charges a generic function what its bounded argument's
`impl` supplies — but a bounded value reached through a **container**
falls back to the union over every impl, which is documented and
deliberate.  A table of resolvers is exactly that container.  So a
table has to name **one fixed effect row**, and the only row that
admits every resolver anybody might write is the whole host budget.
Every GraphQL server in the language would then declare
`[io, fs, net, time, mutate]` whether its resolvers read a database or
answered from a constant — which makes the row a ceiling instead of a
description, and a ceiling tells a reviewer nothing.

One trait, implemented once, does not have that problem:

```novo norun:pseudo
pub fn execute<R: GqlResolver[e]>(…) -> Result<GqlResponse, GqlFault> [e]
```

`tests/gqlexec_tests.nv` implements two resolvers — one
`GqlResolver[]`, one `GqlResolver[fs]` — and calls `execute` against
both.  The test over the first declares `[io]`; the test over the
second declares `[io, fs]`.  Same call, two rows, and the compiler is
the thing asserting it.

**And it is the faithful port.**  async-graphql's `#[Object]` macro
*generates* a dispatch over field names on one impl; strawberry's
decorator does the same over one class.  What you write here is that
dispatch by hand — a `match` on `(ctx.type_name, ctx.field_name)` —
which is the code those macros produce, written where a reader can see
it.  There is no macro to write it for you because `@derive` is
reserved and refused in this language.

## Does the language half earn its own row? Yes.

Five of the seven modules are `[]` throughout — `gqlerr`, `gqllang`,
`gqlsdl`, `gqlcheck` and `gqlintro` — and they are the bigger half by
some distance: a lexer, two parsers, twenty-two validation rules, and
introspection answered from a schema value.

Unlike the usual case, this one has **several real second consumers**:

- a **linter** wants the parser and the rules and no executor;
- a **code generator** wants the schema and the document and nothing
  else;
- a **schema registry** wants `build`, `print_schema`,
  `breaking_changes` and `deprecations`, and never runs a query;
- a **gateway** wants `gqlintro.schema_of` to turn a downstream
  service's introspection answer back into a schema;
- and this language's own tooling would want all of the above before
  it wants an endpoint.

Every one of those is forced to take a `host` package today, for an
executor it will not call.

So **`graphql-core-nv` is a row this grid is missing**, and that is
reported rather than acted on unilaterally: the plan is the
authoritative row list.  The five modules are written so the split is a
file move with **no signature change** — nothing in them names
`gqlexec` or `gqlhttp`, and `gqlexec` depends on them and not the other
way round.

## What is in, and what is not

**October 2021**, and the three things it adds over June 2018 that an
implementation notices:

- an interface may implement other interfaces (§ 3.7), which makes
  conformance a transitive walk rather than a lookup;
- a directive may be `repeatable` (§ 3.13), and introspection answers
  `__Directive.isRepeatable`;
- `@specifiedBy(url:)` on a custom scalar (§ 3.5.7), which is the only
  machine-readable meaning a custom scalar has.

**Subscriptions are a named gap.**  `gqllang` parses one and `gqlsdl`
accepts a subscription root type; `gqlexec.execute` refuses one with
`GqlUnsupported("subscription")`.  The missing half is not § 6.2.3's
source event stream — that is a resolver answering many values instead
of one — it is the **transport**: the graphql-ws sub-protocol over a
WebSocket, which is websocket-nv's surface plus a message protocol of
its own.  When it is written it is `gqlexec.subscribe` beside
`execute`, and nothing in the current surface changes.

**Out with reasons**: `@defer` and `@stream` (in no ratified
specification), federation and schema stitching (a different package
over `gqlsdl`), dataloader-style batching (every solution to it is a
cache with a lifetime, which a library cannot choose — `GqlFieldCtx.context`
is where a caller's loader lives), and the custom-scalar half of
§ 5.6.1 (whether `"2026-13-45"` is a valid `Date` is the scalar's own
question, and `gqlcheck.unchecked_rules` says so from the code).

## Five things this package refuses to let happen quietly

**A 500 for a failed resolver.**  A response carrying `errors` is a
**200** as long as execution began — the query ran, some fields failed,
the body says which.  A client library that retried on 5xx would,
against a server that answered 500, retry every partial success
forever.  `gqlhttp.status_for` is the whole table, and it takes the
negotiated media type because that is what the GraphQL over HTTP
specification makes the status depend on.

**A mutation over GET.**  `<img src="/graphql?query=mutation{deleteAll}">`
on any page anywhere.  `gqlhttp.refuses_get_mutation` answers `true` by
default, and the refusal carries `Allow: POST` — which tells the client
what to do, where a 400 would only say something was wrong.

**A form-encoded GraphQL endpoint.**  A GraphQL endpoint that accepts
`application/x-www-form-urlencoded` has a cross-site request forgery on
every mutation, because a form on another site can send one and JSON it
cannot.  `gqlhttp.reads_content_type` answers `false` for it, and
`gqlhttp.looks_preflighted` is the check that reasoning belongs to.

**A fragment cycle.**  `fragment A { ...B }` with `fragment B { ...A }`
is a document that expands forever — a single-request denial of
service.  It is § 5.5.2.2, it is in `gqlcheck.rule_names()`, and
`gqlcheck.fragment_cycle` answers the cycle it found so a diagnostic
can show `A → B → A` rather than "cyclic fragment".

**A query whose depth a client chose.**  `{a{a{a{a{…}}}}}` costs the
server its stack before a validation rule has run, so `GqlLimits` is
checked **as the parse descends**.  Aliases have their own ceiling,
because `{a:me b:me c:me …}` is one field resolved a thousand times
from a document a thousand characters long and a depth limit does not
see it.

## Two more decisions worth reading before you depend on this

**`validate` cannot fail.**  It answers a list, never a `Result`: a
document that does not satisfy the schema is the *answer*, and
modelling it as an error would put a developer's typo on the failure
path.  Everything genuinely wrong is wrong in the schema, and
`gqlsdl.build` is the only call here that answers a `Result` about one.
schema-nv and openapi-nv make the same split, and the three compose
because of it.

**No `data` member and `"data": null` are different things.**  The
first means the request never executed; the second means it did and the
root field failed.  § 7.1 makes the distinction and a client library
branches on it, so `GqlResponse.data` is empty text for the first case
and the literal `null` for the second, and `gqlexec.has_data` is how a
caller tells them apart.

## What it depends on

One package: **form-nv**, for the GET transport's query string — and
specifically for the case that matters, a request carrying two `query`
parameters, which `http.server.query_get` answers the first of and
drops the rest.

Not serde-nv (the executor answers the standard library's own JSON
value), not http-codec-nv (this package reads a body a server handed
it), and not websocket-nv (depending on a WebSocket client in order
*not* to implement subscriptions would be a dependency every
query-only deployment paid for).

## Related

- [openapi-nv](https://github.com/novolang/openapi-nv) — the same
  question for the other API style, and the same conclusion about
  deriving a schema from types
- [schema-nv](https://github.com/novolang/schema-nv) — validation that
  cannot fail, arrived at independently
- [session-nv](https://github.com/novolang/session-nv) — where the
  signed-in person a resolver's `context` names comes from
