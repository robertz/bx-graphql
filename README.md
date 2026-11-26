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
| `schemaPaths` | Yes, unless `modelPaths` or `servicePaths` is set | Array of paths to `.graphqls` files — literal files, directories, or wildcards. See *Adding your schema* above. |
| `resolverBasePackage` | Yes, unless `modelPaths` or `servicePaths` is set | Where resolver classes live. A dotted package path (e.g. `models.resolvers`), resolved the same way `expandPath()` resolves any relative path — relative to the entry point of the running script, not necessarily the file that set this option (see *A note on `expandPath()`* below). A value containing `/` is treated as an already-literal path instead. |
| `modelPaths` | No | Array of class files or directories (searched recursively). Classes marked `@graphqlType` / `@graphqlInput` become types and inputs. See *Code-first schema* below. |
| `servicePaths` | No | Array of class files or directories (searched recursively). Public functions marked `@graphqlQuery` / `@graphqlMutation` become fields of `Query` / `Mutation`. |
| `instanceFactory` | No | `( classPath, meta ) => instance`, called once per service class at construction time, for services that need dependencies. Default: a bare `new`. `classPath` is the class file's absolute path without its extension; `meta` is its `getClassMetadata()`. |
| `scalars` | No | `{ ScalarName : { serialize, parseValue, parseLiteral? } }`. See *Custom scalars* below. |

## Resolver convention

For a schema type `TypeName` with field `fieldName`, the module looks for `{resolverBasePackage}/{TypeName}Resolver.bx` and, if it exists and implements a `fieldName()` method, calls it. Otherwise it falls back to graphql-java's `PropertyDataFetcher` — reading a same-named key off the parent object. **You only write a resolver for fields that need custom logic** — a struct with matching keys just works.

When the parent is a class instance rather than a struct, the fallback reads the field through `get{FieldName}()` — generated by `accessors="true"` or written by hand — else a public member of that name, else null.

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

## Code-first schema

Instead of (or as well as) writing `.graphqls` files and resolver classes, annotate the classes you already have. The module generates SDL from them, merges it with any hand-written files, and wires each generated `Query` / `Mutation` field straight to its method.

```javascript
graphQLService = bxGraphQL( {
	"modelPaths"   : [ expandPath( "models/beans" ) ],
	"servicePaths" : [ expandPath( "models/services" ) ]
} )
```

### Beans become types and inputs

```javascript
/** A thing we sell. */
@graphqlType
@graphqlInput
class accessors="true" {

	@graphqlId
	property name="id" type="numeric";

	/** What it's called on the shelf. */
	property name="name" type="string" required="true";

	property name="maker" type="Maker";

	@graphqlField( "[Tag!]" )
	@graphqlInputField( "[String!]" )
	property name="tags" type="array";

	@graphqlReadOnly
	property name="createdAt" type="date";

	@graphqlIgnore
	property name="passwordHash" type="string";

}
```

generates

```graphql
scalar DateTime

"""A thing we sell."""
type Widget {
	id: ID
	"""What it's called on the shelf."""
	name: String!
	maker: Maker
	tags: [Tag!]
	createdAt: DateTime
}

"""A thing we sell."""
input WidgetInput {
	id: ID
	"""What it's called on the shelf."""
	name: String!
	maker: MakerInput
	tags: [String!]
}
```

