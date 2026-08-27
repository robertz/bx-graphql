# bx-graphql

A schema-agnostic GraphQL server for native BoxLang, wrapping [graphql-java](https://www.graphql-java.com/) (vendored, v26.0). The module ships with zero domain schema of its own — it parses `.graphqls` files you supply, wires resolvers by convention, and hands back a plain `{data, errors}` struct for whatever you execute. No ColdBox, no WireBox, no HTTP framework required — how you route requests to it is entirely up to you.

BoxLang's own module system auto-loads the jars under `libs/` onto the module's classloader, so `createObject("java", ...)` and `createDynamicProxy()` always resolve graphql-java classes through the same classloader — no separate classloader-registration step, and no per-engine branching, unlike the ColdBox edition of this module.

## Installation

```bash
box install bx-graphql
```

Requires BoxLang **1.15.0+**. This module only runs on BoxLang — it isn't tested against Lucee or Adobe ColdFusion.

## Quick start

```javascript
graphQLService = bxGraphQL( {
	"schemaPaths"         : [ expandPath( "/graphql/schema" ) ],
	"resolverBasePackage" : "models.resolvers"
} )

result = graphQLService.execute(
	query          = '{ widget(id: "1") { name } }',
	queryVariables = {},
	context        = ""
)
// result == { "data" : { "widget" : { "name" : "..." } }, "errors" : [] }
```

`bxGraphQL()` is a global BIF (`bifs/bxGraphQL.bx`), auto-registered by the module system the moment this module loads — no `new` or namespace needed. It's just a thin wrapper: `new bxModules.bxgraphql.models.GraphQLService( settings )` still works identically if you'd rather be explicit about where it comes from.

`GraphQLService` builds the graphql-java engine once, at construction time — not per request — so build it once (e.g. at app startup) and reuse it. `execute()` doesn't touch HTTP at all — wiring `POST /graphql` (or wherever you want it) up to a real request is on your app's own router.

## Adding your schema

Point `schemaPaths` at one or more `.graphqls` files, directories, or wildcards — every `*.graphqls` file resolved is parsed and merged into one schema automatically (see `resolveSchemaFiles()` in `models/GraphQLService.bx`):

```javascript
schemaPaths : [ expandPath( "/graphql/schema" ) ]   // every *.graphqls file in this directory
schemaPaths : [ expandPath( "/graphql/schema/*.graphqls" ) ]   // equivalent, explicit wildcard
schemaPaths : [ expandPath( "/graphql/schema.graphqls" ) ]   // a single literal file
```

Splitting a schema across files works out of the box — forward references across files (a type used in one file, defined in another) resolve fine regardless of file order, since all files are parsed and merged before validation. You can also split operations themselves across files with GraphQL's `extend` keyword, e.g. `extend type Query { widget(id: ID!): Widget }` in a file that doesn't define `Query` itself — useful once a single growing `Query`/`Mutation` block gets unwieldy.

The module fails fast at construction time — `BxGraphQL.ConfigurationException` — if `schemaPaths` is empty, or if any configured path (including one wildcard entry among several) doesn't resolve to at least one file.

## Settings

Passed as a struct to `GraphQLService.init()`:

| Setting | Required | Description |
|---|---|---|
| `schemaPaths` | Yes | Array of paths to `.graphqls` files — literal files, directories, or wildcards. See *Adding your schema* above. |
| `resolverBasePackage` | Yes | Where resolver classes live. A dotted package path (e.g. `models.resolvers`), resolved the same way `expandPath()` resolves any relative path — relative to the entry point of the running script, not necessarily the file that set this option (see *A note on `expandPath()`* below). A value containing `/` is treated as an already-literal path instead. |

## Resolver convention

For a schema type `TypeName` with field `fieldName`, the module looks for `{resolverBasePackage}/{TypeName}Resolver.bx` and, if it exists and implements a `fieldName()` method, calls it. Otherwise it falls back to graphql-java's `PropertyDataFetcher` — reading a same-named key off the parent object. **You only write a resolver for fields that need custom logic** — a struct with matching keys just works.

Resolver methods receive four named arguments:

```javascript
any function fieldName( any source, struct args, any context, any env ){
	// source  — the parent object (whatever the parent resolver returned, or a plain
	//           struct/key). NULL for root Query/Mutation fields — don't mark it `required`.
	// args    — the field's GraphQL arguments, as an ordinary BoxLang struct/array
	//           (converted from graphql-java's raw arguments via a JSON round-trip).
	// context — whatever was passed as `context` to GraphQLService.execute() — typically
	//           the current request, however your app represents it.
	// env     — the raw graphql.schema.DataFetchingEnvironment, for advanced use.
}
```

### Example

`schema.graphqls`:

```graphql
type Query {
	widget(id: ID!): Widget
}

type Widget {
	id: ID!
	name: String
	slug: String
}
```

`models/resolvers/QueryResolver.bx`:

```javascript
class {
	any function widget( any source, required struct args, any context, any env ){
		return { "id" : arguments.args.id, "name" : "Test Widget", "slug" : "test-widget" };
	}
}
```

Resolver classes are instantiated with a bare, no-argument `new()` — there's no DI container here, so a resolver can't declare a required `init()` dependency the way a WireBox-managed CFC could. Reach any dependency it needs (a service, a datasource) through your app's own means — a singleton lookup, an application-scope reference, whatever pattern the rest of your app already uses.

No `WidgetResolver.bx` is needed at all — `id`, `name`, and `slug` all resolve via `PropertyDataFetcher` off the struct `widget()` returned. If `slug` later needs to be computed rather than stored, add `models/resolvers/WidgetResolver.bx` with just a `slug()` method; `id` and `name` keep resolving automatically.

### Mutations

`Mutation` gets no special treatment — it's just another object type name to the wiring loop in `GraphQLService.wireResolvers()`, so it follows the exact same `{resolverBasePackage}/{TypeName}Resolver.bx` convention as `Query`, with the same four named arguments. Per the GraphQL spec, top-level mutation fields in a single request execute serially rather than in parallel like top-level query fields — that's handled entirely by graphql-java's own execution strategy once it sees a `Mutation` root type; nothing to configure here.

## Executing queries

```javascript
result = graphQLService.execute(
	query          = "...",    // required
	queryVariables = {},       // optional — GraphQL $variables
	context        = ""        // optional — forwarded as-is to every resolver's `context` arg
)
```

Returns a plain BoxLang struct — `{ data : {...}, errors : [...] }` — so callers never touch graphql-java classes directly. `errors` is always present (an empty array on success) and holds one message string per graphql-java error, e.g. for an invalid field or a resolver exception.

## Accessing the schema programmatically

`getSchema()` returns the raw `graphql.schema.GraphQLSchema` graphql-java built from your `.graphqls` files; `getEngine()` returns the underlying `graphql.GraphQL` engine itself. Both are live graphql-java objects — useful for anything graphql-java itself supports, most commonly printing the schema back out as SDL via `graphql.schema.idl.SchemaPrinter` for docs or codegen tooling.

## Not yet implemented

- Custom scalars
- Interface/union `TypeResolver` wiring — every object type gets its own convention-based resolver, but resolving *which* concrete type implements an interface/union at runtime isn't wired up
- An introspection on/off toggle (introspection is always available, per graphql-java's default)

## Tests

[TestBox](https://testbox.ortusbooks.com/) specs, run headlessly (no server needed):

```bash
boxlang setup-tests.bxs   # once per checkout
boxlang run-tests.bxs     # every time after that
```

`run-tests.bxs` exits non-zero on any failure/error, so it's CI-friendly — just run `setup-tests.bxs` once beforehand (e.g. in a CI image build step). Structure:

- `tests/specs/GraphQLServiceSpec.bx` — the resolver convention end-to-end against a real engine: `PropertyDataFetcher` fallback, explicit-resolver precedence, missing-resolver-class fallback, query variables actually reaching resolver arguments, and spec-shaped errors for an invalid query.
- `tests/specs/GraphQLServiceConfigSpec.bx` — startup config validation: missing/invalid `schemaPaths` or `resolverBasePackage`, including a wildcard `schemaPaths` entry whose directory doesn't exist.
- `tests/specs/DataFetcherAdapterSpec.bx` — resolver dispatch in isolation, via a fake `DataFetchingEnvironment` (`tests/helpers/FakeDataFetchingEnvironment.bx`). Doesn't cover the `PropertyDataFetcher`-fallback path — graphql-java's real `PropertyDataFetcher.get()` calls more of that interface than a fake can cheaply satisfy — that path is covered end-to-end instead, in `GraphQLServiceSpec`.
- `tests/specs/JavaClassFactorySpec.bx` — Java class-handle resolution, using plain JDK classes so it runs without the module-loading setup below.
- `tests/specs/BifsSpec.bx` — the `bxGraphQL()` global BIF, built entirely through the friendly entry point rather than `new bxModules.bxgraphql.models.GraphQLService()` directly.

`setup-tests.bxs` creates a self-referencing `boxlang_modules/bxgraphql` symlink (gitignored, not committed) so BoxLang discovers this project as a real module and loads `libs/` onto the classloader `createObject("java", ...)` sees — module discovery only happens once at BoxLang process startup, so this has to run as its own invocation before `run-tests.bxs`, not get folded into it.

### A note on `expandPath()`

`expandPath()` resolves relative to the entry point of the running script — `run-tests.bxs` at the repo root when running the test suite — not relative to whichever file happens to call it. This is why `resolverBasePackage` in the examples above is documented as resolving relative to "the entry point," and why the test specs use repo-root-relative paths (`tests/resources/...`) rather than paths relative to each spec file's own directory.

## License

MIT
