# bx-graphql

A schema-agnostic GraphQL server for native BoxLang, wrapping [graphql-java](https://www.graphql-java.com/) (vendored, v26.0). The module ships with zero domain schema of its own — it parses `.graphqls` files you supply, wires resolvers by convention, and hands back a plain `{data, errors}` struct for whatever you execute. No ColdBox, no WireBox, no HTTP framework required — how you route requests to it is entirely up to you.

BoxLang's own module system auto-loads the jars under `libs/` onto the module's classloader, so `createObject("java", ...)` and `createDynamicProxy()` always resolve graphql-java classes through the same classloader — no separate classloader-registration step, and no per-engine branching, unlike the ColdBox edition of this module.

## Installation

```bash
box install bx-graphql
```

Requires BoxLang **1.15.0+** (developed and tested on 1.19.0; the code-first features read `getClassMetadata()`, which hasn't been tried on earlier versions). This module only runs on BoxLang — it isn't tested against Lucee or Adobe ColdFusion.

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
| `instanceFactory` | No | `( classPath, meta, kind ) => instance`, for classes that need dependencies. `kind` is `"service"` or `"resolver"` (asked once, the instance is kept) or `"input"` (asked for every input bean: return a new one each time). Default: a bare `new`. `classPath` is the class file's absolute path without its extension; `meta` is its `getClassMetadata()`. |
| `authorizer` | When anything is marked `@graphqlRequires` | `( requirement, context, info ) => boolean`. See *Authorization* below. |
| `loaders` | No | `{ name : ( keys, context ) => values }`. See *Batch loading* below. |
| `structuredErrors` | No | `true` to get `errors` as structs rather than message strings. See *Errors* below. |
| `onError` | No | `( detail ) => {}`, called once per error. See *Errors* below. |
| `maskErrors` | No | `true`, or `( detail ) => message`: what a client is told when a resolver throws. See *Errors* below. |
| `scalars` | No | `{ ScalarName : { serialize, parseValue, parseLiteral? } }`. See *Custom scalars* below. |
| `maxQueryDepth` | No | A whole number above zero. A query nested deeper is refused before any resolver runs. See *Limiting queries* below. |
| `maxQueryComplexity` | No | A whole number above zero. A query asking for more fields is refused before any resolver runs. |
| `introspection` | No | `false` to refuse `__schema` and `__type` queries. Default: `true`. |

## Resolver convention

For a schema type `TypeName` with field `fieldName`, the module looks for `{resolverBasePackage}/{TypeName}Resolver.bx` and, if it exists and implements a `fieldName()` method, calls it. Otherwise it falls back to graphql-java's `PropertyDataFetcher` — reading a same-named key off the parent object. **You only write a resolver for fields that need custom logic** — a struct with matching keys just works.

When the parent is a class instance rather than a struct, the fallback reads the field through `get{FieldName}()` — generated by `accessors="true"` or written by hand — else a public member of that name, else null.

Resolver methods receive four named arguments:

```javascript
any function fieldName( any source, struct args, any context, any env ){
	// source  — the parent object (whatever the parent resolver returned, or a plain
	//           struct/key). NULL for root Query/Mutation fields — don't mark it `required`.
	// args    — the field's GraphQL arguments, as an ordinary BoxLang struct/array
	//           (graphql-java's maps and lists made structs and arrays; every other
	//           value as it was coerced, so a custom scalar's parseValue result arrives as is).
	// context — whatever was passed as `context` to GraphQLService.execute() — typically
	//           the current request, however your app represents it.
	// env     — the raw graphql.schema.DataFetchingEnvironment, for advanced use.
}
```

Take the context from the `context` argument, not from `env.getContext()`: graphql-java has deprecated that, and the module no longer fills it. (It travels in `env.getGraphQlContext()` under the key `bxgraphql.context`.)

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
| `@graphqlWriteOnly` | property | In the input, not the type. |
| `@graphqlIgnore` | property | In neither. |
| `@graphqlDeprecated( "why" )` | property, function | Marks the type's field `@deprecated`. A doc comment's `@deprecated why` does the same. |
| `@graphqlField` or `@graphqlField( "SDL type" )` | function | The function is a field of the type (not the input). See *Functions as fields* below. |
| `@graphqlImplements( "Node, Named" )` | class | The type `implements` those interfaces. See *Interfaces and unions* below. |
| `@graphqlEnum( "A, B, C" )` | class | Emit `enum {SimpleName}` with those values, and nothing else from the class. See *Enums* below. |
| `@graphqlCrud( "ServiceName" )` | class | Generate get, list, save and delete fields, carried out by that service. See *CRUD from a bean* below. |
| `@graphqlCrudNames( "list=people" )` | class | Rename those fields. |
| `@graphqlRequires( "what" )` | class, property, function | Guarded: see *Authorization* below. |

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

### Functions as fields

A public function on a bean marked `@graphqlField` is a field of its type, called on the bean when asked for. Its parameters become the field's arguments exactly as a service function's do, `context`, `env` and `args` included.

```javascript
/**
 * The name, as a URL would have it.
 *
 * @separator What goes between the words.
 */
@graphqlField
string function slug( string separator = "-" ){
	return lcase( replace( variables.name, " ", arguments.separator, "all" ) );
}
```

```graphql
"""The name, as a URL would have it."""
slug(
	"""What goes between the words."""
	separator: String! = "-"
): String
```

`@graphqlField( "Maker" )` states the return type verbatim.

### Enums

```javascript
/** Where a widget is in its life. */
@graphqlEnum( "ACTIVE, PAUSED, RETIRED" )
class {
}
```

generates `enum Status { ACTIVE PAUSED RETIRED }`, named after the class. The values are strings in BoxLang, so a property or parameter that holds one is typed `string` and names the enum:

```javascript
@graphqlField( "Status" )
property name="status" type="string";

string function byStatus( string status = "ACTIVE" graphqlType="Status" ){ ... }
```

Typing it as the class (`type="Status"`) fails at construction time, since BoxLang would refuse the string.

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
	widgets( limit: Float! = 20 ): [Widget!]!
}

type Mutation {
	saveWidget( input: WidgetInput! ): Widget
	renameWidget( id: ID!, name: String! ): Widget
}
```

- The function must be public. The field takes the function's name, or the annotation's value.
- Each parameter becomes an argument, its type mapped as above and `!` added when `required`. `graphqlType="..."` on a parameter states the type verbatim instead.
- A default that is a plain string, number or boolean is written into the schema, and makes the argument non-null: `limit: Float! = 20` can be left out, but not given as null. An argument the query leaves out isn't passed, so the function's own default applies whether or not it could be written into the schema.
- A doc comment's `@parameterName ...` line becomes that argument's description. `@graphqlDeprecated( "why" )`, or a doc comment's `@deprecated why`, marks the field deprecated.
- Parameters named `context` and `env` receive what a resolver's would, and `args` the arguments as plain structs and arrays. None of the three is an argument in the schema.
- Annotated functions inherited from a parent class count. When a parent and its child are both scanned, the field goes to the child, so the child's instance is the one called; that holds when the child overrides the function with the annotation repeated, too. Two scanned classes that supply the same field with neither descending from the other fail at construction time.
- The return type is mapped from the function's; `@graphqlReturns( "..." )` states it verbatim, and is needed for `array`, `struct`, `any` or no declared type.
- A parameter typed as an `@graphqlInput` class receives an instance of that class, filled through its setters. So do nested beans, and lists of them: a property typed `Tag[]`, or typed `array` with `@graphqlInputField( "[TagInput!]" )`. A string for a property typed `date` becomes a date. Type the parameter `struct` (with `graphqlType="WidgetInput!"`) to receive the plain struct.
- An argument given as `null` reaches the function as null when the parameter has no default. When it has one, the null is refused rather than quietly replaced, because BoxLang gives a parameter its default whenever it receives null: a plain default is refused by the schema (the argument is non-null), and a default that is an expression (`since = now()`) by an error on that field (`BxGraphQL.NullArgument`).
- A key given as `null` sets that property to null; a key left out leaves it alone. Both read as null on the bean afterwards, so to tell them apart take `args` as well and look at its keys: `arrayContainsNoCase( structKeyArray( args.input ), "price" )` (`structKeyExists()` is false for a null).
- One instance of each service class is built at construction time, by `instanceFactory` if given.

No operations are generated from a bean alone: a bean doesn't say how it's loaded or saved. `@graphqlCrud` names the service that does.

### CRUD from a bean

```javascript
@graphqlType
@graphqlInput
@graphqlCrud( "WidgetService" )
class accessors="true" {

	@graphqlId
	property name="id" type="numeric";

	property name="name" type="string" required="true";

}
```

`WidgetService` is a class under `servicePaths`, found by its simple name. Each of these functions it has becomes a field, with no annotation needed on the function:

| Service function | Generated field |
|---|---|
| `get( id )` | `widget( id: ID! ): Widget` on `Query` |
| `list()` | `widgets: [Widget!]!` on `Query` |
| `save( widget )` | `saveWidget( input: WidgetInput! ): Widget` on `Mutation` |
| `delete( id )` | `deleteWidget( id: ID! ): Boolean` on `Mutation` |

- A function the service doesn't have makes no field. Neither does `save` when the bean isn't an `@graphqlInput`.
- The bean needs exactly one `@graphqlId` property; the `id` argument takes that property's name.
- The functions' parameters can be called anything: the first one that isn't `context`, `env` or `args` receives the id, or the bean filled from the input. `save` creates or updates, as the service sees fit; `delete` answers with the boolean it returns.
- The list's name is the type's with an `s`. `@graphqlCrudNames( "list=people" )` on the bean renames any of `get`, `list`, `save` and `delete`.
- `@graphqlRequires` on the service or its functions guards these fields as it does any other.
- A generated field that a service function already declares fails at construction time.

There is no filtering, sorting or paging: write those as `@graphqlQuery` functions.

### Mixing with hand-written schema

Generated SDL is merged with whatever `schemaPaths` supplies. When a hand-written file declares `Query` or `Mutation`, the generated fields are added with `extend type`. A hand-written `extend type Widget { slug: String }` plus a `WidgetResolver.bx` with a `slug()` method adds a computed field to a generated type.

Declaring the same type by hand and by annotation, or the same field from two service functions, fails at construction time with `BxGraphQL.ConfigurationException`.

`getGeneratedSDL()` returns the generated SDL on its own; `printSchema()` returns the whole executable schema as SDL.

## Authorization

`@graphqlRequires( "what" )` guards a field. It goes on a service function, on a bean's property or `@graphqlField` function, on a method of a hand-written `{Type}Resolver.bx`, or on any of those classes to cover everything in it. Before a guarded field is read or its function called, `authorizer` is asked:

```javascript
"authorizer" : ( requirement, context, info ) => arrayContains( context.user.roles, requirement )
```

`requirement` is the annotation's value (`""` for a bare `@graphqlRequires`), `context` is what was passed to `execute()`, and `info` is `{ type, field, args }`. Anything but `true` refuses the field: it resolves to null, with `Not authorized: Query.secrets` in `errors` (type `BxGraphQL.NotAuthorized`), and the rest of the query carries on. A property's or function's own annotation replaces its class's, and a class without one takes its nearest parent's, so a guard on a base class covers what its children expose.

A guard on a bean's class covers its properties in the type (an input isn't guarded). One on a resolver class covers that resolver's methods, not the fields read straight off the parent. A field written only in a `.graphqls` file, with no resolver method, has nothing to carry the annotation: give it a resolver method, or check `context` where its parent is loaded.

Marking anything `@graphqlRequires` without setting `authorizer` fails at construction time.

## Batch loading

A list of 50 widgets that each look up their maker is 50 lookups. A loader turns them into one:

```javascript
"loaders" : {
	"makers" : ( keys, context ) => makerService.byIds( keys )
}
```

```javascript
@graphqlField( "Maker" )
function maker( any env ){
	return arguments.env.getDataLoader( "makers" ).load( variables.makerId );
}
```

`load()` returns a promise, which the function returns as it is. graphql-java gathers the keys asked for across that level of the query, drops repeats, and calls the loader once. The loader returns a struct of values by key (a key it has nothing for resolves to null), or an array with one value per key in the keys' order. Any resolver can use a loader the same way, through its `env` argument. Loaded values last for one `execute()`.

## Interfaces and unions

Write the interface or union in a `.graphqls` file. A generated type joins an interface with `@graphqlImplements( "Named" )`, and a union by being listed in it (`union Found = Book | Film`).

For a field typed as an interface or union, the concrete type of each value is worked out in this order:

1. `{resolverBasePackage}/{InterfaceOrUnion}Resolver.bx`, if it has a `resolveType( value, context, env )` method: it returns the type's name.
2. A struct's `__typename` key.
3. A class instance's class: the type generated from it, else its simple name.

A value none of these can place is an error on that field.

## Errors

`errors` is an array of graphql-java's message strings, as it always was. With `"structuredErrors" : true` each is a struct as the GraphQL spec has it:

```javascript
{
	"message"    : "None left",                  // the exception's own message
	"path"       : [ "saveWidget" ],
	"locations"  : [ { "line" : 1, "column" : 12 } ],
	"extensions" : { "type" : "Shop.OutOfStock", "classification" : "DataFetchingException" }
}
```

`type` is the BoxLang exception type a resolver or service function threw; `classification` is graphql-java's (`ValidationError`, `InvalidSyntax`, `DataFetchingException`...). `path`, `locations` and `extensions` are left out when empty.

Either way, `onError` is called once per error with `{ message, rawMessage, path, locations, type, classification, exception }` (`exception` being the one thrown, if any): the place to log what `errors` only summarizes.

### Masking

An exception's message can say more than a client should read: a table name, a file path. With `"maskErrors" : true`, whatever a resolver or service function throws reaches the client as `Internal server error`, with no `type`; `onError` still gets all of it. To let some messages through, give a function instead, whose answer is the message:

```javascript
"maskErrors" : ( detail ) => left( detail.type, 5 ) == "Shop." ? detail.message : "Something went wrong"
```

Errors in the query itself and the module's own refusals (`BxGraphQL.NotAuthorized`, `BxGraphQL.NullArgument`) are never masked.

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

What `parseValue` returns is what the resolver or service function receives — a date, a `java.time` value, a `BigDecimal` — not a copy turned to text on the way.

`parseLiteral` (optional) handles a value written in the query itself rather than passed as a variable; it defaults to `parseValue`, given the literal as a plain string, number or boolean. Whatever a conversion throws becomes an entry in `errors`, not a failed request. A declared scalar with no conversions fails at construction time.

Two need no configuration:

- `DateTime` — an ISO-8601 instant. Out: a BoxLang date, a `java.time` or `java.util` date, or an ISO string with an offset, always written in UTC (`2026-01-31T09:30:00Z`). In: the resolver receives the same normalized string.
- `Long` — a whole number up to 64 bits.

## Executing queries

```javascript
result = graphQLService.execute(
	query          = "...",    // required
	queryVariables = {},       // optional — GraphQL $variables
	context        = "",       // optional — forwarded as-is to every resolver's `context` arg
	operationName  = ""        // optional — which operation to run, when the query holds several
)
```

Returns a plain BoxLang struct — `{ data : {...}, errors : [...] }` — so callers never touch graphql-java classes directly. `errors` is always present (an empty array on success): message strings by default, structs with `structuredErrors`. See *Errors* above.

### From a request body

`handle()` does the same for a GraphQL-over-HTTP request body, given as a struct or as the JSON text it arrived as:

```javascript
result = graphQLService.handle( requestBodyText, context )
// body: { "query" : "...", "variables" : { ... }, "operationName" : "..." }
```

Only `query` is required. A body that can't be run (not JSON, no query, `variables` that isn't an object) comes back as `{ data : null, errors : [ ... ] }` rather than an exception. Routing, status codes and the response's content type stay with your app.

## Limiting queries

For an endpoint anyone can reach, bound what one query may ask for:

```javascript
"maxQueryDepth"      : 8,
"maxQueryComplexity" : 200,
"introspection"      : false
```

Depth counts levels of nesting: `{ widget { name } }` is 2. Complexity counts fields: `{ widget { id name } }` is 3. A query over either limit comes back with one entry in `errors` and no data, and no resolver is called. `"introspection" : false` refuses `__schema` and `__type`; `__typename` still works.

## Accessing the schema programmatically

`getSchema()` returns the raw `graphql.schema.GraphQLSchema` graphql-java built from your `.graphqls` files; `getEngine()` returns the underlying `graphql.GraphQL` engine itself. Both are live graphql-java objects — useful for anything graphql-java itself supports, most commonly printing the schema back out as SDL via `graphql.schema.idl.SchemaPrinter` for docs or codegen tooling.

## Not yet implemented

- Filtering, sorting and paging for the lists `@graphqlCrud` generates
- Interfaces, unions and input unions generated from classes: they are written in SDL (see *Interfaces and unions*)
- Subscriptions

## Tests

[TestBox](https://testbox.ortusbooks.com/) specs, run headlessly (no server needed):

```bash
boxlang setup-tests.bxs   # once per checkout
boxlang run-tests.bxs     # every time after that
```

`run-tests.bxs` exits non-zero on any failure/error, so it's CI-friendly — just run `setup-tests.bxs` once beforehand (e.g. in a CI image build step); `.github/workflows/tests.yml` does exactly that on every push and pull request. Structure:

- `tests/specs/GraphQLServiceSpec.bx` — the resolver convention end-to-end against a real engine: `PropertyDataFetcher` fallback, explicit-resolver precedence, missing-resolver-class fallback, query variables actually reaching resolver arguments, and spec-shaped errors for an invalid query.
- `tests/specs/GraphQLServiceConfigSpec.bx` — startup config validation: missing/invalid `schemaPaths` or `resolverBasePackage`, including a wildcard `schemaPaths` entry whose directory doesn't exist.
- `tests/specs/DataFetcherAdapterSpec.bx` — resolver dispatch in isolation, via a fake `DataFetchingEnvironment` (`tests/helpers/FakeDataFetchingEnvironment.bx`). Doesn't cover the `PropertyDataFetcher`-fallback path — graphql-java's real `PropertyDataFetcher.get()` calls more of that interface than a fake can cheaply satisfy — that path is covered end-to-end instead, in `GraphQLServiceSpec`.
- `tests/specs/JavaClassFactorySpec.bx` — Java class-handle resolution, using plain JDK classes so it runs without the module-loading setup below.
- `tests/specs/ScalarsSpec.bx` — custom scalars: the built-in `DateTime` and `Long`, and one given in `scalars`, as output, literal and variable.
- `tests/specs/ClassInstanceSourceSpec.bx` — fields read from class instances rather than structs.
- `tests/specs/CodeFirstSpec.bx` — schema and operations generated from the annotated beans and services under `tests/resources/codefirst`, alone and merged with a hand-written file. `expected.graphqls` there is the SDL they should generate.
- `tests/specs/CodeFirstFeaturesSpec.bx` — the same fixtures: enums, bean functions as fields, inherited functions, defaults and deprecation, lists and nulls in inputs, authorization, structured errors, batch loaders, `instanceFactory`'s kinds.
- `tests/specs/AbstractTypesSpec.bx` — interfaces and unions, over `tests/resources/abstract`.
- `tests/specs/CrudSpec.bx` — `@graphqlCrud`, over `tests/resources/crud`.
- `tests/specs/ErrorMaskingSpec.bx` — `maskErrors`, as `true` and as a function.
- `tests/specs/HandleSpec.bx` — `handle()`: request bodies as structs and as JSON, and the ones it refuses.
- `tests/specs/GuardsSpec.bx` — `@graphqlRequires` on properties and hand-written resolvers, over `tests/resources/guards`, and the context reaching resolvers and loaders.
- `tests/specs/RequestOptionsSpec.bx` — `operationName`, and the `maxQueryDepth`, `maxQueryComplexity` and `introspection` settings.
- `tests/specs/BifsSpec.bx` — the `bxGraphQL()` global BIF, built entirely through the friendly entry point rather than `new bxModules.bxgraphql.models.GraphQLService()` directly.

`setup-tests.bxs` creates a self-referencing `boxlang_modules/bxgraphql` symlink (gitignored, not committed) so BoxLang discovers this project as a real module and loads `libs/` onto the classloader `createObject("java", ...)` sees — module discovery only happens once at BoxLang process startup, so this has to run as its own invocation before `run-tests.bxs`, not get folded into it.

### A note on `expandPath()`

`expandPath()` resolves relative to the entry point of the running script — `run-tests.bxs` at the repo root when running the test suite — not relative to whichever file happens to call it. This is why `resolverBasePackage` in the examples above is documented as resolving relative to "the entry point," and why the test specs use repo-root-relative paths (`tests/resources/...`) rather than paths relative to each spec file's own directory.

## License

MIT