Only annotated classes are exposed. Within one, every property is a field unless ignored, inherited properties included (parents' first).

| Annotation | On | Meaning |
|---|---|---|
| `@graphqlType` or `@graphqlType( "Name" )` | class | Emit `type Name`. Default name: the class's simple name. |
| `@graphqlInput` or `@graphqlInput( "Name" )` | class | Emit `input Name`. Default name: `{SimpleName}Input`. |
| `@graphqlId` | property | The field is an `ID`. |
| `@graphqlField( "SDL type" )` | property | The field's type, verbatim (so include any `!`). |
| `@graphqlInputField( "SDL type" )` | property | The same, for the input only. Without it the input uses `@graphqlField`'s. |
| `@graphqlReadOnly` | property | In the type, not the input. |
| `@graphqlIgnore` | property | In neither. |

An annotation's value needs parentheses: `@graphqlField( "[Tag!]" )`, not `@graphqlField "[Tag!]"`.

| BoxLang type | GraphQL type |
|---|---|
| `string` | `String` |
| `boolean` | `Boolean` |
| `integer`, `int` | `Int` |
| `numeric`, `float`, `double` | `Float` |
| `long` | `Long` |
| `date`, `datetime` | `DateTime` |
| an annotated class (`Maker`, `models.Maker`) | its type; its input when used in an input or as an argument |
| `Something[]` | `[Something!]` |
| `array`, `struct`, `any`, anything else | none: state it with `@graphqlField`, or construction fails naming the property |

`required="true"` adds `!`. `DateTime` and `Long` are declared for you when a generated field uses them.

### Service methods become queries and mutations

```javascript
class {

	/** One widget by id. */
	@graphqlQuery
	Widget function widget( required numeric id graphqlType="ID!", struct context ){ ... }

	@graphqlQuery
	@graphqlReturns( "[Widget!]!" )
	array function widgets( numeric limit = 20 ){ ... }

	@graphqlMutation
	Widget function saveWidget( required Widget input ){ ... }

	@graphqlMutation( "renameWidget" )
	Widget function rename( required numeric id graphqlType="ID!", required string name ){ ... }

}
```

generates

```graphql
type Query {
	"""One widget by id."""
	widget( id: ID! ): Widget
	widgets( limit: Float ): [Widget!]!
}

type Mutation {
	saveWidget( input: WidgetInput! ): Widget
	renameWidget( id: ID!, name: String! ): Widget
}
```

- The function must be public. The field takes the function's name, or the annotation's value.
- Each parameter becomes an argument, its type mapped as above and `!` added when `required`. `graphqlType="..."` on a parameter states the type verbatim instead.
- An argument the query leaves out isn't passed, so the function's own default applies.
- Parameters named `context` and `env` receive what a resolver's would, and aren't arguments in the schema.
- The return type is mapped from the function's; `@graphqlReturns( "..." )` states it verbatim, and is needed for `array`, `struct`, `any` or no declared type.
- A parameter typed as an `@graphqlInput` class receives an instance of that class: built with a bare `new`, then filled through its setters, nested beans included. Type the parameter `struct` (with `graphqlType="WidgetInput!"`) to receive the plain struct.
- One instance of each service class is built at construction time, by `instanceFactory` if given.

No operations are generated from a bean alone: a bean doesn't say how it's loaded or saved.

### Mixing with hand-written schema

Generated SDL is merged with whatever `schemaPaths` supplies. When a hand-written file declares `Query` or `Mutation`, the generated fields are added with `extend type`. A hand-written `extend type Widget { slug: String }` plus a `WidgetResolver.bx` with a `slug()` method adds a computed field to a generated type.

Declaring the same type by hand and by annotation, or the same field from two service functions, fails at construction time with `BxGraphQL.ConfigurationException`.

`getGeneratedSDL()` returns the generated SDL on its own; `printSchema()` returns the whole executable schema as SDL.

## Custom scalars

A `scalar` the schema declares needs its conversions, given in `scalars`:

```javascript
"scalars" : {
	"Upper" : {
		"serialize"  : ( value ) => ucase( value ),   // a resolver's value, on its way out
		"parseValue" : ( value ) => ucase( value )    // an argument, on its way in
	}
}
```

`parseLiteral` (optional) handles a value written in the query itself rather than passed as a variable; it defaults to `parseValue`, given the literal as a plain string, number or boolean. Whatever a conversion throws becomes an entry in `errors`, not a failed request. A declared scalar with no conversions fails at construction time.

Two need no configuration:

- `DateTime` — an ISO-8601 instant. Out: a BoxLang date, a `java.time` or `java.util` date, or an ISO string with an offset, always written in UTC (`2026-01-31T09:30:00Z`). In: the resolver receives the same normalized string.
- `Long` — a whole number up to 64 bits.

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

- Enums, and operations (CRUD) generated from a bean alone
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
- `tests/specs/ScalarsSpec.bx` — custom scalars: the built-in `DateTime` and `Long`, and one given in `scalars`, as output, literal and variable.
- `tests/specs/ClassInstanceSourceSpec.bx` — fields read from class instances rather than structs.
- `tests/specs/CodeFirstSpec.bx` — schema and operations generated from the annotated beans and services under `tests/resources/codefirst`, alone and merged with a hand-written file. `expected.graphqls` there is the SDL they should generate.
- `tests/specs/BifsSpec.bx` — the `bxGraphQL()` global BIF, built entirely through the friendly entry point rather than `new bxModules.bxgraphql.models.GraphQLService()` directly.

`setup-tests.bxs` creates a self-referencing `boxlang_modules/bxgraphql` symlink (gitignored, not committed) so BoxLang discovers this project as a real module and loads `libs/` onto the classloader `createObject("java", ...)` sees — module discovery only happens once at BoxLang process startup, so this has to run as its own invocation before `run-tests.bxs`, not get folded into it.

### A note on `expandPath()`

`expandPath()` resolves relative to the entry point of the running script — `run-tests.bxs` at the repo root when running the test suite — not relative to whichever file happens to call it. This is why `resolverBasePackage` in the examples above is documented as resolving relative to "the entry point," and why the test specs use repo-root-relative paths (`tests/resources/...`) rather than paths relative to each spec file's own directory.

## License

MIT
